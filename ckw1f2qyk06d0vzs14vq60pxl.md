---
title: ".NET 6 에서 Serilog 설정 (번역)"
datePublished: Tue Nov 16 2021 01:28:25 GMT+0000 (Coordinated Universal Time)
cuid: ckw1f2qyk06d0vzs14vq60pxl
slug: net-6-serilog
tags: dotnet

---

> 요약: [완전한 기능을 갖춘 샘플을 GitHub](https://github.com/datalust/dotnet6-serilog-example)에서 확인 할 수 있습니다.

.NET 6 이 공식 출시되었습니다! 내부적으로 많은 변화가 있었으며, 새로운 프로젝트로 시작할 경우 표면적으로도 많은 것이 변경되었습니다.

`dotnet new web`으로 시작하는 최소 ASP.NET Core 애플리케이션:
- 명시적인 `Program` 클래스, `Startup` 클래스, `Main()` 메서드가 없으며
- 새로운 `WebApplicationBuilder`로 API를 구성합니다.

우리가 알고 사랑하는 모든 것이 여전히 거기에 숨겨져 있지만, Serilog 구성과 같은 간단한 작업조차 업데이트된 지침을 보증할 수 있을 만큼 충분히 변경되었습니다.


## .NET 6 에서 Serilog을 사용하는 이유는 무엇입니까?

Serilog는 여전히 .NET 6 애플리케이션과 관련이 있나요? 로그로 하고 싶은 일이 터미널에서 일부 텍스트를 보는 것이라면 아닐 수도 있습니다.

만약 로그를 구조화된 데이터 (애플리케이션을 계측하고 관찰 가능성을 높이는 일류 이벤트 스트림) 으로  처리하려는 경우, Serilog는 여전히 그 위치에 있습니다. Serilog는 타의 추종을 불가하는 출력 대상(싱크)를 선택할 수 있고 구조화된 로그 이벤트를 보강, 라우팅, 필터링 및 형식화 하는 기능을 통해 Serilog는 실제 애플리케이션에서 없어서는 안될 필수 요소입니다.

.NET 6 의 로깅 API와 프레임워크는 프로덕션 진단 및 분석에 유용한 풍부한 구조화 데이터를 생성할 수 있습니다. Serilog는 이 모든 마법의 잠금을 해제하는 방법입니다.


## 샘플

이 포스트는 바로 시작할 수 있는 최소한의 Serilog + .NET 6 구성을 보여줍니다. 우리는 [`datalust/dotnet-6-serilog-example` GitHub 리포지토리](https://github.com/datalust/dotnet6-serilog-example)에서 프로덕션 배포에 필요할 수 있는 모든 것이 포함된 완전한 샘플로 확장했습니다.

![.NET 6 Seq 웹 애플리케이션 로그](https://cdn.hashnode.com/res/hashnode/image/upload/v1637024214926/9s1gvlxIk.png)


## 시작하기

진행하려면 .NET 6 SDK가 필요합니다. 확실하지 않다면 터미널에서 `dotnet --version`을 실행해 보유한 버젼을 확인하세요.

`dotnet new web`으로 생성한 프로젝트는 CSPROJ 파일 및 구성 파일, 그리고 매우 간단한 Program.cs 파일이 있습니다.

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

글쎄요, 확실히 다릅니다!


## Serilog 패키지 추가

시작 하기 위해 필요한 Serilog.AspNetCore:

```
dotnet add package Serilog.AspNetCore
```

여기에는 Serilog 핵심 패키지, ASP.NET Core 구성 및 호스팅 하부 구조, 기본 싱크(출력) 및 향상된 요청 로깅을 위한 미들웨어가 포함됩니다.

선호하는 싱크에 대한 패키지도 필요합니다. Seq의 경우:

```
dotnet add package Serilog.Sinks.Seq
```


## `UseSerilog()` 호출

.NET 5 의 Serilog 구성에 익숙하다면 가장 먼저 하려고 할 일은 다음의 `WebApplicationBuilder`에서  `UserSerilog()`를 호출 하려고 하는 것입니다.

```csharp
using Serilog;

var builder = WebApplication.CreateBuilder(args)
    .UseSerilog(...);

// error CS1061: 'WebApplicationBuilder'에는 'UseSerilog'에 대한 정의가 포함되어 있지 않고, 'WebApplicationBuilder' 형식의 
// 첫 번째 인수를 허용하는 액세스 가능한 확장 메서드 'UseSerilog'이(가) 없습니다. using 지시문 또는 어셈블리 참조가 있는지 확인하세요.
```
아니요! 치즈는 이동했습니다; 🐁

필요로 하는 `IHostBuilder`는 `builder.Host`로 이동:

```csharp
using Serilog;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, lc) => lc
    .WriteTo.Console()
    .WriteTo.Seq("http://localhost:5341"));

var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

그리고 멋진 터미널의 Serilog 출력과 Seq에 구조화된 로그를 가져오기에 충분합니다.

`lc` 매개변수를 통해 Serilog 구성 API의 나머지 부분을 열고, 추가 속성으로 로그 이벤트를 쉽게 강화하고 이벤트를 더 많은 싱크로 보낼 수 있습니다.

**주의!** `builder.WebHost` 속성은 `builder.Host`와 매우 유사해 보이며 `UseSerilog()` 메서드도 노출하지만 이것은 레거시 통합 지점이며 최신 API 만큼 훌륭하게 작동하지 않습니다.


## 그림 완성하기

[샘플 GitHub 리포지토리](https://github.com/datalust/dotnet6-serilog-example)에서 시작 오류, 종료 시 플러시, 간소화된 요청 로깅, JSON 구성, Seq를 사용한 중앙 집중 로깅, 롤링 파일 로깅, 필터링 등을 처리하는 방법을 보여주는 모든 기능에 대한 예제를 찾을 수 있습니다.

.NET 6 의 시작 및 호스팅 변경 사항에 대해 자세히 알아보려면 [앤드류 록이 작성한 환상적인 게시물](https://andrewlock.net/exploring-dotnet-6-part-2-comparing-webapplicationbuilder-to-the-generic-host/)을 확인하세요.


## 원문
- [Setting up Serilog in .NET 6](https://blog.datalust.co/using-serilog-in-net-6/) | Seq | Nicholas Blumhardt
