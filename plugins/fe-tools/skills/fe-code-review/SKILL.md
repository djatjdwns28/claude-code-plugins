---
name: fe-code-review
description: 현재 브랜치의 변경사항을 파일별로 인터랙티브하게 코드 리뷰합니다. "코드 리뷰", "code review", "fe review", "리뷰 시작" 등을 말하면 실행됩니다.
user-invocable: true
allowed-tools: Bash, Read, Grep, Glob, AskUserQuestion, Task, WebSearch
---

# 프론트엔드 코드 리뷰

현재 브랜치의 변경사항을 base 브랜치와 비교하여 파일별로 인터랙티브하게 코드 리뷰를 진행합니다.

## 리뷰 프로세스

### 0. 사전 자동 분석 (Pre-Analysis)

파일별 리뷰 전, 전체 변경사항에 대해 자동 분석을 수행합니다.

```bash
# base 브랜치 자동 감지 (master 또는 main)
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "master")

# 변경된 파일 목록 가져오기
git diff ${BASE_BRANCH}...HEAD --name-only
```

변경된 파일들을 대상으로 다음 3가지 관점에서 자동 스캔:

#### Security Scan
- `v-html`, `innerHTML` 직접 사용
- `window.open` 보안 옵션 누락 (`noopener,noreferrer`)
- 민감 정보 하드코딩 (.env 값, API 키, 토큰)
- XSS 취약점 패턴
- `eval()`, `Function()` 사용

#### Performance Check
- computed 내부 API 호출
- 불필요한 watch 트리거 (deep watch 남용)
- 이벤트 리스너 미해제 (메모리 누수)
- 대용량 배열 직접 렌더링 (가상 스크롤 필요)
- 불필요한 reactive/ref 사용

#### Architecture Compliance
- composable 미사용 비즈니스 로직
- store 직접 접근 vs getter 사용
- SSR 호환성 (`process.client` 가드)
- 컴포넌트 책임 분리 (SRP)

**분석 결과 형식:**
```
## 사전 분석 결과

| 카테고리 | 발견 이슈 | 심각도 |
|----------|----------|--------|
| Security | v-html 사용 (file.vue:42) | Critical |
| Performance | computed 내 API 호출 (api.ts:15) | Major |
| Architecture | composable 미사용 로직 | Minor |

총 N개 이슈 발견. 파일별 상세 리뷰를 시작합니다.
```

---

### 1. 브랜치 및 변경사항 확인

먼저 현재 브랜치와 변경된 파일들을 확인합니다:

```bash
# 현재 브랜치 확인
git branch --show-current

# base 브랜치 이후 커밋 목록
git log ${BASE_BRANCH}..HEAD --oneline

# 변경 파일 통계
git diff ${BASE_BRANCH}...HEAD --stat
```

사용자에게 리뷰 대상을 확인합니다:
- 현재 브랜치 (base와 비교)
- 특정 PR
- 특정 파일/디렉토리

### 2. 파일 목록 정리

변경된 파일들을 순서대로 나열합니다:
- 새로 추가된 파일 (신규)
- 수정된 파일
- 삭제된 파일

에셋 파일(이미지, SVG 등)은 목록에 포함하되 리뷰 시 간단히 패스할 수 있습니다.

### 3. 파일별 인터랙티브 리뷰

각 파일에 대해 다음 순서로 진행합니다:

#### 3-1. 변경사항 표시

```bash
git diff ${BASE_BRANCH}...HEAD -- [파일경로]
```

변경사항을 보기 좋게 정리하여 표시:
- **변경 요약**: 무엇이 변경되었는지 한 줄 설명
- **변경 코드**: 주요 변경 부분을 코드 블록으로 표시
  - 스크립트, 템플릿, 스타일 등 섹션별로 구분
  - 긴 코드는 핵심 부분만 발췌

#### 3-2. 사용자 리뷰 의견 요청

> "이 파일에 대한 리뷰 의견 있으신가요?"

사용자가 먼저 의견을 제시하도록 기다립니다.

#### 3-3. AI 리뷰 의견 제시 (다중 전문가 관점 포함)

사용자 의견에 대해:
- **동의**: "동의합니다" + 추가 설명
- **부분 동의**: 동의하는 부분과 다른 의견 제시
- **반대**: 이유와 함께 대안 제시

