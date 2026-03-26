---
name: fe-team-review
description: |
  4명의 전문 리뷰어 에이전트가 병렬로 코드를 분석하는 팀 기반 코드 리뷰.
  "팀 리뷰", "team review", "팀 코드리뷰" 등을 말하면 실행됩니다.
user-invocable: true
allowed-tools: Bash, Read, Grep, Glob, AskUserQuestion, Agent, ToolSearch
---

# FE 팀 코드 리뷰

4명의 전문 에이전트가 병렬로 코드를 분석하는 팀 기반 코드 리뷰 시스템입니다.

## 리뷰어 구성

| 리뷰어 | 역할 | 조건 |
|--------|------|------|
| **convention-reviewer** | 코드 컨벤션/스타일 준수 여부 | 항상 |
| **evan-reviewer** | 타입 안전성/설계/API 패턴 | 항상 |
| **east-reviewer** | 에러 처리/Null Safety/비동기 | 항상 |
| **leo-reviewer** | 성능/번들 사이즈/메모리/UX | 항상 |

---

## Step 0: 변경사항 수집

```bash
# 현재 브랜치
BRANCH=$(git branch --show-current)

# base 브랜치 결정
BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "master")

# 변경된 파일 목록
git diff ${BASE}...HEAD --name-only

# 변경 통계
git diff ${BASE}...HEAD --stat

# 커밋 목록
git log ${BASE}..HEAD --oneline
```

변경된 파일 수와 목록을 사용자에게 표시합니다.

**리뷰 제외 파일** (건너뛰기):
- `locales/*.json`
- `models/spec/likey-api.ts`
- `static/**/*`
- `CHANGELOG.md`

---

## Step 1: 4개 서브에이전트 병렬 스폰

**반드시 Agent 도구를 사용하여 4명의 리뷰어를 한 번의 응답에서 병렬로 스폰하세요.**

각 에이전트의 프롬프트에 다음을 포함합니다:
- 변경된 파일 목록 (파일 타입별 분류)
- 각 파일의 변경 내용 (diff)
- "변경된 코드만 검사하세요"

### 리뷰어 1: convention-reviewer
```
Agent(subagent_type="convention-reviewer")
```
- 에이전트 정의: `agents/convention-reviewer.md`
- CLAUDE.md의 코드 리뷰 기준과 에이전트 지침에 따라 검사

### 리뷰어 2: evan-reviewer
```
Agent(subagent_type="evan-reviewer")
```
- 에이전트 정의: `agents/evan-reviewer.md`
- 타입 안전성, 설계, API 패턴 관점에서 분석

### 리뷰어 3: east-reviewer
```
Agent(subagent_type="east-reviewer")
```
- 에이전트 정의: `agents/east-reviewer.md`
- 에러 처리, Null Safety, 비동기 처리 관점에서 분석

### 리뷰어 4: leo-reviewer
```
Agent(subagent_type="leo-reviewer")
```
- 에이전트 정의: `agents/leo-reviewer.md`
- 성능, 번들 사이즈, 메모리, UX 관점에서 분석

---

## Step 2: 교차 검증

에이전트 결과에서 **Critical/Major 이슈**를 추출하고:
1. 해당 코드를 직접 Read/Grep 도구로 읽어서 실행 흐름을 추적
2. 실제로 재현 가능한지 확인
3. 오탐이면 심각도를 Info로 하향 조정하고 이유를 명시

---

## Step 3: 통합 리포트 생성

아래 형식으로 최종 리뷰 결과를 출력하세요:

```markdown
## 팀 코드 리뷰 결과

> 리뷰어: convention-reviewer, evan-reviewer, east-reviewer, leo-reviewer
> 브랜치: {BRANCH} → {BASE}

### 종합 평가

| 영역 | Critical | Major | Minor |
|------|----------|-------|-------|
| 코드 컨벤션 | N | N | N |
| Evan 리뷰 | N | N | N |
| East 리뷰 | N | N | N |
| Leo 리뷰 | N | N | N |
| **합계** | **N** | **N** | **N** |

### Critical/Major 이슈 (즉시 수정 필요)

각 이슈:
- 📍 **위치**: `파일명:라인번호`
- 👤 **리뷰어**: 발견한 리뷰어 이름
- 🔍 **문제**: 설명
- ❌ **문제 코드** / ✅ **권장 수정**: 코드 블록
- 💡 **이유**

### 코드 컨벤션 리뷰
[convention-reviewer 결과 요약 — 최대 5개]

### Evan 리뷰
[evan-reviewer 결과 요약 — 최대 5개]

### East 리뷰
[east-reviewer 결과 요약 — 최대 5개]

### Leo 리뷰
[leo-reviewer 결과 요약 — 최대 5개]

### Good Practices
[잘 작성된 코드 간단히 언급]
```

---

## 주의사항

- 모든 리뷰는 반드시 한국어로 작성합니다.
- 각 카테고리는 최대 5개 항목으로 제한합니다.
- 변경된 코드만 리뷰합니다. 기존 코드의 이슈는 범위 밖입니다.
