---
title: "Visual Studio Code로 Vue.js + TypeScript 개발환경 구성"
datePublished: Thu Aug 10 2023 05:26:18 GMT+0000 (Coordinated Universal Time)
cuid: cll4pv1qe000809mkfbsdbwa5
slug: visual-studio-code-vuejs-typescript
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691645171137/49a2aad2-b7d4-48c4-a806-c1c64ed961d6.jpeg
tags: vuejs, typescript

---

> .NET 개발자의 Vue.js + TypeScript 도전기 시리즈를 시작합니다. 본 시리즈는 최대한 짧은 주제로 꾸준히 글을 올리는 것을 목표로 합니다.

저는 .NET 개발자로 Vue.js의 경험은 전무하며 TypeScript 언어는 스터디 수준으로 알고 있으며, Visual Studio Code로 개발 환경을 구축하는데 경험이 부족합니다. 본 글은 저와 비슷한 위치에 있는 분 또는 초급 개발자가 Visual Studio Code에서 Vue.js + TypeScript 구성으로 개발을 시작할 수 있도록 기록합니다.

## Vue.js란?

Vue.js(간단히 Vue [/vjuː/](https://en.wikipedia.org/wiki/Help:IPA_for_English), 뷰)는 웹 애플리케이션 사용자 인터페이스를 만들기 위한 오픈소스로 프로그레시브 자바스크립트 프레임워크입니다. Vue.js 3부터 TypeScript에 대한 지원이 강화되었습니다. 다른 라이브러리와의 결합이 용이하며 점진적으로 적용할 수도 있다고 합니다. SPA(Single Page Application) 앱을 구축하는데 사용할 수 있습니다.

컴포넌트로 화면 단위를 구성할 수 있으며 컴포넌트가 잘 구축이 되었다면 복잡한 화면도 수정 시 일관성을 가지게 됩니다. (예: A페이지에서는 수정되었는데 B페이지에서는 수정되지 않는 문제 없음)

기본적인 Vue.js의 컴포넌트 구성은 다음과 같습니다. 스크립트와 템플릿, 스타일이 하나의 컴포넌트로 구성됩니다.

```xml
<script setup lang="ts">
defineProps<{
  msg: string
}>()
</script>

<template>
  <div class="greetings">
    <h1 class="green">{{ msg }}</h1>
    <h3>
      You’ve successfully created a project with
      <a href="https://vitejs.dev/" target="_blank" rel="noopener">Vite</a> +
      <a href="https://vuejs.org/" target="_blank" rel="noopener">Vue 3</a>. What's next?
    </h3>
  </div>
</template>

<style scoped>
h1 {
  font-weight: 500;
  font-size: 2.6rem;
  position: relative;
  top: -10px;
}

h3 {
  font-size: 1.2rem;
}

.greetings h1,
.greetings h3 {
  text-align: center;
}

@media (min-width: 1024px) {
  .greetings h1,
  .greetings h3 {
    text-align: left;
  }
}
</style>
```

## Vue.js + TypeScript 프로젝트 생성

### npm 설치

npm은 JavaScript 언어를 위한 패키지 관리자입니다. [node.js를 설치](https://nodejs.org/ko)하면 npm을 사용할 수 있게 됩니다.

### create-vue를 이용한 프로젝트 생성

create-vue를 이용해 쉽게 Vue.js + TypeScript 프로젝트를 생성할 수있습니다. 다음의 명령을 사용합니다.

```powershell
npm create vue@latest
```

이후 프로젝트명 등 각종 설정을 비활성/활성화 할 수 있습니다. 이 때 `Add TypeScript`를 선택해서 TypeScript용으로 프로젝트를 구성할 수 있습니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691643019901/fe7eaf02-c41e-43df-8407-593dda994c24.png align="center")

이후 `npm install` 및 `npm run format`, `npm run dev`를 통해 생성된 프로젝트를 실행해 볼 수 있습니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691643191702/599f51c1-61fe-417b-bf9e-889d46f34e5c.png align="center")

`npm run dev`로 실행된 로컬 주소 `http://localhost:5173/`을 통해 다음의 실행 화면을 확인할 수 있습니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691643271075/33077d42-08f6-47a0-825a-2ff4bf2e2e4c.png align="center")

## Visual Studio Code 개발 환경 구성

[Visual Studio Code를 설치](https://code.visualstudio.com/download) 했다면 프로젝트 디렉토리에서 `code .`를 통해 Visual Studio Code에 진입할 수 있습니다.

```powershell
PS W:\Learnings\vue-learning\vue-test> code .
```

개발에 필요한 개개의 확장을 별도로 설치해도 되지만 Visual Studio Code에서 필요로 하는 확장을 안내해주는 것을 설치해도 됩니다. `package.json`을 선택했을 때 다음의 권장 패키지를 설치할 수 있습니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691644477606/586d0f73-8399-415d-8156-7ec7314795bf.png align="center")

이후 Visual Studio Code에서 Vue.js + TypeScript 개발을 진행할 수 있습니다.