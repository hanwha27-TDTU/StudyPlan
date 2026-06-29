# 설치 · 배포 가이드 (StudyPlan)

> `studyplan-dev` 스킬 참고 문서. 진입점은 상위 폴더의 SKILL.md다.
> StudyPlan은 **Supabase + GitHub Pages만** 쓴다. Cloudinary·Edge Function·Supabase Storage는 없다.

## 1. 전체 구조

```text
앱 HTML (단일 파일)
  ├─ localStorage : 일정/목표/체크/설정 (오프라인 기본 동작)
  └─ Supabase     : 기기 간 동기화 (study_plan_records + language_sync_meta)
```

이미지·파일 저장소가 없으므로 배포 산출물은 사실상 **`index.html` 하나**다.

## 2. Supabase 설정

1. Supabase 프로젝트 생성 → `Project Settings → API`에서 **Project URL**과 **anon public key** 확인.
   - 앱에는 **anon public key**만 넣는다. `service_role` key·DB 비밀번호는 앱/GitHub에 절대 넣지 않는다.
2. `SQL Editor → New query`에 앱이 제공하는 통합 SQL(클라우드 모달의 `SQL_TEXT`) 붙여넣고 Run.
   - 생성 테이블: `study_plan_records`, `language_sync_meta`. RLS 정책(anon all)까지 포함.
   - 기존 버전에서 `id` 단독 PK였던 경우를 위한 PK 보정 블록(`do $$ ... end $$;`)이 들어 있으니 그대로 실행.
3. 앱의 `☁ 클라우드` 모달에서 **URL · anon key · 동기화 ID**를 입력하고 "연결/저장 후 동기화".
   - 여러 기기에서 **같은 동기화 ID**를 쓰면 데이터가 공유된다.

> Supabase 코드(테이블/컬럼/메타 row)를 바꾸면 앱 내부 SQL 상수 · 개발자 정보의 SQL · 업데이트 이력 · 최종 답변 SQL을 **같은 버전으로** 맞춘다. 완료 보고에 `SQL 변경 필요`를 명시.

## 3. GitHub Pages 배포

### 3.1 올릴 파일 / 올리면 안 되는 값
- 올림: `index.html` (최신 HTML). 그게 전부.
- **올리지 않음:** Supabase `service_role` key, DB 비밀번호, 개인 토큰. (anon key는 공개키라 앱에 들어가지만, 리포에 비밀값을 커밋하지 않는 습관 유지.)

### 3.2 Pages 설정
```text
Settings → Pages
Source: Deploy from a branch
Branch: main   Folder: /(root)
```
- 루트에 `index.html`이 있으면 폴더 주소(`https://<사용자>.github.io/<repo>/`)로 자동으로 열린다.
- 최신 작업 파일명이 `YYMMDD_HH.MM ...html`이면, 배포용으로는 그 내용을 **`index.html`로 복사**해 올린다.

### 3.3 데스크탑 — git 직접 push (자동화)
배포 = `main` 루트의 `index.html` 교체일 뿐. 배포 repo를 클론해 **최신 HTML → `index.html` 복사 → commit → push**.

- **사전 준비(1회):** ① `git config --global user.name/user.email` ② repo를 HTTPS로 클론 ③ 인증 — Git for Windows의 Git Credential Manager가 있으면 첫 push 때 브라우저 로그인 1회로 끝(gh CLI/PAT 불필요). 없으면 `gh auth login` 또는 PAT.
- **머신별 구체값(repo 주소·클론 경로·Pages URL)은 스킬이 아니라 사용자 메모리에 둔다.**
- 완료 보고에 푸시한 커밋/버전·Pages URL, 첫 배포면 "브라우저 로그인 1회 필요"를 적는다.

### 3.4 모바일/웹에서의 배포
- 일반 모바일(claude.ai 앱): 로컬 git이 없으므로 새 HTML은 채팅/아티팩트로 전달하고 **배포는 데스크탑에서** 안내.
- **claude.ai code(원격 컨테이너):** git이 있으므로 여기서 직접 `commit`·`push`로 GitHub 반영 가능. 단 컨테이너는 임시이므로 **push해야 보존**되고, 데스크탑에 반영하려면 데스크탑에서 `git pull` 필요.

## 4. 캐시 / 배포 확인 체크리스트

1. GitHub Pages 주소에서 앱이 열린다.
2. 새 버전이 안 보이면 강력 새로고침(캐시) — 개발자 정보의 `APP_VERSION`/`APP_UPDATED_AT`로 실제 반영 버전 확인.
3. Supabase 연결 테스트 성공 → 즉시 동기화 → 다른 기기에서 같은 동기화 ID로 데이터 보이는지.
4. 백업 내보내기 → 로컬 초기화 → 복원 → 일정/목표/체크가 모두 돌아오는지.

## 5. 빠른 복구 (새 PC/브라우저)
1. GitHub Pages 또는 최신 HTML 열기.
2. `☁ 클라우드`에 Supabase URL/anon key/동기화 ID 입력 → "클라우드 최종본 받기".
3. 동기화 후 일정/목표/체크 표시 확인.
