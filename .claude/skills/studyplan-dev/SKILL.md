---
name: studyplan-dev
description: >-
  Dr. Bugeon의 학습계획표(StudyPlan)처럼 단일 HTML 파일 + localStorage + Supabase 동기화로
  만든 개인용 일정/학습계획 앱을 개발·수정·디버그·검증·배포할 때 사용한다. 다룰 수 있는 작업:
  단일 HTML 앱 수정 방법론, localStorage 저장 계층, Supabase 저장·동기화(일반 병합 동기화 vs
  클라우드 최종본 덮어쓰기 구분, canonical_version, tombstone 소프트삭제), 학기중/휴일/방학 모드,
  일정 블록·목표·완료체크 도메인, overnight(자정 넘김) 블록, 24시간 시계 캔버스, 백업/복원,
  Supabase SQL 점검, GitHub Pages 배포. "동기화가 안 맞아요", "삭제한 게 되살아나요",
  "최종본 저장", "모드/도메인 추가", "체크가 진행률에 반영 안 돼요", "배포/캐시 문제" 같은
  요청에서 호출한다. (이미지/Cloudinary/IndexedDB/의학도메인은 이 앱 범위가 아니다.)
---

# StudyPlan — 통합 개발 스킬

단일 HTML 파일로 동작하고 **로컬(localStorage) 우선 + Supabase 기준본(canonical) 동기화**로
구성된 개인용 학습계획표 앱(`Dr. Bugeon의 학습계획표`)을 개발·수정·검증할 때 쓰는 스킬이다.

> 이 앱은 Medical Note 계열과 같은 동기화 철학을 공유하지만, **Cloudinary 이미지·IndexedDB·
> 의학 도메인·검사수치 파서는 없다.** 저장은 localStorage 하나, 데이터 도메인은 일정/목표/체크뿐이다.

## 핵심 원칙 (먼저 기억할 것)

1. **클라우드(Supabase)가 기준 원본**, localStorage는 빠른 화면 캐시다. 온라인에서 클라우드 성공이 확인될 때만 저장을 확정한다.
2. **일반 동기화 ≠ 클라우드 최종본 덮어쓰기.** 두 모드를 절대 섞지 않는다. 일반 병합 동기화는 `canonical_version`을 먼저 읽고 진행한다.
3. **삭제는 소프트삭제 + tombstone.** hard delete 금지 → 다른 기기에서 부활 방지.
4. **새 도메인/모드는 카드 렌더링만으로 끝이 아니다.** 저장/로드·상세·편집·삭제·tombstone·일반동기화·최종본저장·백업/복원·저장본 버전 확인·SQL까지 같은 수준으로 연결한다. (현재 도메인: 일정 블록 `schedule`, 목표 `goal`, 완료체크 `check` + tombstone)
5. **상태를 바꾸면 의존 UI를 모두 다시 그린다.** 예: 체크 토글은 표뿐 아니라 상단 진행률·7일 성취·시계 세그먼트까지 갱신(`updateUI()`)해야 한다. "보이는 상태"와 "저장되는 상태"가 다르면 회귀로 본다.
6. **개인용 앱:** 치명적 위험(키 하드코딩, RLS 비활성 등)만 보안 지적, 그 외 과도한 보안 리팩토링은 먼저 제안하지 않는다.
7. **항상 결과 보고 + 환경 인식.** 모든 작업은 마지막에 **결과 보고**로 마무리한다(아래 "완료 보고"). 그리고 실행 환경(데스크톱 vs 모바일/웹)에 맞춰 동작한다(아래 "실행 환경").

## 실행 환경 (데스크톱 vs 모바일/웹) — 동일 목표, 환경별 동작

이 스킬은 데스크톱(로컬 파일시스템·node·git)과 모바일/웹(claude.ai — 로컬 도구 없음, 임시 컨테이너) 양쪽에서 호출될 수 있다. **환경을 먼저 인식하고 같은 목표를 환경에 맞게 달성한다.**

- **환경 무관(어디서나 동일):** 설계 원칙·데이터 모델·동기화 규칙·UX 판단·콘텐츠/JSON 생성. 모바일에서도 100% 같게 적용.
- **데스크톱 전용(로컬 도구 필요):** 파일 직접 수정, `node --check` 구문검사, git 클론·배포. 머신별 절대경로는 스킬에 하드코딩하지 말고 사용자 메모리에서 참조.
- **모바일/웹 대체 동작:** ① 수정본 HTML/코드 조각을 채팅/아티팩트로 출력(붙여넣기용), ② 구문검사는 수동 검토(괄호·따옴표·템플릿 짝)로 대신하고 "node 미실행" 명시, ③ 배포는 데스크톱에서 하도록 안내. 단, **claude.ai code(원격 컨테이너)** 환경이면 git이 있으므로 직접 commit·push가 가능하다 — 이때는 데스크톱처럼 처리하되 "컨테이너는 임시이므로 push해야 보존됨"을 잊지 않는다.
- **첫 행동:** 로컬 파일/도구 접근 가능 여부로 환경을 판단하고, 불가하면 위 대체 동작으로 자동 전환한다("데스크톱에서만 됩니다"로 거절하지 말고, 가능한 부분을 해주고 나머지는 데스크톱 단계로 넘긴다).

