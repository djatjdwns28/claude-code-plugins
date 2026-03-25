---
name: bundle-check
description: 현재 변경사항이 번들 사이즈에 미치는 영향을 분석합니다. "번들 체크", "bundle check", "번들 분석", "사이즈 체크" 등을 말하면 실행됩니다.
user-invocable: true
allowed-tools: Bash, Read, Grep, Glob, AskUserQuestion
---

# 번들 사이즈 영향 분석

현재 변경사항이 번들 사이즈에 미치는 영향을 분석하고 최적화를 제안합니다.

## 실행 흐름

### 1. 변경 파일 수집

```bash
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main")
git diff ${BASE_BRANCH}...HEAD --name-only
```

### 2. Import 분석

변경된 파일에서 새로 추가된 import를 분석합니다:

```bash
# 새로 추가된 import 라인 추출
git diff ${BASE_BRANCH}...HEAD | grep "^+" | grep -E "import |require\("
```

각 import에 대해:
- 패키지 이름 추출
- node_modules에서 해당 패키지 사이즈 확인
- tree-shaking 가능 여부 확인

### 3. 패키지 사이즈 체크

새로 추가된 외부 패키지에 대해:

```bash
# 패키지 사이즈 확인 (node_modules 기준)
du -sh node_modules/<package-name> 2>/dev/null
```

### 4. 위험 패턴 감지

| 패턴 | 위험도 | 설명 |
|------|--------|------|
| `import _ from 'lodash'` | Critical | 전체 lodash import. `lodash-es` 또는 `lodash/함수명` 사용 |
| `import moment from 'moment'` | Critical | moment.js 전체 import. `dayjs` 또는 `date-fns` 권장 |
| `import * as pkg from '...'` | Major | 전체 re-export. named import로 변경 |
| 사용하지 않는 import | Minor | dead code. 제거 권장 |
| 동적 import 미사용 | Minor | 큰 컴포넌트는 `defineAsyncComponent` / `lazy()` 고려 |
| 이미지 직접 import | Minor | 최적화 (webp, lazy loading) 확인 |

### 5. 최적화 제안

발견된 이슈별로 구체적인 최적화 방법 제시:

```
## 번들 분석 결과

### 새로 추가된 패키지
| 패키지 | 예상 사이즈 | 비고 |
|--------|------------|------|
| lodash | ~71KB (gzip ~25KB) | tree-shaking 불가 |
| axios | ~14KB (gzip ~5KB) | - |

### 위험 패턴
| # | 파일 | 패턴 | 제안 |
|---|------|------|------|
| 1 | utils.ts | lodash 전체 import | lodash-es/함수명 으로 변경 |
| 2 | Chart.vue | 동적 import 미사용 | defineAsyncComponent 적용 |

### 권장 조치
1. `import { debounce } from 'lodash-es'` 로 변경 (-65KB)
2. Chart 컴포넌트 lazy loading 적용
```

### 6. 빌드 사이즈 비교 (선택)

프로젝트에 빌드 명령어가 있으면 실제 빌드 비교를 제안합니다:

```bash
# 현재 브랜치 빌드 사이즈
npm run build 2>&1 | tail -20
```

## 에러 처리

| 상황 | 대응 |
|------|------|
| node_modules 없음 | "npm install을 먼저 실행해주세요" |
| package.json 없음 | "프로젝트 루트에서 실행해주세요" |
| 변경 파일 없음 | "변경사항이 없습니다" |

## 사용 예시

```
사용자: 번들 체크해줘

AI: 변경된 파일 5개를 분석합니다...

## 새로 추가된 import
- lodash (전체): ~71KB
- chart.js: ~200KB

## 위험 패턴 2개 발견
1. lodash 전체 import -> lodash-es로 변경 권장 (-65KB)
2. chart.js -> 동적 import 권장

총 예상 영향: +271KB -> 최적화 시 +206KB
```
