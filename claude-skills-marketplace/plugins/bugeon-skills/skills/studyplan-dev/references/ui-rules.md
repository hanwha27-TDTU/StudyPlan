# UI / 상태 회귀 방지 (StudyPlan)

> `studyplan-dev` 스킬 참고 문서. 진입점은 상위 폴더의 SKILL.md다.
> Medical Note의 UI 규칙 중 StudyPlan에 실제로 해당하는 것만 추렸다(표 편집기·이미지·검사배지 등은 제외).

## 상태 ↔ 화면 동기화 (가장 자주 나는 회귀)

- **데이터를 바꾸는 핸들러는 의존 UI를 전부 다시 그린다.** StudyPlan에서 완료체크(`toggleCheck`)는 표 행뿐 아니라 상단 진행률 바, 7일 성취 패널, 시계 세그먼트 ✓까지 바뀐다 → `renderTable()` + `updateDateNav()` + **`updateUI()`** 를 함께 호출. 하나라도 빠지면 "체크했는데 %가 안 변함" 버그.
- 목표 추가/삭제(`addGoal`/`removeGoal`)는 `renderGoals()`로 충분(진행률에 영향 없음). 영향 범위를 따져 필요한 것만 호출하되, **빠뜨리지는 않는다.**
- `innerHTML`로 패널을 다시 그리면 입력 포커스/커서가 사라진다. 입력 중인 영역은 결과 영역만 다시 렌더하거나 `requestAnimationFrame`으로 focus/selection 복원.

## 시간·날짜

- 사용자가 입력/기본값으로 채우는 날짜는 `toISOString().slice(0,10)`로 만들지 않는다. UTC라 한국시간 자정 전후로 전날로 밀린다. **`getFullYear/getMonth/getDate` 기반 로컬 날짜 helper**(`todayStr`, `dateKey`)를 쓴다.
- "현재 진행 중" 배너·시계 바늘은 **실시간(`new Date()`)** 기준이고, 진행률·체크는 **선택일(`viewDate`)** 기준이다. 두 시간축을 섞어 혼란 주지 않도록, active-row 같은 "지금" 강조는 `viewDate===오늘`일 때만 적용한다.
- 시계 캔버스는 1초 인터벌(`tickClock`)로 다시 그려 초침이 끊기지 않게 하고, 무거운 전체 UI(`updateUI`)는 30초 간격 유지.

## overnight(자정 넘김) 블록

- `end < start`이면 overnight. 충돌/갭/시계 호/현재블록 판정은 `segIntervals`로 `[start,24]`,`[0,end]` 두 구간으로 펼쳐 처리한다(`isOvernight`, `inBlock`, `blocksCollide`).
- 새 블록을 추가/편집해 기존과 겹치면 편집본이 이기고 이웃을 잘라낸다(`resolveOverlaps`/`clipBlockAgainst`). 완전히 덮인 이웃은 tombstone 처리.
- "현재 진행 중 블록 없음"(일정 사이 빈 시간)일 때 첫 블록으로 폴백하지 말고 빈 시간대 안내를 표시한다.

## 모달

- 긴 입력 편집창(시간대 추가/수정)은 **배경 클릭으로 닫지 않는다**(입력 보호). 닫기는 ✕/취소/저장과 **ESC**로만.
- 클릭 가능한 카드 안의 버튼(체크/수정/삭제)은 `event.stopPropagation()`으로 카드 동작과 섞이지 않게 한다.
- 모달을 열 때 시간 피커는 `setTimeout(...,0)`으로 DOM 생성 후 빌드하고, 저장 시 `pStart/pEnd` 미생성 race를 가드한다.

## 모드 / 도메인 이름 변경 전파

- 모드(학기중/휴일/방학)나 표시명을 바꾸면 헤더(`modeHeading`), 모드 버튼, toolbar 제목, 저장 상태 문구, 백업 라벨까지 같은 이름으로 통일한다.
- 새 도메인을 추가하면 목록 표시만으로 끝내지 말고 추가/편집·삭제·tombstone·일반동기화·최종본저장·백업/복원·저장본 버전·SQL·항목 수 카운트를 한 번에 점검(SKILL 원칙 4).
- 목표 헤더처럼 "오늘"이라 쓰면서 실제로는 모드별 저장이면 표시 단위와 저장 단위를 일치시킨다.

## 반응형 / 출력

- 레이아웃 변경은 데스크톱·태블릿·모바일 폭을 나눠 검증하고, `document.scrollingElement.scrollWidth <= innerWidth`로 가로 스크롤이 생기지 않는지 확인한다. 모바일은 표를 카드형으로 전환(`@media(max-width:900px)`).
- A4 인쇄는 캔버스를 이미지로 그려 새 창에 넣고, `onafterprint`로 창을 닫는다(0.5초 타이머로 강제로 닫으면 인쇄가 잘릴 수 있음).
- 출력 이스케이프(`escapeHTML`)는 사용자 입력을 DOM에 넣는 모든 경로(표·배너·목표 태그·개발자 정보)에 일관 적용. `safeColor`로 색상값을 검증.

## 개발자 정보 모달

- 표시 항목: 앱 이름·버전(`APP_VERSION`/`APP_VERSION_SHORT`), 코드 최종 수정 시각, 현재 파일명·저장본(`APP_FILE_FALLBACK` — 파일명을 직접 못 읽는 환경용 fallback), 업데이트 이력(`UPDATE_HISTORY`).
- 디자인: 아이덴티티 헤더(아이콘+앱명+개발자·소속) + 날짜 카드 + 파일명 카드 + 이력(최신 강조 + 그리드 + 펼치기). 본문 대시보드보다 조밀하게.
- 이력이 길면 최신 1~2개만 기본 노출, 과거는 접기/펼치기. 버전·날짜·파일명·이력 문구는 코드 변경 시 함께 갱신.

## 셔플 / 무작위 (해당 시)

- 순서 무작위화가 의미 있는 곳(있다면)은 `array.sort(()=>Math.random()-0.5)` 금지 → Fisher-Yates. 우선순위가 의미 있는 목록은 셔플하지 않는다.
