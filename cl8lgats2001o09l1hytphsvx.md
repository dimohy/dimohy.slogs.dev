---
title: ".net으로 웹어셈블리 애플리케이션 시작하기"
datePublished: Wed Sep 28 2022 09:54:33 GMT+0000 (Coordinated Universal Time)
cuid: cl8lgats2001o09l1hytphsvx
slug: first-dotnet-webassembly
tags: webassembly, dotnet

---

.NET은 [Blazor Webassembly](https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor)를 통해 SPA(Single Page Application) 서비스를 제작할 수 있는 환경을 이미 제공하고 있습니다.

또한 .NET RC1 이후부터 `wasm-experimental` 워크로드를 설치하면 Blazor를 사용하지 않고도 웹브라우저 및 콘솔에서 동작하는 웹어셈블리용 애플리케이션을 만들 수 있습니다.

> wasm-experimental 워크로드는 이후 .NET 8에 정식 릴리스 될 예정입니다.


## 워크로드 설치 및 프로젝트 생성

> .NET 버전이 RC1 이상이여야 합니다.

`wasm-tools` 및 `wasm-experimental` 워크로드를 설치합니다.

```
dotnet workload install wasm-tools
dotnet workload install wasm-experimental
```

그 다음 웹브라우저용 웹어셈블리 템플릿으로 프로젝트를 생성합니다.


```
dotnet new wasmbrowser
```


## 컴파일 및 실행

컴파일

```
dotnet build
```

실행

> `dotnet-serv`가 설치되지 않았을 경우 `dotnet tool install --global dotnet-serve`로 설치합니다.

```
dotnet serv --directory bin\Debug\net7.0\browser-wasm\AppBundle
```

> RC1 버전에서 `dotnet run`으로 실행되지 않는 문제가 있습니다. 버전 RC2에서는 개선이 된다고 합니다.

다음처럼 콘솔에서 서버가 시작되고,
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1664355416447/qRT0YZD2M.png align="left")

해당 주소로 접속하면 다음처럼 .NET으로 컴파일 된 웹어셈블리 코드가 잘 실행되는 것을 확인할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1664355470691/zJtYhEQSX.png align="left")


## 게시

다음의 명령으로 배포할 수 있습니다. 게시된 경로는 `bin\Release\net7.0\browser-wasm\AppBundle` 입니다.
```
dotnet publish -c Release
```


## 트림 설정 및 AOT 컴파일

프로젝트에 다음의 설정을 추가해서 트림 및 AOT 컴파일을 할 수 있습니다.

| csproj
```
...
	  <PublishTrimmed>true</PublishTrimmed>
	  <TrimMode>full</TrimMode>
	  <RunAOTCompilation>true</RunAOTCompilation>
...
```

이제 게시를 했을 때 트림 및 AOT 컴파일로 인해 좀 더 컴파일 시간이 늘어나지만 좀 더 빠른 시작 시간 및 동작속도의 웹어셈블리 파일을 생성할 수 있습니다.


## 프로젝트 둘러보기

템플릿에 의해 기본 생성되는 프로젝트는 다음의 구조입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1664356753495/4ASM9xH8t.png align="left")

html이 로딩되면서 웹어셈블리를 시작해야 하는데요, 그것을 담당하는 곳이 `main.js` 입니다. 이곳에서 관련 초기화 및 .NET 메소드를 호출하는 코드와 .NET의 `main()` 호출 및 매개변수를 전달하는 코드를 확인할 수 있습니다.

```javascript
// Licensed to the .NET Foundation under one or more agreements.
// The .NET Foundation licenses this file to you under the MIT license.

import { dotnet } from './dotnet.js'

const is_browser = typeof window != "undefined";
if (!is_browser) throw new Error(`Expected to be running in a browser`);

const { setModuleImports, getAssemblyExports, getConfig, runMainAndExit } = await dotnet
    .withDiagnosticTracing(false)
    .withApplicationArgumentsFromQuery()
    .create();

setModuleImports("main.js", {
    window: {
        location: {
            href: () => globalThis.window.location.href
        }
    }
});

const config = getConfig();
const exports = await getAssemblyExports(config.mainAssemblyName);
const text = exports.MyClass.Greeting();
console.log(text);

document.getElementById("out").innerHTML = `${text}`;
await runMainAndExit(config.mainAssemblyName, ["dotnet", "is", "great!"]);
```

`Program.cs`를 보면 JavaScript에서 호출 할 수 있도록 `[JSExport]` 특성을 사용할 수 있고, JavaScript 함수를 호출할 수 있도록 `[JSImport]` 특성을 사용하고 `partial` 키워드를 줘서 코드 생성기를 통해 해당 interop 코드를 생성합니다.


## 배포

생성된 AppBundle은 정적 파일(static files)이므로 정적 파일을 올려서 웹주소로 접근 가능하면 실행할 수 있습니다. `bin\Release\net7.0\browser-wasm\AppBundle`의 파일을 배포할 위치로 업로드 합니다.

github 페이지 기능을 이용해 .NET 웹어셈블리 애플리케이션을 실행해 볼 수 있습니다.

https://dimohy.github.io/FirstWebassembly/


## 정리

웹어셈블리 환경은 격리된 환경에서 플랫폼에 상관없이 서비스를 제공할 수 있는 멋진 환경입니다. .NET으로 만들 수 있는 웹어셈블리 애플리케이션의 가능성은 무궁무진합니다. 앞으로 .NET 웹어셈블리로 동작하는 다양한 서비스가 등장하길 기대해봅니다.
