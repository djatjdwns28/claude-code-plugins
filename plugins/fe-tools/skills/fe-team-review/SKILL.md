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

## 프로젝트 컨텍스트

- **프레임워크**: Nuxt 2 + Vue 2.7 (Nuxt Bridge, Webpack 기반)
- **언어**: TypeScript
- **상태관리**: Vuex (`store/`)
- **HTTP**: Axios (`@nuxtjs/axios`)
- **i18n**: vue-i18n (`$t()` 사용, 번역 키는 한국어 원문)

## 리뷰어 구성

| 리뷰어 | 전담 영역 |
|--------|----------|
| **convention-reviewer** | 코드 컨벤션 / 스타일 가이드 준수 |
| **evan-reviewer** | 타입 안전성 / 설계 / API 패턴 |
| **east-reviewer** | 에러 처리 / Null Safety / 비동기 / 보안 / SSR |
| **leo-reviewer** | 성능 / 번들 / 메모리 / UX / 접근성 |

## 심각도 기준 (통일)

| 심각도 | 기준 | 예시 |
|--------|------|------|
| **Critical** | 즉시 수정 필요. 버그/보안/런타임 에러 | 메모리 누수, XSS, null 에러 |
| **Major** | 수정 권장. 유지보수성/안정성 저하 | any 타입, 에러 처리 누락, SRP 위반 |
| **Minor** | 개선 고려. 코드 품질 향상 | 네이밍, 주석, 최적화 기회 |

## 리뷰 제외 파일

- `locales/*.json` — 자동 생성 번역 파일
- `models/spec/likey-api.ts` — 자동 생성 API 스펙
- `static/**/*` — 정적 에셋
- `CHANGELOG.md` — 자동 생성
- `package-lock.json`

## 공통 원칙

- **변경된 코드만 리뷰**합니다. 기존 코드의 이슈는 범위 밖입니다.
- diff에서 추가/수정된 라인만 대상입니다.
- 모든 리뷰는 **한국어**로 작성합니다.
- 각 이슈에는 반드시 **파일:라인** 참조를 포함합니다.

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

---

## Step 0.5: 결정론적 컨벤션 사전 스캔

AI 판단이 필요 없는 패턴을 grep으로 먼저 수집합니다. 이 결과를 convention-reviewer와 evan-reviewer 프롬프트에 주입합니다.

```bash
# 리뷰 대상 파일 목록 추출 (제외 대상 필터링)
git diff ${BASE}...HEAD --name-only \
  -- ':!locales/' ':!models/spec/likey-api.ts' ':!static/**' \
     ':!CHANGELOG.md' ':!package-lock.json' \
  > /tmp/review_files.txt && cat /tmp/review_files.txt
```

이후 아래 검사를 모두 실행하고 결과를 수집합니다:

