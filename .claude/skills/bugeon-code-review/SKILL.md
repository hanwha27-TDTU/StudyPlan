---
name: bugeon-code-review
description: >-
  코드 변경분(git diff / PR / 방금 수정한 파일)을 커밋·머지 전에 리뷰할 때 사용한다.
  정확성 버그와, 김부건의 단일 HTML + localStorage/Supabase 동기화 앱에서 반복되는 함정을
  우선순위로 점검하고 심각도·확신도를 붙여 보고한다. 점검 항목: 상태 변경 후 렌더 누락,
  batch upsert res.ok 미확인, tombstone/updated_at 누락, 로컬 날짜 helper, escapeHTML,
  일반 동기화 vs 최종본 동기화 혼용, 백업-후-동기화 순서, innerHTML 재렌더 시 포커스 유실,
  카드 내부 버튼 stopPropagation. "이 변경 리뷰해줘", "커밋 전에 봐줘", "PR 점검해줘" 같은
  요청에서 호출한다. (앱 전체 전수점검은 static-html-review, Anthropic 기본 diff 리뷰는
  내장 /code-review 를 쓴다 — 이 스킬은 변경분 + 반복 함정 체크리스트에 특화.)
---

# bugeon-code-review — 변경분 리뷰 스킬

**앱 전체가 아니라 "이번에 바뀐 것"** 을 커밋·머지 전에 리뷰한다. 목표는 회귀와 정확성 버그를
잡는 것. 막연한 칭찬·중복 나열을 피하고 `파일:줄번호`로 근거를 대며, 심각도와 확신도를 붙인다.

> 구분: 앱 **전체** 감사는 `static-html-review`, 범용 diff 리뷰는 내장 `/code-review`.
> 이 스킬은 **변경분 + 아래 반복 함정 체크리스트**에 특화된 경량 리뷰다.

## 진행 순서

1. **무엇이 바뀌었는지 먼저 본다.** `git diff`(또는 최근 수정 파일)를 읽고 변경 범위가
   UI / 저장 / 동기화 / 배포 중 무엇에 닿는지 구분한다. 바뀌지 않은 코드는 리뷰 대상이 아니다.
2. **JS 구문 검사.** `<script>` 본문을 `new Function(src)` 또는 `node --check`로 파싱.
3. 아래 체크리스트로 변경분을 훑고 **정확성 → UX 일관성 → 성능/유지보수** 순으로 정리.
4. 각 항목에 심각도(🔴/🟡/🟢), `줄번호`, 한 줄 수정 방향, **확신도(확실/추정)** 를 붙인다.
5. 확실한 버그와 추정을 섞지 않는다. 추정은 그렇게 표시한다.

## 반복 함정 체크리스트 (Bugeon 앱 공통)

### 🔴 정확성 / 상태
- **상태 변경 후 의존 UI 렌더 누락.** 데이터를 바꾸는 핸들러가 관련 화면을 모두 다시 그리는가?
  (예: 체크 토글이 표만 갱신하고 진행률·통계·시계는 안 그림 → `updateUI()` 누락)
- **batch upsert는 `res.ok` 확인.** `fetch` 성공만 보고 넘어가지 않는가?
- **삭제는 tombstone + `updated_at`.** hard delete로 다른 기기에서 부활하지 않는가?
  외부/백업에서 온 tombstone은 `updated_at` 없으면 `deleted_at`으로 보정하는가?
- **일반 동기화 vs 최종본(canonical) 동기화 혼용.** 일반 병합은 `canonical_version`을 먼저 읽는가?
  최종본 지정은 원격 전용 row까지 정리하는가?
- **백업 가져오기와 원격 반영 분리.** 가져오기 중 자동 flush로 오래된 백업이 최신본을 덮지 않는가?
  (`suppressAutoSync` 가드)

### 🟡 UX 일관성
- **로컬 날짜 helper.** 날짜 기본값을 `toISOString().slice(0,10)`로 만들지 않는가?
  (UTC라 자정 전후 전날로 밀림 → `getFullYear/Month/Date` 기반 helper)
- **`innerHTML` 재렌더 시 포커스/커서 유실.** 입력 중 영역을 통째로 다시 그리지 않는가?
- **카드 내부 버튼 `event.stopPropagation()`.** 카드 클릭과 버튼 클릭이 섞이지 않는가?
- 모달이 명시적(닫기/취소/ESC)으로만 닫히고, 라벨/헤더가 실제 저장 단위와 일치하는가?
- 새 도메인/모드 추가 시 저장·삭제·tombstone·동기화·백업/복원·저장본 버전·항목 수 카운트까지 함께 갱신했는가?

### 🟢 성능 / 유지보수
- 렌더 루프 안에서 `localStorage`/`JSON.parse`를 반복하지 않는가? (렌더당 1회 로드)
- 사용자 입력을 DOM에 넣는 경로에 `escapeHTML`이 빠지지 않았는가?
- `index.html` 변경이면 버전 상수(`APP_VERSION`·`APP_VERSION_SHORT`·`APP_UPDATED_AT`·
  `APP_FILE_FALLBACK`·`UPDATE_HISTORY`)를 함께 올렸는가?

## 검증
- JS 파싱은 필수. 레이아웃 변경이면 헤드리스 Chromium(`/opt/pw-browsers/chromium`)으로
  가로 스크롤 없음(`scrollWidth<=innerWidth`)·핵심 버튼 가시성 확인. `networkidle` 금지.
- 검증을 환경 문제로 못 돌렸으면 그 사실을 보고에 남긴다.

## 출력 형식
- 총평(좋은 점 1~2개) → 🔴/🟡/🟢 순 findings(파일:줄·심각도·확신도·수정 방향) → 우선순위 한 줄.
- Supabase에 닿았으면 `SQL 변경 필요: 있음/없음/불확실`을 명시.
- 수정까지 요청받으면 동작 동일성 유지, 한 번에 한 의미 단위로 편집, 편집 후 재검증.
