# 저장 계층 & Supabase 동기화 (StudyPlan)

> `studyplan-dev` 스킬 참고 문서. 진입점은 상위 폴더의 SKILL.md다.
> StudyPlan은 **localStorage만** 쓴다(IndexedDB·Cloudinary 없음). 데이터 도메인은 일정/목표/체크뿐이다.

## 1. localStorage 키 목록 (실제 코드 기준)

```text
study_plan_v5                      일정(schedule) — semester 모드. 다른 모드는 study_plan_v5_<mode>
study_goals_v1                     목표(goals) — semester. 다른 모드는 study_goals_v1_<mode>
study_plan_active_mode_v1          현재 모드 (semester | holiday | vacation)
study_plan_checks_v1               완료체크 { 'YYYY-MM-DD': { blockKey: checkRecord } }
study_plan_tombstones_v1           삭제 묘비(tombstone) 배열
study_plan_last_canonical_v1       기기별 마지막으로 반영한 canonical_version
study_plan_device_instance_v1      기기 고유 id
study_plan_sb_cfg_v1               Supabase 설정 { url, key, device }
```

- **모드별 분리:** 일정·목표는 모드(semester/holiday/vacation)마다 별도 키. `scheduleKey(mode)`, `goalsKey(mode)` 헬퍼를 거친다.
- **체크는 모드 무관 날짜축:** `checks[dateStr][blockKey]` 구조. 단 각 check 레코드 내부에 `mode` 필드가 있어 동기화 시 모드별로 분리된다.
- 로컬 캐시는 빠른 임시 표시용이며, Supabase 로드/동기화 성공 후의 화면을 기준본으로 본다.

## 2. 레코드 정규화 규칙 (필수)

모든 레코드(schedule/goal/check/tombstone)는 `id`, `created_at`, `updated_at`을 가진다.

- 저장·수정·체크 토글·삭제 등 **모든 변경 시 `updated_at`을 갱신**한다.
- 같은 `id`가 로컬과 클라우드에 모두 있으면 `updated_at`이 최신인 쪽을 선택한다(`newerRow`).
- 외부 JSON/구버전 백업에서 tombstone을 가져올 때 `updated_at`이 없고 `deleted_at`만 있으면 **삭제 시각을 `updated_at`으로 보정**한다. 가져오기 시각으로 임의 승격하면 정상 레코드를 잘못 숨긴다(`normalizeTombstone`).
- 통계·진행률·항목 수는 tombstone/soft-delete 항목을 **제외한 활성 항목** 기준으로 계산한다.

## 3. Supabase 테이블

```text
study_plan_records    한 줄 = 한 레코드. 컬럼: id, sync_id, type, mode, payload(jsonb),
                      created_at, updated_at, deleted_at.  PK = (sync_id, id)
                      type ∈ { schedule, goal, check }.  sync_id = 사용자 동기화 ID(기기 공통)
language_sync_meta    key/value 메타. canonical_version 저장용.
                      key = `study_plan:<device>:canonical_version`
```

- `sync_id`는 사용자가 입력하는 "동기화 ID"(여러 기기가 같은 값을 쓰면 데이터 공유). `study_plan_records`의 PK가 `(sync_id, id)`이므로 ID가 다른 기기 그룹끼리는 격리된다.
- payload에는 도메인별 필드만 담는다: schedule=`{start,end,icon,task,strategy,color,label,category}`, goal=`{text}`, check=`{date,block_id,checked}`.

### 통합 SQL (앱 내부 `SQL_TEXT` 상수와 동일하게 유지)