## 작업 전 필수 절차

- 수정 전 실제 코드를 먼저 읽고, 문제가 UI / 저장 / 동기화 / 배포 캐시 중 무엇인지 구분한다.
- 단일 HTML 구조는 유지하고, `CSS → HTML → 상태 변수 → 신규 함수 → 기존 함수 호출 1줄` 순서로 점진 수정한다.
- 같은 지점에서 3회 이상 패치 실패 시 해당 섹션/함수를 통째로 깨끗한 UTF-8로 재작성하고 JS 구문 검사를 한다.
- Supabase 관련 작업은 끝에 **SQL 변경 필요: 있음/없음/불확실**을 반드시 보고한다.

## 검증 원칙

StudyPlan은 단일 HTML이 ~100KB로 가벼워(Medical Note의 1MB와 달리) 브라우저 검증 부하가 낮다. 그래도 순서는 지킨다.

1. **JS 구문 검사 먼저.** `<script>` 본문을 추출해 `new Function(src)` 또는 `node --check`로 파싱.
2. **텍스트·수치 검증.** `document.scrollingElement.scrollWidth <= innerWidth`(가로 스크롤 없음), 주요 패널 `getBoundingClientRect()`가 viewport 안인지, 모달 닫기/저장 버튼 가시성, 항목 수/상태 확인. 회귀 판정 대부분은 여기서 끝.
3. **스크린샷은 증거용으로 마지막에, 가볍게.** 뷰포트 한 장 또는 특정 요소만. 헤드리스 Chromium은 `executablePath:'/opt/pw-browsers/chromium'`(원격 환경) 사용.
4. **페이지 로드 대기는 `domcontentloaded` + 구체 DOM 신호.** `networkidle` 금지 — Supabase REST·Google Fonts·cdn(supabase-js) 때문에 idle이 안 된다.
5. 검증을 환경 문제로 못 돌렸으면 그 사실을 완료 보고에 남긴다.

## 출력 파일명 규칙

업데이트 HTML 제공 시: `YYMMDD_HH.MM 앱이름_업데이트내역.html`. 앱 내부 개발자 정보(`APP_VERSION`, `APP_UPDATED_AT`, `APP_FILE_FALLBACK`, `UPDATE_HISTORY`)에도 같은 버전·변경 내역을 남긴다.

## 완료 보고 (항상 마지막에)

최소 포함: ① 무엇을 바꿨나(핵심만), ② 검증 결과(*실제로 한 것*과 결과 — JS 파싱/렌더 등), ③ 산출물(파일 경로), ④ `SQL 변경 필요: 있음/없음/불확실`, ⑤ `지침 업데이트: <문서명>/없음`, ⑥ 배포했으면 배포 결과·Pages URL(또는 사용자가 직접 push해야 하는지). 못한 것·실패한 검증은 숨기지 않는다.

## 참고 문서 (필요할 때만 읽기 — progressive disclosure)

- **[references/storage-sync.md](references/storage-sync.md)** — localStorage 키 목록, `study_plan_records` 스키마, 동기화 모드 구분(일반 병합 vs 최종본), canonical_version, tombstone, 백업/복원 분리, Supabase SQL. **저장·동기화·삭제·백업 작업 시 필수.**
- **[references/ui-rules.md](references/ui-rules.md)** — UI/상태 회귀 방지: 체크↔진행률 동기화, 모달 닫힘 정책, 로컬 날짜 helper, overnight 블록, 시계 캔버스, 모드 전환/이름 변경 전파, 반응형, 개발자 정보 규칙.
- **[references/deploy.md](references/deploy.md)** — GitHub Pages 배포(루트 `index.html`), Supabase SQL 적용, 데스크탑 git push 자동화, 모바일→데스크탑 배포 인계. (Cloudinary/Edge Function 없음.)

## 지침 자동 구분 규칙

작업 중 새 규칙이 생기면: **모든 앱에 적용할 짧은 원칙**은 마켓플레이스의 공용 규칙 문서로, **StudyPlan 전용 절차**는 위 참고문서(`storage-sync`/`ui-rules`/`deploy`)에 둔다. 완료 보고에 `지침 업데이트: <문서명> / 없음`을 명시한다.
