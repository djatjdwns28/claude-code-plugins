---
name: component-gen
description: 팀 컨벤션에 맞는 컴포넌트 보일러플레이트를 생성합니다. "컴포넌트 생성", "component gen", "컴포넌트 만들어" 등을 말하면 실행됩니다.
user-invocable: true
allowed-tools: Bash, Read, Write, Grep, Glob, AskUserQuestion
argument-hint: "[ComponentName]"
---

# 컴포넌트 보일러플레이트 생성

팀 컨벤션에 맞는 Vue/React 컴포넌트 보일러플레이트를 생성합니다.

## 실행 흐름

### 1. 프로젝트 감지

프로젝트의 프레임워크를 자동 감지합니다:

```bash
# package.json에서 프레임워크 확인
cat package.json | grep -E '"vue"|"nuxt"|"react"|"next"'
```

| 감지 결과 | 생성 형식 |
|-----------|----------|
| Vue/Nuxt | `.vue` SFC |
| React/Next | `.tsx` |
| 감지 실패 | 사용자에게 선택 요청 |

### 2. 컴포넌트 이름 확인

인자로 전달된 이름이 없으면 사용자에게 물어봅니다.
- PascalCase로 변환 (예: `user-card` -> `UserCard`)

### 3. 생성 위치 확인

프로젝트 구조를 분석하여 적절한 위치를 제안합니다:

```bash
# 기존 컴포넌트 디렉토리 탐색
find . -type d -name "components" | head -5
```

AskUserQuestion으로 위치를 확인합니다:
- 감지된 components 디렉토리 제안
- 사용자 직접 입력 가능

### 4. 컴포넌트 유형 선택

AskUserQuestion으로 유형을 선택합니다:

| 유형 | 설명 |
|------|------|
| 기본 컴포넌트 | props, emits, 기본 템플릿 |
| 페이지 컴포넌트 | 라우트 페이지용 (Nuxt: definePageMeta, React: metadata) |
| 레이아웃 컴포넌트 | slot/children 기반 레이아웃 |
| 합성 컴포넌트 | composable/hook 포함 |

### 5. 파일 생성

#### Vue/Nuxt 컴포넌트 (기본)

```vue
<script setup lang="ts">
interface Props {
  // TODO: props 정의
}

interface Emits {
  // TODO: emits 정의
}

const props = defineProps<Props>()
const emit = defineEmits<Emits>()
</script>

<template>
  <div class="component-name">
    <!-- TODO: 템플릿 작성 -->
  </div>
</template>

<style scoped lang="scss">
.component-name {
  //
}
</style>
```

#### React/Next 컴포넌트 (기본)

```tsx
interface ComponentNameProps {
  // TODO: props 정의
}

export default function ComponentName({ }: ComponentNameProps) {
  return (
    <div className="component-name">
      {/* TODO: 템플릿 작성 */}
    </div>
  )
}
```

### 6. 추가 파일 생성 (선택)

AskUserQuestion(multiSelect=true)으로 추가 파일을 선택합니다:

| 파일 | 설명 |
|------|------|
| 테스트 파일 | `ComponentName.test.ts` — vitest/jest 기본 테스트 |
| 스토리북 | `ComponentName.stories.ts` — Storybook 스토리 |
| Composable/Hook | `useComponentName.ts` — 관련 로직 분리 |
| Index | `index.ts` — re-export |

### 7. 기존 패턴 참조

같은 디렉토리의 기존 컴포넌트를 분석하여 패턴을 맞춥니다:
- import 스타일
- 스타일 방식 (scss, tailwind, css modules 등)
- 파일 구조 (폴더별 분리 vs 단일 파일)

```bash
# 같은 디렉토리의 기존 파일 확인
ls <target-directory>/
```

### 8. 완료

생성된 파일 목록을 정리하여 표시:

```
컴포넌트 생성 완료!

생성된 파일:
- components/UserCard/UserCard.vue
- components/UserCard/UserCard.test.ts
- components/UserCard/index.ts

TODO:
- Props 인터페이스 정의
- 템플릿 작성
- 스타일 작성
```

## 사용 예시

```
사용자: 컴포넌트 만들어줘 UserProfile

AI: Vue/Nuxt 프로젝트를 감지했습니다.

컴포넌트 유형을 선택해주세요:
[기본 / 페이지 / 레이아웃 / 합성]

사용자: 기본

AI: 생성 위치를 선택해주세요:
- components/user/ (추천)
- components/

사용자: components/user/

AI: 추가 파일을 선택해주세요:
[x] 테스트 파일
[x] Index
[ ] 스토리북
[ ] Composable

AI: 컴포넌트 생성 완료!
- components/user/UserProfile.vue
- components/user/UserProfile.test.ts
- components/user/index.ts
```
