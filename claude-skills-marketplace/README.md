# bugeon-skills — Claude Code 공용 스킬 마켓플레이스 (템플릿)

여러 프로젝트(StudyPlan, Medical Note 등)와 여러 기기(데스크탑·웹/모바일)에서
**같은 스킬을 한 곳에서 관리·업데이트**하기 위한 플러그인 마켓플레이스 템플릿입니다.

## 폴더 구조

```
.claude-plugin/
  marketplace.json          ← 마켓플레이스 정의 (플러그인 목록)
plugins/
  bugeon-skills/
    .claude-plugin/
      plugin.json           ← 플러그인 정의
    skills/
      static-html-review/
        SKILL.md            ← 실제 스킬 (마크다운 1개 = 스킬 1개)
README.md
```

스킬을 추가하려면 `plugins/bugeon-skills/skills/<새-스킬>/SKILL.md` 를 하나 더 만들면 됩니다.

## ⭐ 권장: 이 폴더를 "전용 저장소"로 옮기기

마켓플레이스는 **자기만의 저장소**에 두는 게 깔끔합니다. (앱 저장소 안에 두면 섞입니다.)

1. 새 빈 저장소를 만든다. 예: `hanwha27-TDTU/claude-skills`
2. 이 `claude-skills-marketplace/` **폴더의 내용물**을 그 저장소 루트로 복사한다.
   (즉 새 저장소 루트에 `.claude-plugin/`, `plugins/`, `README.md` 가 오도록)
3. 커밋·푸시한다.

## 설치 (각 기기에서 1회)

```
/plugin marketplace add hanwha27-TDTU/claude-skills
/plugin install bugeon-skills@bugeon-skills
```

설치하면 그 기기의 **모든 프로젝트에서** 스킬을 쓸 수 있습니다.

## 업데이트 흐름 (데스크탑 ↔ 웹/모바일 양방향)

스킬 파일이 git에 있으므로 동기화는 git으로 합니다.

- **수정한 쪽**: `SKILL.md` 수정 → `commit` → `push`
  - ⚠️ Claude에게 시킬 땐 **"수정하고 커밋·푸시까지 해줘"** 라고 해야 GitHub에 올라갑니다.
    (그냥 "수정해줘"는 로컬 파일만 바뀝니다.)
- **받는 쪽**: `git pull` 후 `/plugin update` 로 설치본 갱신.

### 웹/모바일에서 쓰려면
웹 세션은 임시 컨테이너라 데스크탑 설치 플러그인이 따라오지 않습니다. 둘 중 하나:
- (간단) 자주 쓰는 스킬은 각 앱 저장소의 `.claude/skills/` 에도 커밋해 두기
- (고급) 세션 시작 셋업에서 `/plugin marketplace add ...` 를 구성하기

## 매번 "커밋·푸시까지"를 자동으로

이 저장소의 `CLAUDE.md` 등에 아래 같은 규칙을 적어두면 매번 말하지 않아도 됩니다.

```
스킬(SKILL.md)을 수정하면 항상 commit 후 origin에 push 한다.
```
