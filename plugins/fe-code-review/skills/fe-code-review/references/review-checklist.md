# 프론트엔드 코드 리뷰 체크리스트

이 파일은 코드 리뷰 시 참조하는 팀 표준 체크리스트입니다.
프로젝트별로 필요에 맞게 커스터마이징할 수 있습니다.

---

## SFC 구조
- [ ] `<script setup lang="ts">` 사용 (Vue3/Nuxt3)
- [ ] SFC 블록 순서: script -> template -> style
- [ ] script setup과 일반 script 혼용 금지
- [ ] `<style scoped lang="scss">` 사용

## 템플릿 & 네이밍
- [ ] 컴포넌트명 PascalCase 사용 (`<VBtn />`, `<UserCard />`)
- [ ] Props는 명사형 (`userId`, `productList`)
- [ ] Emits는 동사형 (`updateUser`, `closeModal`)
- [ ] Boolean props는 `is/has/should` 접두어 (`isOpen`, `hasError`)
- [ ] `v-for`의 `:key` 적절성 (index 사용 지양)
- [ ] `v-if` 조건 복잡도 (3개 이상이면 computed 분리)
- [ ] 삼항 연산자 1 depth 제한
- [ ] Attribute 순서: 동적 props -> 정적 props -> boolean props

## TypeScript
- [ ] `any` 타입 사용 금지 (`unknown` 후 타입 가드 사용)
- [ ] `import type` 사용 (타입만 import 시)
- [ ] computed/ref/함수 반환 타입 명시
- [ ] Props/Emit 제네릭 타입 정의
- [ ] 타입 강제 캐스팅 (`as Type`) 지양 -> 타입 가드 사용
- [ ] 새 객체/파생 데이터 생성 시 타입 명시
- [ ] nullable 값은 옵셔널 체이닝 (`?.`) 또는 nullish coalescing (`??`) 사용

## API 타입 관리
- [ ] API 응답 타입 헬퍼 사용 (수동 타이핑 금지)
- [ ] DTO는 스키마 타입 참조
- [ ] 수동 DTO 중복 생성 금지
- [ ] API 응답 직접 전달 금지 -> 필요한 필드만 추출하여 전달

## 함수 & 코드 품질
- [ ] 화살표 함수 사용 (function 키워드 지양)
- [ ] 함수 단일 책임 원칙 (SRP)
- [ ] 비즈니스 로직은 composable/store/service로 분리
- [ ] 에러 처리: 공통 에러 핸들러 사용
- [ ] 메모리 누수 가능성 (observer, listener 정리)
- [ ] 불필요한 import/변수/console.log 제거
- [ ] 매직 넘버/문자열 상수화
- [ ] 함수 위에 "무엇을 하는지" 한 줄 주석

## Import 경로
- [ ] `~/` 또는 `@/` 절대경로 사용
- [ ] `../../` 상대경로 금지

## SSR
- [ ] 브라우저 API (`window`, `localStorage`) 사용 시 `process.client` 가드
- [ ] 서버/클라이언트 동작 다른 컴포넌트는 `<ClientOnly>` 사용

## 보안
- [ ] `window.open`에 `noopener,noreferrer` 옵션
- [ ] XSS 취약점 (`v-html`, `innerHTML` 사용 주의)
- [ ] 민감 정보 노출 여부