코드의 `study_plan_records`/`language_sync_meta` 정의가 바뀌면 앱 내부 SQL 상수, 개발자 정보/클라우드 모달의 SQL, 최종 답변 SQL을 **같은 버전으로 맞춘다.** PostgreSQL dollar quote(`do $$ ... end $$;`)가 JS template literal에서 `${...}` 보간으로 오염되지 않았는지 추출본을 확인한다. SQL 제공 시 전체를 ```sql 코드블록으로 답변에 함께 제공한다.

## 4. 동기화 모드 — 절대 섞지 않기

### 일반 병합 동기화 (`cloudAutoSync`, `queueAutoSync`)
- 로컬과 클라우드를 `updated_at` 기준으로 병합한 뒤 클라우드에 upsert해 로컬 전용 항목을 다른 기기로 전파.
- **진입 시 먼저 `canonical_version`을 읽는다.** 원격 canonical이 로컬에 반영된 값과 다르면 → 병합하지 말고 **클라우드 기준을 로컬에 반영만** 하고 종료.
- 자동 동기화는 `suppressAutoSync` 플래그로 가져오기/복원 중 차단한다.
- flush가 이미 실행 중이면 즉시 성공 반환하지 말고 기존 `syncPromise`를 기다린다.

### 최종본 기준 동기화 (`cloudPush` = 이 기기를 최종본으로, `cloudPull` = 클라우드 최종본 받기)
- "이 기기 → 클라우드"는 해당 기기를 최종본으로 지정하는 동작. `language_sync_meta`의 `canonical_version`을 갱신한다.
- 최종본 지정은 대표 도메인만 처리하면 안 된다. **현재 기기에 없는 원격 전용 row를 delete/tombstone 처리**해 기준본을 완성한다(`cloudPush`의 cloudOnlyDeletes).
- 다른 기기는 `canonical_version` 변경을 감지하면 로컬 전용 항목을 보존하지 않고 클라우드 기준으로 맞춘다.
- `저장본 버전 확인`(`cloudVersionCheck`)은 read-only로, 최종 저장 위치·canonical_version·항목 수·기기 라벨·manifest/hash 일치 여부를 표시한다.

### 공통
- batch upsert는 `fetch` 성공만 보지 말고 **반드시 `res.ok`를 확인**(`restUpsert`)한다.
- 연결 테스트 성공 시 표시로 끝내지 말고 즉시 동기화를 실행한다.

## 5. 백업/복원 — 동기화와 분리

- 전체 JSON 백업(`makeBackupPayload`)은 **모든 모드의 일정·목표 + 전체 체크 + tombstone**을 포함한다. 새 도메인을 추가하면 백업 객체와 `version`/스키마 값도 함께 올린다.
- 백업 복원은 **먼저 로컬 반영·검증을 끝낸 뒤**, 사용자가 일반 동기화 또는 최종본 지정을 선택하게 한다. 가져오기 도중 자동 flush/upsert가 섞이면 오래된 백업이 원격 최신본을 덮을 수 있다 → `suppressAutoSync=true`로 감싼다.
- 복원 시 기존 데이터는 tombstone을 남기고 교체한다(`restoreModeData`).
- 다운로드 파일명은 `백업날짜_앱이름_전체백업` 형식, 금지문자(`/ \ : * ? " < > |`)는 `safeFileName`으로 제거.
- 구버전(단일 모드) 백업과 신버전(`backupScope:'all_modes'`) 백업을 구분해 복원한다.

## 6. 동기화 테스트 시나리오 (회귀 점검)

1. 같은 sync_id 두 기기에서 서로 다른 일정 추가 → 일반 동기화 → 양쪽에 모두 보이는지.
2. 한 기기에서 블록 삭제 → 동기화 → 다른 기기에서 **되살아나지 않는지**(tombstone 동작).
3. 기기 A에서 "최종본으로 올리기" → 기기 B에서 일반 동기화 → B가 A 기준으로 맞춰지고 B 로컬 전용 항목이 정리되는지.
4. 오래된 JSON 백업 복원 → 자동 업로드가 막히고, 사용자가 모드 선택해야 원격 반영되는지.
5. 완료 보고에 `SQL 변경 필요: 있음/없음/불확실` 명시.