```bash
# any 타입 사용 (신규 추가 라인만)
echo "=== any 타입 ===" && \
git diff ${BASE}...HEAD -- $(cat /tmp/review_files.txt | tr '\n' ' ') \
  | grep "^+" | grep -v "^\+\+\+" | grep -E ": any[,;\) ]|: any$"

# ref() 제네릭 누락
echo "=== ref without <T> ===" && \
grep -n "= ref(" $(cat /tmp/review_files.txt) 2>/dev/null | grep -v "ref<"

# computed() 제네릭 누락
echo "=== computed without <T> ===" && \
grep -n "= computed(" $(cat /tmp/review_files.txt) 2>/dev/null | grep -v "computed<"

# reactive() 제네릭 누락
echo "=== reactive without <T> ===" && \
grep -n "= reactive(" $(cat /tmp/review_files.txt) 2>/dev/null | grep -v "reactive<"

# export function 반환 타입 미명시
echo "=== missing return type ===" && \
git diff ${BASE}...HEAD -- $(cat /tmp/review_files.txt | tr '\n' ' ') \
  | grep "^+" | grep -v "^\+\+\+" \
  | grep -E "export (async )?function " | grep -v "): " | grep -v "void"

# unscoped style 블록
echo "=== unscoped style ===" && \
grep -n "<style" $(cat /tmp/review_files.txt) 2>/dev/null | grep "lang=" | grep -v "scoped"

# v-for :key 누락
echo "=== v-for without :key ===" && \
git diff ${BASE}...HEAD -- $(cat /tmp/review_files.txt | tr '\n' ' ') \
  | grep "^+" | grep -v "^\+\+\+" | grep "v-for" | grep -v ":key"

# 하드코딩 한국어 문자열 (i18n 미적용)
echo "=== hardcoded strings ===" && \
git diff ${BASE}...HEAD -- $(cat /tmp/review_files.txt | tr '\n' ' ') \
  | grep "^+" | grep -v "^\+\+\+" \
  | grep -E "\"[가-힣]|'[가-힣]|\`[가-힣]" \
  | grep -v "//.*[가-힣]" | grep -v "^\+\s*//"

# new Date() 사용
echo "=== new Date ===" && \
git diff ${BASE}...HEAD -- $(cat /tmp/review_files.txt | tr '\n' ' ') \
  | grep "^+" | grep -v "^\+\+\+" | grep "new Date("

# t()/$t() 뒤 불필요한 .toString()
echo "=== unnecessary toString() ===" && \
git diff ${BASE}...HEAD -- $(cat /tmp/review_files.txt | tr '\n' ' ') \
  | grep "^+" | grep -v "^\+\+\+" \
  | grep -E "\\\$t\(.*\)\.toString\(\)|[^a-zA-Z]t\(.*\)\.toString\(\)"
```

grep 결과를 `{GREP_RESULTS}` 변수로 보관합니다.

---

## Step 1: 4개 서브에이전트 병렬 스폰

**반드시 Agent 도구를 사용하여 4명의 리뷰어를 한 번의 응답에서 병렬로 스폰하세요.**

각 에이전트 프롬프트에 다음 공통 정보를 포함합니다:
- 변경된 파일 목록
- 리뷰 제외 파일 목록
- "변경된 코드만 검사하세요"
- 심각도 기준 (Critical / Major / Minor)
- 프로젝트 컨텍스트 (Nuxt 2 + Vue 2.7, Vuex, Axios, vue-i18n)

### 리뷰어별 추가 주입 정보

| 리뷰어 | 추가 주입 |
|--------|----------|
| convention-reviewer | `{GREP_RESULTS}` 전체 |
| evan-reviewer | `{GREP_RESULTS}` 중 타입 관련 항목 |
| east-reviewer | 없음 |
| leo-reviewer | 없음 |

### 스폰 방법

```
Agent(subagent_type="convention-reviewer") — 컨벤션/스타일
Agent(subagent_type="evan-reviewer") — 타입/설계/API
Agent(subagent_type="east-reviewer") — 에러/비동기/보안/SSR
Agent(subagent_type="leo-reviewer") — 성능/번들/UX/접근성
```

---

## Step 2: 교차 검증

4개 에이전트 결과를 받은 후 conductor가 직접 수행합니다.

### 2-1. Critical/Major 이슈 검증

해당 코드를 직접 Read/Grep 도구로 읽어서:
1. 실제 코드 확인 (라인 번호 정확성)
2. 실행 흐름 추적
3. 오탐이면 심각도를 Info로 하향 조정하고 이유 명시

### 2-2. 교차 비교

| 지적 리뷰어 수 | 처리 방법 |
|---------------|----------|
| 2명 이상 공통 지적 | 높은 신뢰도 표기, 심각도 상향 검토 |
| 1명만 지적 | `(단독 지적)` 표시, 오탐 가능성 주석 |
| 리뷰어 간 상충 | 양쪽 근거 기록, `(의견 상충)` 표시 |

### 2-3. 중복 제거

- 동일 파일:라인을 여러 리뷰어가 지적 → 한 항목으로 통합, 지적 리뷰어 명시
- 같은 문제를 다른 파일에서 지적 → 각각 별도 항목 유지

---

## Step 3: PR Review 게시 (인라인 코멘트 + 요약)

