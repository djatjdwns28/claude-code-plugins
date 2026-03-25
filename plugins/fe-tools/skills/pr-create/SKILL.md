---
name: pr-create
description: PR을 자동으로 생성합니다. "PR 올려줘", "PR 생성", "create PR", "PR 만들어" 등을 말하면 실행됩니다.
user-invocable: true
allowed-tools: Bash, Read, Grep, Glob, AskUserQuestion
---

# PR 자동 생성

현재 브랜치의 변경사항을 분석하여 PR을 자동으로 생성합니다.

## 실행 흐름

### 1. 현재 브랜치 및 상태 확인

```bash
# 현재 브랜치 확인
git branch --show-current

# base 브랜치 자동 감지
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main")

# 커밋 히스토리 확인
git log ${BASE_BRANCH}..HEAD --oneline

# 변경 파일 통계
git diff ${BASE_BRANCH}...HEAD --stat
```

현재 브랜치가 base 브랜치(main/master)이면 중단:
> "base 브랜치에서는 PR을 생성할 수 없습니다. feature 브랜치로 이동해주세요."

### 2. Base Branch 선택

AskUserQuestion으로 base branch를 선택합니다:
- main / master / develop 등
- 사용자 직접 입력 가능

### 3. PR 제목 생성

#### 3-1. 브랜치명에서 이슈 번호 추출

브랜치명에서 이슈 번호를 추출합니다.
예: `feature/LK-30303-some-desc` -> `LK-30303`
예: `fix/JIRA-123-bug` -> `JIRA-123`

이슈 번호가 없으면 사용자에게 직접 입력 요청합니다.

#### 3-2. Prefix 선택

AskUserQuestion으로 prefix를 선택합니다:

| Prefix | 설명 |
|--------|------|
| feat | 새 기능 |
| fix | 버그 수정 |
| refactor | 리팩토링 |
| chore | 기타 작업 |
| perf | 성능 최적화 |
| style | 스타일 수정 |
| docs | 문서 |
| test | 테스트 |
| ci | CI/CD |

#### 3-3. 제목 설명 생성

커밋 히스토리와 diff를 분석하여 간결한 설명을 자동 생성합니다.

#### 3-4. 최종 제목 형식

```
<prefix>: <설명> (<이슈번호>)
```

예: `feat: 사용자 프로필 페이지 추가 (LK-30303)`

사용자에게 제목을 보여주고 수정 기회를 줍니다.

### 4. PR 본문 생성

변경사항을 분석하여 본문을 자동 생성합니다:

```markdown
## Summary
- 주요 변경사항 요약 (자동 생성)

## Changes
- 변경된 파일별 설명

## Checklist
- [ ] 개발자 테스트 진행 완료
- [ ] 코드 리뷰 완료
```

### 5. Label 선택

AskUserQuestion(multiSelect=true)으로 label을 선택합니다.
리포에 존재하는 label 목록을 `gh label list`로 조회하여 보여줍니다.

### 6. PR 미리보기 및 확인

생성할 PR 정보를 사용자에게 보여줍니다:

```
PR 미리보기
────────────────────────────
Base: main <- Head: feature/LK-30303
제목: feat: 사용자 프로필 페이지 추가 (LK-30303)
Labels: enhancement

본문:
[본문 내용 표시]
────────────────────────────

이대로 생성할까요?
```

### 7. PR 생성

```bash
gh pr create \
  --base <base-branch> \
  --title "<제목>" \
  --body "<본문>" \
  --label "<label1>" \
  --label "<label2>"
```

### 8. 완료 안내

```
PR 생성 완료!
- PR: <PR_URL>
```

## 에러 처리

| 상황 | 대응 |
|------|------|
| base 브랜치에서 실행 | "feature 브랜치에서 실행해주세요" |
| gh CLI 미설치 | "gh CLI가 필요합니다: brew install gh" |
| remote에 push 안됨 | "먼저 git push를 실행해주세요" |
| 이슈 번호 추출 실패 | 사용자에게 직접 입력 요청 |
| PR 생성 실패 | 에러 메시지 표시 후 수동 생성 안내 |