추가로 발견한 이슈가 있으면 함께 제시:
- 타입 안전성
- 잠재적 버그
- 성능 이슈
- 코드 스타일/일관성
- 베스트 프랙티스

**다중 전문가 관점 분석:**

각 이슈에 대해 3가지 관점으로 평가합니다:

| 관점 | 평가 내용 |
|------|----------|
| Security | 보안 위험도, 취약점 여부, 공격 벡터 |
| Performance | 성능 영향, 최적화 필요성, 리소스 사용 |
| Architecture | 유지보수성, 확장성, 패턴 준수, 결합도 |

**분석 예시:**
```
[Issue] computed 내부에서 API 호출

Security: 위험 없음 - 인증 토큰이 헤더에서만 전달됨
Performance: Critical - 매 렌더링마다 API 호출 가능, 불필요한 네트워크 요청
Architecture: Major - useFetch/useAsyncData로 분리하여 캐싱 및 SSR 지원 필요

권장: composable로 분리하고 useFetch 사용
```

#### 3-4. 리뷰 코멘트 작성

합의된 내용을 바탕으로 리뷰 코멘트 문구를 제안합니다:

```
**파일:** [파일경로]

**코멘트 1 (N행):**
> [코멘트 내용 - 질문 형식으로 부드럽게]

**심각도:** Critical / Major / Minor
```

코멘트 톤 가이드:
- 질문 형식으로 부드럽게 ("~한 이유가 궁금합니다!")
- 제안 형식 ("~하면 어떨까요?")
- 긍정적 피드백도 포함

#### 3-5. 다음 파일로 이동

사용자가 "다음", "넘어가자", "패스" 등을 말하면 다음 파일로 진행합니다.

### 4. 리뷰 체크 항목

리뷰 시 `references/review-checklist.md`의 체크리스트를 참조합니다.
체크리스트는 프로젝트별로 커스터마이징할 수 있습니다.

### 5. 리뷰 완료

모든 파일 리뷰가 끝나면:
- 전체 리뷰 요약 제공
- 심각도별 이슈 개수
- 전체 품질 점수 (선택적)

### 6. 개선 제안 & 참고 자료

발견된 이슈 유형별 공식 문서 및 베스트 프랙티스 링크를 제공합니다.

**참고 자료 테이블:**

| 이슈 유형 | 참고 자료 |
|----------|----------|
| Composition API | Vue3 Composition API Guide |
| Data Fetching | Nuxt3 Data Fetching |
| TypeScript | Vue3 + TypeScript |
| SSR | Nuxt3 Server-Side Rendering |
| State Management | Pinia Best Practices |
| Security | OWASP XSS Prevention |
| Performance | Vue3 Performance Guide |

**동작 방식:**
- WebSearch를 활용하여 발견된 이슈에 맞는 최신 문서 링크 검색
- 각 이슈별 구체적인 해결 방법과 코드 예시 제공
- 프로젝트 코드베이스 내 유사 패턴 참조 (있는 경우)

---

## 심각도 기준

| 심각도 | 설명 | 예시 |
|--------|------|------|
| Critical | 즉시 수정 필요, 버그/보안 이슈 | 메모리 누수, 타입 에러 |
| Major | 수정 권장, 유지보수성 저하 | any 타입, 에러 처리 부족 |
| Minor | 개선 고려, 코드 품질 향상 | 변수명, 주석, 일관성 |

## 사용 예시

```
사용자: 코드 리뷰 시작해줘
AI: 현재 브랜치 `feature-xxx`의 변경사항을 확인합니다...

## 파일 1/10: `components/Example.vue`
[변경 코드 표시]

이 파일에 대한 리뷰 의견 있으신가요?

사용자: computed 반환 타입이 없는 것 같아
AI: 동의합니다
[리뷰 코멘트 제안]

다음 파일로 넘어갈까요?

사용자: 다음
AI: ## 파일 2/10: ...
```

## 단축 명령어

- "다음", "넘어가자", "패스" -> 다음 파일로
- "스킵" -> 현재 파일 리뷰 없이 넘어감
- "너 의견은?" -> AI 리뷰 의견 요청
- "리뷰 끝", "완료" -> 리뷰 종료 및 요약