교차 검증된 결과를 **GitHub Pull Request Review API**로 게시합니다.
Critical/Major 이슈는 해당 파일의 특정 라인에 **인라인 코멘트**로, 나머지는 **요약(review body)**으로 게시합니다.

### 3-1. PR 번호 확인

```bash
PR_NUMBER=$(gh pr view --json number --jq '.number')
REPO=$(gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"')
```

### 3-2. 인라인 코멘트 대상 선별

| 이슈 심각도 | 게시 방식 |
|------------|----------|
| Critical / Major | **인라인 코멘트** (해당 파일:라인에 직접) |
| Minor | **요약 코멘트** (review body에 포함) |
| Good Practices | **요약 코멘트** |
| 종합 평가표 | **요약 코멘트** |

### 3-3. Review API로 게시

인라인 코멘트 배열과 요약을 하나의 review로 게시합니다.

```bash
# 리뷰 데이터를 JSON 파일로 작성
cat > /tmp/review-payload.json << 'EOF'
{
  "body": "## 팀 코드 리뷰 결과\n\n> 리뷰어: convention, evan, east, leo\n\n### 종합 평가\n| 영역 | Critical | Major | Minor |\n|------|----------|-------|-------|\n| ... | ... | ... | ... |\n\n### Minor 이슈\n- ...\n\n### Good Practices\n- ...",
  "event": "COMMENT",
  "comments": [
    {
      "path": "파일경로",
      "line": 라인번호,
      "body": "**CR-01. [이슈 제목]** (심각도)\n\n👤 리뷰어: ...\n🔍 문제: ...\n✅ 권장: ..."
    }
  ]
}
EOF

# Review API 호출
gh api repos/${REPO}/pulls/${PR_NUMBER}/reviews \
  --method POST \
  --input /tmp/review-payload.json
```

### 3-4. 인라인 코멘트 작성 규칙

각 인라인 코멘트의 `body` 형식:

```markdown
**CR-{순번}. [{심각도}] {이슈 제목}**

👤 **리뷰어**: {발견 리뷰어} {공통 지적 시: "(N명 공통)"}
🔍 **문제**: {설명}

❌ **문제 코드**:
\```
{문제 코드}
\```

✅ **권장 수정**:
\```
{수정 코드}
\```

💡 **이유**: {근거}
```

### 3-5. review body (요약) 형식

```markdown
## 팀 코드 리뷰 결과

> 리뷰어: convention-reviewer, evan-reviewer, east-reviewer, leo-reviewer
> 브랜치: {BRANCH} → {BASE}

### 종합 평가

| 영역 | Critical | Major | Minor |
|------|----------|-------|-------|
| 컨벤션 | N | N | N |
| 타입/설계/API | N | N | N |
| 에러/보안/SSR | N | N | N |
| 성능/UX/접근성 | N | N | N |
| **합계** | **N** | **N** | **N** |

> Critical/Major 이슈는 해당 파일에 인라인 코멘트로 게시되었습니다.

### Minor 이슈
[각 카테고리 최대 5개]

### Good Practices
[잘 작성된 코드 간단히 언급]
```

### 3-6. 라인 번호 주의사항

- Review API의 `line`은 **diff의 새 파일(RIGHT side) 기준** 라인 번호
- 새로 추가된 파일: diff 라인 = 파일 라인 (문제 없음)
- 수정된 파일: `git diff`의 `@@ -a,b +c,d @@` hunk에서 `+` 쪽 라인 기준
- 에이전트가 보고한 라인 번호를 Read 도구로 직접 확인 후 사용

---

## 주의사항

- 4명의 리뷰어는 반드시 **병렬(동시)** 실행합니다
- 교차 검증은 **conductor가 직접** 수행합니다 (에이전트 위임 금지)
- 각 카테고리는 최대 **5개 항목**으로 제한합니다
- grep 결과를 에이전트 프롬프트에 실제로 붙여넣어야 합니다
- Critical/Major만 인라인 코멘트로 게시하여 **노이즈를 최소화**합니다
- `event`는 기본 `"COMMENT"` 사용. Critical 있을 때만 `"REQUEST_CHANGES"` 검토
