---
title: "싱글 파일 컴포넌트 (sfc)"
datePublished: Fri Aug 11 2023 02:12:41 GMT+0000 (Coordinated Universal Time)
cuid: cll5ydwsc000509mhggfg87kc
slug: sfc
tags: vuejs, typescript

---

Vue.js는 컴포넌트 템플릿, 로직 및 스타일을 하나의 파일로 관리하는 SFC 형식을 지원합니다. `.Vue` 파일이 그것인데요, Vue SFC에 대해 알아봅시다.

## SFC의 기본 구조

Vue SFC는 `.vue` 확장자를 가지며 `<script>`, `<template>`, `<style>` 블록으로 구성되어 있습니다. 각각 컴포넌트의 로직, 뷰, 스타일에 해당하며 하나의 컴포넌트로 캡슐화가 됩니다.

```xml
<script setup>
import { ref } from 'vue'
const greeting = ref('Hello World!')
</script>

<template>
  <p class="greeting">{{ greeting }}</p>
</template>

<style>
.greeting {
  color: red;
  font-weight: bold;
}
</style>
```

## 왜 SFC를 사용해야 하나요?

다음의 이점이 있습니다.

* 이미 친숙한 HTML, CSS 및 JavaScript(TypeScript)를 이용해 하나의 파일로 하나의 컴포넌트를 구성할 수 있음
    
* 런타임 시 파싱하지 않고 컴파일 시 처리되어 파싱 비용 없음
    
* 컴포넌트에만 적용되는 CSS (scoped CSS)
    
* 컴포지션 API 사용시
    
    * 더 적은 사용구로 간결한 코드 표현
        
    * 순수 TypeScript를 사용하여 props 및 내보낼 이벤트를 선언 가능
        
    * 더 나은 런타임 성능
        
    * 더 나은 IDE 타입 추론
        
* 더 빠른 컴파일 시간
    
* 자동 완성 및 유형 검사
    
* 즉시 사용 가능한 핫 모듈 교체(Hot-Module Replacement: HMR) 지원
    

## 작동 방식

Vue SFC는 완전한 JavaScript(ES) 모듈로 컴파일 되므로 다음과 같이 `import`할 수 있습니다.

```javascript
import MyComponent from './MyComponent.vue'

export default {
  components: {
    MyComponent
  }
}
```

참고로 개발시 `<style>`은 핫 업데이트를 위해 `<style>` 태그로 삽입되나 프로덕션 배포시에는 별도의 단일 CSS 파일로 추출되어 병합됩니다.

## 참고

[https://ko.vuejs.org/guide/scaling-up/sfc.html](https://ko.vuejs.org/guide/scaling-up/sfc.html)