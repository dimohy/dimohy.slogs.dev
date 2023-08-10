---
title: "`.vue`에서 중단점 설정하기"
datePublished: Thu Aug 10 2023 10:53:49 GMT+0000 (Coordinated Universal Time)
cuid: cll51k8na000x09jj5xrh7hxc
slug: visual-studio-code-vue-debug
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691663279279/88fb9a04-8e9e-4746-8b17-9d2d652a5f6c.jpeg
tags: vuejs, typescript

---

[이전 글](https://dimohy.slogs.dev/visual-studio-code-vuejs-typescript)에서 Visual Studio Code로 `Vue.js + TypeScript` 개발 환경을 구성했다면 `.js` 파일 또는 `.vue` 파일을 디버깅 하기 위해 중단점을 설정할 수 있어야 합니다.

하지만 웹브라우저에서 동작하는 코드는 소스 자체가 아니므로

1. 웹브라우저 디버거와 Visual Studio Code 디버거를 연결시켜야 하고
    
2. 중단점을 설정했을 때 동작하는 코드가 아닌 소스코드의 중단점이 표시되어야 함
    

`1.`은 이전에는 별도의 확장을 설치해야 했지만 Visual Studio Code에 `JavaScript Debugger` 확장이 기본 설치되어 있으므로 별도의 확장을 설치할 필요는 없습니다.

`2.`의 경우 `.ts` 및 `.vue`이 최종 웹브라우저에서는 적절한 `.js` 형식으로 변환되므로 설정을 해줘야 합니다.

## 웹팩 관련 설정

디버깅을 하기 위해서는 웹팩에 소스 맵을 이용해 특정 배포용 파일이 원본 소스의 어떤 부분인지 알려줘야 합니다. `vue.config.js` 파일을 프로젝트 루트에 다음의 코드로 생성합니다.

```typescript
module.exports = {
  configureWebpack: {
    devtool: 'source-map'
  }
}
```

## 디버깅 관련 설정

Visual Studio Code에서도 역시 웹팩의 특정 경로가 실제 어떤 경로와 매핑되었는지 힌트를 줘야 합니다. `launch.json`에 다음과 같이 설정할 수 있습니다.

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "chrome",
            "request": "launch",
            "name": "vuejs: chrome",
            "url": "http://localhost:5173",
            "webRoot": "${workspaceFolder}/src",
            "sourceMapPathOverrides": {
                "webpack:///./src/*": "${webRoot}/*",
                "webpack:///src/*": "${webRoot}/*",
                "webpack:///*": "*",
                "webpack:///./~/*": "${webRoot}/node_modules/*"
            }
        }
    ]
}
```

`url`은 `npm run dev` 로 실행되는 로컬의 url을 넣어줍니다. `webRoot` 및 `sourceMapPathOverrides` 을 통해 특정 위치의 파일이 어떤 소스코드 파일인지를 Visual Studio 디버거가 알 수 있도록 합니다.

## 중단점 설정 확인

이제 기본 생성된 프로젝트에서 `src/App.vue`의 코드를 다음과 같이 편집하여 중단점 설정이 잘 되는지 확인해봅시다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691664535580/ceafd7b2-15aa-4caf-b38a-b6377ff26f85.png align="center")

이때 F5로 디버깅을 시작하기 전에 `npm run dev`를 미리 실행해둬야 합니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691664739009/0bd4e7c5-2415-43e2-a635-b740c2e0163b.png align="center")

`.vue` 파일에 설정한 중단점이 잘 동작하는 것을 확인할 수 있습니다.

## 참고

[https://itinerant.tistory.com/188](https://itinerant.tistory.com/188)