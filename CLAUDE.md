# StudyPlan — 작업 규칙 (Claude Code)

## Git 워크플로 (상시 적용)

- **코드/문서를 수정하면 항상 자동으로 `커밋 → 푸시 → PR 생성 → 머지`까지 진행한다.**
  머지 여부를 매번 되묻지 않는다. (사용자 상시 승인: "항상 자동 머지 적용")
- 작업은 `main`에 직접 하지 않고 별도 브랜치에서 한 뒤 PR로 머지한다.
- 머지 후 로컬 `main`을 동기화하고 작업 브랜치를 정리한다.
  (원격 브랜치 삭제가 git 프록시로 막히면 GitHub의 `Delete branch` 버튼으로 안내한다.)

## 버전 관리 (index.html 수정 시)

`index.html`을 바꾸면 개발자 정보 상수를 함께 갱신한다:
`APP_VERSION`, `APP_VERSION_SHORT`(v1.x), `APP_UPDATED_AT`, `APP_FILE_FALLBACK`,
그리고 `UPDATE_HISTORY` 맨 앞에 한 줄 추가. 헤더 우측 상단 배지가 `APP_VERSION_SHORT`를 표시한다.

## 스킬

- 이 앱의 개발·수정·동기화·배포는 **`studyplan-dev`** 스킬을 따른다.
  - 활성 사본: `.claude/skills/studyplan-dev/`
  - 원본(단일 진실 공급원): `claude-skills-marketplace/plugins/bugeon-skills/skills/studyplan-dev/`
- 단일 HTML 앱 점검(보안 외 전수점검)은 `static-html-review` 스킬을 쓴다.
- 스킬 원본을 고치면 활성 사본에도 반영한다(두 곳은 별개 파일).

## 검증

- 프론트엔드 변경 후 JS 구문 검사(`new Function` 또는 `node --check`)를 하고,
  가능하면 헤드리스 Chromium(`/opt/pw-browsers/chromium`)으로 렌더를 확인한다.
- 완료 시 마지막에 결과 보고: 변경·검증·산출물 경로·`SQL 변경 필요`·`지침 업데이트`.
