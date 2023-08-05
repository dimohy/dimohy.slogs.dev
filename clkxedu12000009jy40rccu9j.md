---
title: "새로운 미니멀 Api 소스 생성기 살펴보기"
datePublished: Sat Aug 05 2023 02:30:36 GMT+0000 (Coordinated Universal Time)
cuid: clkxedu12000009jy40rccu9j
slug: exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator
canonical: https://andrewlock.net/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator/
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691199312934/b3d89c79-e92b-470b-b4a7-16dafb36f4b2.jpeg
tags: net, dotnet

---

> Andrew Lock님의 [Exploring the new minimal API source generator](https://andrewlock.net/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator/)를 DeepL의 도움을 받아 번역하였습니다.

이번 포스팅은 [.NET 8 미리 보기 살펴보기 시리즈](https://dimohy.hashnode.dev/series/exploring-dotnet8-preview)의 네 번째 포스팅입니다.

[1부 - 새로운 구성 바인더 소스 생성기 사용하기](https://dimohy.hashnode.dev/7ioi66gc7jq0ioq1royessdrsjtsnbjrjzqg7iam7iqkioydneyeseq4scdsgqzsmqk)  
[2부 - 미니멀 API AOT 컴파일 템플릿](https://dimohy.hashnode.dev/the-minimal-api-aot-compilation-template)  
[3부 - WebApplication.CreateBuilder()와 새로운 CreateSlimBuilder() 메서드 비교하기](https://dimohy.hashnode.dev/webapplicationcreatebuilder-createslimbuilder)  
4부 - 새로운 미니멀 API 소스 생성기 살펴보기(이 게시물)  
5부 - 메서드 호출을 인터셉터로 대체하기

이 시리즈에서는 .NET 8 프리뷰에 포함된 몇 가지 새로운 기능을 살펴봅니다. 이번 글에서는 AOT 워크로드를 지원하기 위해 도입된 새로운 미니멀 API 소스 생성기에 대해 살펴봅니다.

> 이 게시물은 모두 프리뷰 빌드를 사용하고 있으므로 2023년 11월에 .NET 8이 최종 출시되기 전에 일부 기능이 변경(또는 제거)될 수 있습니다!

## 미니멀 API는 어떻게 작동하나요?

미니멀 API는 ASP.NET Core를 보다 "미니멀" 시작 환경을 제공하기 위한 노력의 일환으로 .NET 6에 도입되었습니다. .NET 6 이전에는 기본 API 템플릿을 사용하면 일반적으로 최소한 3개의 클래스(Program.cs, Startup.cs, API 컨트롤러)와 여러 규칙을 이해해야 했습니다. 미니멀 API는 이러한 문제를 해결하기 위한 솔루션의 일부였습니다(`WebApplicationBuilder`과 연계하여)

미니멀 API를 사용하는 간단한 HelloWorld 솔루션은 다음과 같습니다.

```csharp
WebApplicationBuilder builder = new WebApplication.CreateBuilder(args);
WebApplication app = builder.Build();

app.MapGet("/", () => "Hello world!");

app.Run();
```

이 모든 것이 비교적 단순해 보이지만(그게 요점입니다!), 그 이면에는 `Map()` 함수가 꽤 많은 일을 하고 있습니다!

> 실제로 많은 일을 하고 있기 때문에 [8부작으로 구성된 시리즈 전체를 작성했습니다](https://andrewlock.net/behind-the-scenes-of-minimal-apis-1-a-first-look-behind-the-scenes-of-minimal-api-endpoints/)!

백그라운드에서 `RequestDelegateFactory`는 엔드포인트 핸들러로 전달된 델리게이트를 검사합니다(이 경우`() => "Hello World"`). 그런 다음 `RequestDelegateFactory`는 `Reflection.Emit` API를 사용하여 요청에 대한 응답으로 호출할 수 있는 `RequestDelegate`를 생성합니다. 이렇게 생성된 `RequestDelegate`는 많은 작업을 수행합니다.

* 인수 유형, HTTP 응답 유형 및 기타 사용자 정의 메타데이터와 같은 [핸들러에 대한 메타데이터를 추출](https://andrewlock.net/behind-the-scenes-of-minimal-apis-2-extracting-metadata-from-a-minimal-api-handler/)합니다.
    
* 요청에 바인딩하고 DI 컨테이너에서 서비스를 검색하는 등 [파라미터의 모델 바인딩](https://andrewlock.net/behind-the-scenes-of-minimal-apis-3-exploring-the-model-binding-logic-of-minimal-apis/)을 구현합니다.
    
* [응답에 대한 반환 값의 직렬화](https://andrewlock.net/behind-the-scenes-of-minimal-apis-6-generating-the-response-writing-expression/)를 구현합니다.
    
* [엔드포인트 또는 라우트 그룹에 적용된 필터로 핸들러 래핑](https://andrewlock.net/behind-the-scenes-of-minimal-apis-8-customising-the-request-delegate-with-filters/)을 처리합니다.
    

많은 작업이 필요하지만, 결과적으로 생성된 코드가 "손으로 롤링한" 코드와 최대한 비슷하도록 설계되었습니다. 즉, 필터와 같은 모든 추상화(예: 필터)에 대해 사용 여부에 관계없이 비용을 지불하는 MVC 프레임워크보다 훨씬 더 효율적입니다.

문제는 .NET 8의 기본 미니멀 API 설계가 AOT와 호환되지 않는다는 것입니다.

## 소스 생성기가 필요한 이유는 무엇인가요?

Ahead-of-time(AOT) 컴파일은 .NET 8에서 개발 중인 주요 기능 중 하나입니다. AOT .NET 애플리케이션에는 Just-in-time(JIT) 컴파일러가 없으며, 모든 컴파일은 런타임이 아닌 빌드 시점에 이루어집니다. [이전 게시물에서 설명했듯이](https://dimohy.hashnode.dev/the-minimal-api-aot-compilation-template) 시작 시간에는 유리할 수 있지만 단점도 있습니다. JIT가 없다는 것은 "플러그인 아키텍처" 애플리케이션이 없다는 것을 의미하며, 중요한 것은 미니멀 API인 `Reflection.Emit`도 없다는 것입니다!

미니멀 API를 .NET 8에서 AOT와 호환되게 만들려면 다른 접근 방식이 필요했습니다. 그리고 그 접근 방식에는 소스 생성기가 필요했습니다. 데이비드 파울러는 미니멀 API에 대한 AOT의 가능성을 탐구하는 문서에서 다음과 같이 말했습니다.

> ASP.NET Core와 같은 프레임워크, 그리고 다른 많은 프레임워크가 깔끔하게 정리되는 유일한 방법은 소스 생성기를 빌드하는 것입니다.

좋아, 소스 생성기가 필요하지만 실제로는 어떤 의미가 있을까요? 미니멀 API는 이미 존재하며 기존 메서드와 확장 메서드를 이미 호출합니다.

```csharp
app.MapGet("/", () => "Hello world!");
app.MapGet("/{name}", (string name) => "Hello {name}!");
```

소스 생성기는 어떻게 도움이 되나요?

곧 보게 되겠지만, AOT를 대상으로 하는 소스 생성기의 전반적인 설계는 현재 호출하고 있다고 생각하는 메서드를 AOT에 더 적합한 다른 메서드로 대체하는 것입니다. [이것이 이전 게시물에서 설명한 구성 소스 생성기에서 사용하는 접근 방식입니다](https://andrewlock.net/exploring-the-dotnet-8-preview-using-the-new-configuration-binder-source-generator/). 구성 소스 생성기는 컴파일러가 선호하는 `global::` 네임스페이스에 오버로드를 제공합니다. 생성된 AOT 친화적인 메서드는 "일반적인" 리플렉션 기반 API 대신 우선적으로 바인딩되고 호출됩니다.

> C#12의 일부인 인터셉터로 [.NET 8 프리뷰 6에서는 새로운 실험적 기능이 제공](https://devblogs.microsoft.com/dotnet/new-csharp-12-preview-features/#interceptors)되었습니다. 이 기능은 네임스페이스 트릭에 의존하여 생성된 메서드를 대신 호출하는 대신 훨씬 더 직접적인 방식으로 메서드를 "대체"하는 것을 목표로 합니다. 미니멀 API 생성기는 인터셉터를 지원하도록 [최근 업데이트](https://github.com/dotnet/aspnetcore/pull/48817)되었으며, 이 기능은 프리뷰 7에 추가될 예정입니다!

다시 미니멀 API 생성기로 돌아와서, 목표는 모든 `MapGet()` 및 `MapPost()` 메서드에 대한 오버로드를 생성하여 ["일반" 확장 메서드](https://github.com/dotnet/aspnetcore/blob/3a026d43392406eae01f956d12dbebbbcd99d97a/src/Http/Routing/src/Builder/EndpointRouteBuilderExtensions.cs#L230) 대신 생성기 메서드가 선택되도록 하는 것입니다. 이 기능에 대한 [원래 GitHub 이슈](https://github.com/dotnet/aspnetcore/issues/46163)에 설명된 대로:

> 요청 델리게이트 생성기의 목적을 위해 RouteEndpointDataSource에서 로직을 호출할 수 있지만 사용자가 메타데이터를 추론하고 요청 델리게이트를 생성하기 위한 사용자 지정 델리게이트를 전달할 수 있는 `Map`\` 메서드의 사용자 지정 구현을 제공하고자 합니다. 소스 생성기는 런타임 코드 생성을 사용하는 API 호출을 우회하기 위해 컴파일 시 생성된 구현으로 이 오버로드를 호출합니다.

이 모든 것이 다소 혼란스럽게 들릴 수 있습니다(제가 쓴 ["미니멀 API의 비하인드 스토리" 시리즈](https://andrewlock.net/series/behind-the-scenes-of-minimal-apis/)를 읽어보셨다면 그 이유를 아실 것입니다. 혼란스럽기 때문입니다😅 이해를 돕기 위해 실제 코드를 살펴보고 작동 방식을 살펴보는 것을 좋아하므로 그렇게 하도록 하겠습니다. 아주 간단한 미니멀 API 애플리케이션으로 시작하여 미니멀 API 소스 생성기를 활성화하고 어떻게 작동하는지 살펴보겠습니다!

## 미니멀 API 소스 생성기 활성화

미니멀 API 소스 생성기의 주요 목적은 AOT 친화적이므로 애플리케이션에서 AOT 게시를 활성화하면 소스 생성기가 자동으로 활성화됩니다. [이전 게시물에서 설명했듯이](https://dimohy.hashnode.dev/series/exploring-dotnet8-preview) .NET 8에 도입된 새로운 `api` 템플릿에는 `<PublishAot>true</PublishAot>`를 설정하는 `--aot` 옵션이 포함되어 있습니다. 이 템플릿을 사용 중이거나 프로젝트에서 `PublishAot=true`를 설정한 경우 이미 소스 생성기를 사용할 수 있습니다.

소스 생성기의 주요 초점은 분명히 AOT이지만, AOT를 사용하지 않더라도 시작 시간이 개선되는 것을 볼 수 있습니다. 미니멀 API가 런타임에 각 핸들러의 `RequestDelegate`에 대한 코드를 동적으로 생성하는 대신, 소스 생성기는 이 모든 작업을 미리 수행할 수 있습니다. 시작 시 작업량이 줄어들면 시작/첫 번째 요청 시간이 빨라질 수 있지만, 이는 제너레이터의 주요 목표가 아니라 잠재적인 이점일 뿐이라고 생각합니다.

> 미니멀 API 앱을 시작하고 단일 요청(런타임 코드 생성을 트리거하기 위해)을 전송한 다음 종료하는 [TimeitSharp를 사용](https://andrewlock.net/exploring-the-dotnet-8-preview-the-minimal-api-aot-template/#measuring-startup-time)하여 간단한 테스트를 실행했습니다. 소스 생성기를 활성화한 상태와 비활성화한 상태에서 모두 AOT 없이 테스트를 실행했습니다. 평균적으로(100회 실행) 소스 생성기를 사용하면 총 실행 시간이 50밀리초 정도 단축되었지만, 생성기를 사용하지 않았을 때는 시작 시간에 큰 롱테일이 있었습니다(~450밀리초 대 500밀리초). 따라서 소폭 개선되었지만 [전체 AOT의 개선에 비하면 아무것도 아닙니다](https://andrewlock.net/exploring-the-dotnet-8-preview-the-minimal-api-aot-template/#measuring-startup-time).

그럼에도 불구하고 AOT 퍼블리싱 없이 소스 생성기를 사용해보고 싶다고 가정해 보겠습니다. 다음을 사용하여 빈 .NET 8 프로젝트를 새로 생성합니다.

```bash
dotnet new web
```

> 기본적으로 .NET은 컴퓨터에 설치된 가장 최신 SDK를 사용합니다. [디렉터리에 global.json 파일을 배치](https://andrewlock.net/exploring-the-new-rollforward-and-allowprerelease-settings-in-global-json/)하여 어떤 SDK를 사용할지 제어할 수 있습니다. 예를 들어, 저는 일반적으로 리포지토리 루트 디렉터리에 기본적으로 최신 안정 릴리스(예: .NET 7)로 설정된 global.json을 저장한 다음, `allowPrerelease: true`를 설정하여 하위 폴더별로 미리 보기 SDK를 명시적으로 사용할 수 있도록 global.json을 추가합니다.

프로젝트 파일에 `EnableRequestDelegateGenerator`를 추가하여 최소 API 소스 생성기를 활성화할 수 있습니다.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <!-- 👇 Add this line -->
    <EnableRequestDelegateGenerator>true</EnableRequestDelegateGenerator>
  </PropertyGroup>

</Project>
```

여기까지입니다! 모든 것이 잘 되었다면 앱에서 어떤 차이점도 느끼지 못할 것입니다. 하지만 IDE의 Program.cs에서 app.MapGet() 호출에서 `F12`(정의로 이동)를 누르면 소스 생성 코드로 이동해야 합니다! 다음 섹션에서는 이 코드가 어떻게 작동하는지 살펴보겠지만, 먼저 생성기가 애플리케이션에서 여러 API를 처리하는 방법을 보여주기 위해 앱에 몇 가지 간단한 API를 추가해 보겠습니다.

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");
app.MapGet("/ping", () => "Pong!");
app.MapGet("/{name}", (string name) => "Hello {name}!");

app.Run();
```

이제 우리는 가지고 있습니다.

* 단순히 `string`을 반환하는 엔드포인트 2개
    
* `string` 경로 매개변수에 바인딩한 다음 `string`을 반환하는 엔드포인트 1개
    

이제 코드를 자세히 살펴보고 어떻게 작동하는지 확인해 보겠습니다!

## 소스 생성기를 사용하여 메서드 호출 가로채기

처음 `F12`를 누르면 소스에서 생성된 `MapGet()` 확장 메서드 중 하나로 이동합니다. 컴파일러가 일반 `EndpointRouteBuilderExtensions.MapGet()` 메서드에 바인딩하는 대신 생성된 메서드에 바인딩하는 것이 바로 이 접근 방식의 마법입니다!

> 이 게시물의 코드는 .NET 8 프리뷰 6에서 생성된 코드에 해당합니다. 이 코드의 세부 사항은 11월에 .NET 8이 출시되기 전에 변경될 것으로 예상합니다. [어쨌든 이미 변경되었으니까요!](https://github.com/dotnet/aspnetcore/pull/48817)

원래 확장자를 보면 서명은 다음과 같이 보입니다.

```csharp
public static RouteHandlerBuilder MapGet(
    this IEndpointRouteBuilder endpoints,
    string pattern,
    Delegate handler);
```

이와 대조적으로 생성된 메서드는 다음과 같이 보입니다.

```csharp
internal static RouteHandlerBuilder MapGet(
    this IEndpointRouteBuilder endpoints,
    string pattern,
    Func<string> handler,
    [CallerFilePath] string filePath = "",
    [CallerLineNumber]int lineNumber = 0)
```

이 두 메서드는 모두 `IEndpointRouteBuilder`의 확장이며, 두 번째 매개변수로 `string`을 받습니다. 하지만 원래 메서드의 세 번째 매개변수는 `Delegate`인 반면, 생성된 `MapGet()`의 세 번째 매개변수는 `Func<string>`입니다. 이것이 바로 마법입니다. 컴파일러는 `Delegate` 메서드보다 "더 구체적"이기 때문에 `Func<string>` 메서드를 선호합니다.

> 생성된 메서드에는 두 개의 [선택적 호출자 정보 매개변수](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/caller-information)(나중에 다시 설명하겠습니다)도 있습니다.

`Func<>` 핸들러 매개변수의 정확한 형식은 애플리케이션에서 정의한 엔드포인트 핸들러와 일치하도록 조정됩니다. 테스트 예제에서는 두 가지 형태만 있습니다: `string`만 반환하는 핸들러의 경우 `Func<string>`, `string`을 경로 매개변수로도 허용하는 핸들러의 경우 `Func<string, string>`입니다.

생성된 코드에서는 이러한 각 형식에 대해 별도의 `MapGet()` 오버로드가 생성됩니다. 다음은 앱에 대해 생성된 `MapGet()` 오버로드를 보여줍니다. `Func<string>`을 사용하는 첫 번째 `MapGet()`은 `/` 및 `/ping` 엔드포인트와 일치하고, `Func<string, string>`을 사용하는 두 번째 `MapGet()`은 `/{name}` 엔드포인트와 일치합니다.

```csharp
// This class needs to be internal so that the compiled application
// has access to the strongly-typed endpoint definitions that are
// generated by the compiler so that they will be favored by
// overload resolution and opt the runtime in to the code generated
// implementation produced here.
internal static class GenerateRouteBuilderEndpoints
{
    private static readonly string[] GetVerb = new[] { HttpMethods.Get };

    internal static RouteHandlerBuilder MapGet(
        this IEndpointRouteBuilder endpoints,
        string pattern,
        Func<string> handler, // 👈 Supports the / and /ping endpoints
        [CallerFilePath] string filePath = "",
        [CallerLineNumber]int lineNumber = 0)
    {
        return GeneratedRouteBuilderExtensionsCore.MapCore(
            endpoints, pattern, handler, GetVerb, filePath, lineNumber);
    }

    internal static RouteHandlerBuilder MapGet(
            this IEndpointRouteBuilder endpoints,
            string pattern,
            Func<string, string> handler, // 👈 Supports the /{name} endpoints
            [CallerFilePath] string filePath = "",
            [CallerLineNumber]int lineNumber = 0)
        {
            return GeneratedRouteBuilderExtensionsCore.MapCore(
                endpoints, pattern, handler, GetVerb, filePath, lineNumber);
        }
}
```

> 위의 샘플은 정규화된 네임스페이스와 일부 어트리뷰트를 제거하여 실제 생성된 코드와 비교하여 간소화했다는 점에 유의하세요. 소스 생성 코드에서는 항상 모든 유형에 대해 전체 네임스페이스를 지정하는 것이 좋습니다(예: `Func<string>` 대신 `global::System.Func<global::System.String>` 사용). 유형 확인 문제를 피할 수 있기 때문입니다.

새로운 `MapGet()` 오버로드는 모두 생성된 또 다른 확장 메서드인 `MapCore()` 메서드로 호출합니다.

```csharp
file static class GeneratedRouteBuilderExtensionsCore
{
    internal static RouteHandlerBuilder MapCore(
        this IEndpointRouteBuilder routes,
        string pattern,
        Delegate handler,
        IEnumerable<string>? httpMethods,
        string filePath,
        int lineNumber)
    {
        // Use the filePath and lineNumber as an index into the map dictionary
        var (populateMetadata, createRequestDelegate) = map[(filePath, lineNumber)];
        // Pass the functions to the minimal API internals
        return RouteHandlerServices.Map(routes, pattern, handler, httpMethods, populateMetadata, createRequestDelegate);
    }
}
```

여기서 흥미로운 점이 있습니다. `MapCore`는 `(string, int)` 튜플에 인덱싱되고 두 개의 `Func<>`로 구성된 튜플을 반환하는 `static` Dictionary로 호출합니다. 이러한 `Func<>`는 .NET 6 및 .NET 7에서 미니멀 API 엔드포인트를 컴파일하는 데 사용되는 `RequestDelegateFactory`의 메서드와 동일합니다.

```csharp
using MetadataPopulator = Func<MethodInfo, RequestDelegateFactoryOptions?, RequestDelegateMetadataResult>;
using RequestDelegateFactoryFunc = Func<Delegate, RequestDelegateFactoryOptions, RequestDelegateMetadataResult?, RequestDelegateResult>;
```

* `MetadataPopulator`는 이전 포스트에서 설명한 대로 엔드포인트의 매개변수에 대한 메타데이터를 추론하는 동일한 작업을 수행하며, 메서드 시그니처는 `RequestDelegateFactory.InferMetadata()`와 동일합니다.
    
* `RequestDelegateFactoryFunc`는 [이전 게시물에서 설명한 것처럼](https://andrewlock.net/behind-the-scenes-of-minimal-apis-7-building-the-final-requestdelegate/#creating-the-targetexpression) 일반적으로 실행 가능한 `RequestDelegate`를 생성하는 `RequestDelegateFactory.Create()`와 동일한 서명을 가집니다.
    

이러한 `Func<>`는 .NET 6 및 .NET 7에서 사용되는 런타임 생성 버전에 대한 컴파일 타임 드롭인 대체 함수입니다. 서명을 동일하게 유지하면 대체가 안정적이고 상대적으로 마찰이 적습니다.

> 이 접근 방식을 설명하는 GitHub 이슈는 이 PR에서 구현된 '[`InferMetadata` 및 `CreateRequestDelegate`에 대한 오버로드가 있는 맵 구현 추가](https://github.com/dotnet/aspnetcore/issues/46163)'입니다. [이 PR](https://github.com/dotnet/aspnetcore/pull/46180)에서는 항상 런타임 생성 버전을 사용하는 대신 이러한 함수를 매개변수로 사용하는 `MapCore()`에서 호출하는 `RouteHandlerServices.Map()` 메서드를 추가했습니다.

특히 흥미로운 것은 `map` 사전입니다. `GenerateRouteBuilderEndpoints.MapGet()` 오버로드는 소스 생성기 메서드가 호출되도록 보장하지만(컴파일러에서 `Delegate` 대신 `Func<string>`이 선택됨), `map`은 소스 생성기가 예제에서의 `() => "Hello World!"` 및 `() => "Pong!"`처럼 동일한 서명을 가진 두 개의 다른 엔드포인트를 구별하는 방식입니다.

> [새로운 "인터셉터" 기능](https://devblogs.microsoft.com/dotnet/new-csharp-12-preview-features/#interceptors)을 사용하면 이 사전이 필요하지 않으므로 작업이 다소 간소화됩니다. <s>하지만 인터셉터는 .NET 9까지 실험적으로 유지될 예정이므로 최소한의 API 소스 생성기에서는 기본적으로 사전 방식을 사용합니다</s>. [제가 틀렸네요](https://twitter.com/captainsafia/status/1682096427881889792)! [자세한 내용은 다음 포스팅을 참고](https://andrewlock.net/exploring-the-dotnet-8-preview-changing-method-calls-with-interceptors/)하세요.

아래 생성된 코드에서 딕셔너리의 `(string, int)` 키는 `MapGet()` 오버로드에서 전달된 `[CallerFilePath]` 및 `[CallerLineNumber]` 인수에서 오는 것을 볼 수 있습니다. 이 튜플은 고유하도록 보장되므로 딕셔너리 키로 완벽하게 작동하며, `(Func<>, Func<>)` 값은 미니멀 API 생성 `RequestDelegate` 함수를 정의합니다(아래 코드에서 `Func<>`는 너무 크기 때문에 생략했습니다!)

```csharp
file static class GeneratedRouteBuilderExtensionsCore
{
    private static readonly Dictionary<(string, int), (MetadataPopulator, RequestDelegateFactoryFunc)> map = new()
    {
        [(@"C:\repos\temp\temp25\Program.cs", 4)] = // () => "Hello World!"
        (
            (methodInfo, options) => { /*   */ }, // MetadataPopulator
            (del, options, inferredMetadataResult) => { /*   */ }, // RequestDelegateFactoryFunc
        ),
        [(@"C:\repos\temp\temp25\Program.cs", 5)] = // () => "Pong!"
        (
            (methodInfo, options) => { /*   */ }, // MetadataPopulator
            (del, options, inferredMetadataResult) => { /*   */ }, // RequestDelegateFactoryFunc
        ),
        [(@"C:\repos\temp\temp25\Program.cs", 6)] = // (string name) => "Hello {name}!"
        (
            (methodInfo, options) => { /*   */ }, // MetadataPopulator
            (del, options, inferredMetadataResult) => { /*   */ }, // RequestDelegateFactoryFunc
        ),
    }
}
```

생성된 `RequestDelegateFunc` 및 `MetadataPopulator` 함수 자체를 살펴보기 전에 소스에서 생성된 `MapCore()` 함수에 의해 호출되는 [새로운 `RouteHandlerServices.Map()` 오버로드를 간단히 살펴보겠습니다](https://github.com/dotnet/aspnetcore/blob/c343768fb12b048fe402999434ebf94bef2444af/src/Http/Routing/src/Builder/RouteHandlerServices.cs).

```csharp
public static class RouteHandlerServices
{
    /// <summary>
    /// Registers an endpoint with custom functions for constructing
    /// a request delegate for its handler and populating metadata for
    /// the endpoint. Intended for consumption in the RequestDelegateGenerator.
    /// </summary>
    public static RouteHandlerBuilder Map(
            IEndpointRouteBuilder endpoints,
            string pattern,
            Delegate handler,
            IEnumerable<string>? httpMethods,
            Func<MethodInfo, RequestDelegateFactoryOptions?, RequestDelegateMetadataResult> populateMetadata,
            Func<Delegate, RequestDelegateFactoryOptions, RequestDelegateMetadataResult?, RequestDelegateResult> createRequestDelegate)
    {
        return endpoints
              .GetOrAddRouteEndpointDataSource()
              .AddRouteHandler(RoutePatternFactory.Parse(pattern),
                               handler,
                               httpMethods,
                               isFallback: false,
                               populateMetadata,
                               createRequestDelegate);
    }
}
```

보시다시피, 이것은 매우 간단한 메서드이며, 먼저 `RouteEndpointDataSource`를 가져온 다음 ["일반적인" `MapGet()` 함수가 하는 것과 마찬가지로](https://andrewlock.net/behind-the-scenes-of-minimal-apis-1-a-first-look-behind-the-scenes-of-minimal-api-endpoints/#from-mapget-to-routeentry) `AddRouteHandler()`를 호출합니다. 여기서 유일한 차이점은 일반적으로 이러한 함수가 런타임에 생성되는 반면, 이 접근 방식은 메타데이터를 채우는 `Func<>`와 `RequestDelegate`를 생성하는 `Func<>`를 전달할 수 있는 새로운 `AddRouteHandler` 메서드를 사용한다는 점입니다.

생성된 코드가 생성된 `RequestDelegate`에 어떻게 삽입되는지 거의 다 설명했으니 이제 `RequestDelegateFunc`와 `MetadataPopulator` 함수를 살펴볼 차례입니다.

## 생성된 `RequestDelegateFunc` 살펴보기

이 섹션에서는 단일 엔드포인트에 대한 `MetadataPopulator`와 `RequestDelegateFunc`에 대해 살펴보겠습니다.

```csharp
app.MapGet("/{name}", (string name) => "Hello {name}!");
```

[이전 미니멀 API 시리즈에서](https://andrewlock.net/series/behind-the-scenes-of-minimal-apis/) 모델 바인딩이 백그라운드에서 어떻게 작동하는지, 미니멀 API가 다양한 매개변수와 반환 유형을 어떻게 처리하는지에 대해 자세히 설명했으므로 여기서는 그 내용을 모두 다루지 않겠습니다. 대신 간단한 엔드포인트의 '대표적인 예시'라고 생각하시면 됩니다. 좋은 소식은 소스 생성기 코드가 런타임 생성 코드와 거의 동일하게 보일 것이므로 해당 시리즈의 일반적인 원칙과 접근 방식이 여기에도 동일하게 적용된다는 점입니다.

이러한 함수는 `map` 사전에서 튜플의 두 부분으로 정의된다는 점을 기억하세요.

```csharp
file static class GeneratedRouteBuilderExtensionsCore
{
    private static readonly Dictionary<(string, int), (MetadataPopulator, RequestDelegateFactoryFunc)> map = new()
    {
        [(@"C:\repos\temp\temp25\Program.cs", 6)] = // (string name) => "Hello {name}!"
        (
            (methodInfo, options) => { /*   */ }, // MetadataPopulator
            (del, options, inferredMetadataResult) => { /*   */ }, // RequestDelegateFactoryFunc
        ),
    }
}
```

`MetadataPopulator` 구현부터 살펴보겠습니다. 간결성을 위해 가능한 경우 어설션과 네임스페이스는 제거했습니다.

```csharp
(methodInfo, options) =>
{
    options.EndpointBuilder.Metadata.Add(new SourceKey(@"C:\repos\temp\temp25\Program.cs", 6));
    options.EndpointBuilder.Metadata.Add(new GeneratedProducesResponseTypeMetadata(type: null, statusCode: StatusCodes.Status200OK, contentTypes: GeneratedMetadataConstants.PlaintextContentType));
    return new RequestDelegateMetadataResult { EndpointMetadata = options.EndpointBuilder.Metadata.AsReadOnly() };
},
```

보시다시피, 이 간단한 엔드포인트에는 추가할 메타데이터가 많지 않습니다. 단순한 `record`와 유사한 유형인 `SourceKey`가 추가되어 있지만, 필터에서 실제로 사용되는 곳은 찾을 수 없었습니다. 추가된 유일한 다른 메타데이터는 응답 유형으로, 항상 `200 OK` 및 `text/plain`으로 문서화되어 있습니다.

> 흥미롭게도 아래의 `RequestDelegateFactoryFunc`를 보면 이 엔드포인트가 실제로 `null` JSON 객체를 반환할 수도 있다는 것을 알 수 있으므로 여기 어딘가에 버그가 있을 수 있습니다 🤔 아직 제대로 파헤쳐서 확인하지는 못했습니다!

이제 마음을 가다듬고 `RequestDelegateFunc`를 살펴볼 시간입니다! 엔드포인트가 호출될 때 실제로 실행되는 코드이므로 모든 모델 바인딩 및 응답 직렬화 코드가 포함되어 있습니다. 읽기 쉽도록 아래에서 약간 정리하고 몇 가지 주석을 추가했지만, 그 외에는 본질적으로 변경되지 않았습니다. 따라서 따라하기 조금 어렵더라도 저를 탓하지 마세요😉

이 예제의 코드 중 일부는 모든 `RequestDelegateFunc`구현에 공통적으로 적용되지만, 일부는 엔드포인트에 특정되어 있습니다. 하지만 이 코드의 장점은 미니멀 API 앱에서 엔드포인트에 대한 중단점을 설정하고 디버깅할 때 단계별로 진행할 수 있다는 것입니다. 이는 미니멀 API의 백그라운드에서 일반적으로 일어나는 일에 대한 8부작 시리즈를 읽고 싶지 않은 사람들에게 매우 유용할 수 있습니다😅

```csharp
(Delegate del, RequestDelegateFactoryOptions options, RequestDelegateMetadataResult? inferredMetadataResult) =>
{
    var handler = (Func<string, string>)del; // The endpoint delegate cast to its native type
    EndpointFilterDelegate? filteredInvocation = null; // if the endpoint has any filters, this will be non-null later
    var serviceProvider = options.ServiceProvider ?? options.EndpointBuilder.ApplicationServices;

    // 👇 A helper type that is used to handle when the model binding is invalid, 
    // e.g. if a required argument is missing, or the body of the Request is 
    // not the right type. Either throws an exception or logs the error, depending
    // on your minimal API configuration
    var logOrThrowExceptionHelper = new LogOrThrowExceptionHelper(serviceProvider, options);

    // The JSON configuration to use when serializing to and from JSON
    var jsonOptions = serviceProvider?.GetService<IOptions<JsonOptions>>()?.Value ?? new JsonOptions();
    var objectJsonTypeInfo = (JsonTypeInfo<object?>)jsonOptions.SerializerOptions.GetTypeInfo(typeof(object));

    // Helper function that tries to fetch a named argument value. Tries the route parameters
    // first, and then the querystring
    Func<HttpContext, StringValues> name_RouteOrQueryResolver = 
        GeneratedRouteBuilderExtensionsCore.ResolveFromRouteOrQuery("name", options.RouteParameterNames);

    // If the endpoint has any filters, this builds and applies them, using more generated code
    // that I don't show in this post. Very similar to the non-source generated version.
    // See my previous blog post for details:
    // https://andrewlock.net/behind-the-scenes-of-minimal-apis-8-customising-the-request-delegate-with-filters/
    if (options.EndpointBuilder.FilterFactories.Count > 0)
    {
        filteredInvocation = GeneratedRouteBuilderExtensionsCore.BuildFilterDelegate(ic =>
        {
            if (ic.HttpContext.Response.StatusCode == 400)
            {
                return ValueTask.FromResult<object?>(Results.Empty);
            }
            return ValueTask.FromResult<object?>(handler(ic.GetArgument<string>(0)!));
        },
        options.EndpointBuilder,
        handler.Method);
    }

    // This is the RequestDelegate implementation that runs when the endpoint
    // does NOT have any filters. If the endpoint does have filters, a different method is
    // used, RequestHandlerFiltered, shown below.
    Task RequestHandler(HttpContext httpContext)
    {
        // Try to resolve the `name` argument
        var wasParamCheckFailure = false;
        // Endpoint Parameter: name (Type = string, IsOptional = False, IsParsable = False, IsArray = False, Source = RouteOrQuery)
        var name_raw = name_RouteOrQueryResolver(httpContext);
        if (name_raw is StringValues { Count: 0 })
        {
            wasParamCheckFailure = true;
            logOrThrowExceptionHelper.RequiredParameterNotProvided("string", "name", "route or query string");
        }

        // This is where any conversion to an int etc would happen, 
        // which is why this seems a bit odd and superfluous in this example!
        var name_temp = (string?)name_raw;
        string name_local = name_temp!;

        // If there was a binding failure, nothing more to do.
        if (wasParamCheckFailure)
        {
            httpContext.Response.StatusCode = 400;
            return Task.CompletedTask;
        }

        // Model binding was successful, so execute the handler. 
        var result = handler(name_local!);
        // Render the response (must be either a `string` or `null``)
        if (result is string)
        {
            httpContext.Response.ContentType ??= "text/plain; charset=utf-8";
        }
        else
        {
            // Note the JSON response here 🤔
            httpContext.Response.ContentType ??= "application/json; charset=utf-8";
        }
        return httpContext.Response.WriteAsync(result);
    }

    // This is the RequestDelegate implementation that runs when the endpoint
    // DOES have filters. The model binding is identical to `RequestHandler`, 
    // the difference is that after binding it calls the filter pipeline, and 
    // renders the response (whatever that may be)
    async Task RequestHandlerFiltered(HttpContext httpContext)
    {
        var wasParamCheckFailure = false;
        // Endpoint Parameter: name (Type = string, IsOptional = False, IsParsable = False, IsArray = False, Source = RouteOrQuery)
        var name_raw = name_RouteOrQueryResolver(httpContext);
        if (name_raw is StringValues { Count: 0 })
        {
            wasParamCheckFailure = true;
            logOrThrowExceptionHelper.RequiredParameterNotProvided("string", "name", "route or query string");
        }
        var name_temp = (string?)name_raw;
        string name_local = name_temp!;

        if (wasParamCheckFailure)
        {
            httpContext.Response.StatusCode = 400;
        }

        // Run the filter pipeline
        var result = await filteredInvocation(EndpointFilterInvocationContext.Create<string>(httpContext, name_local!));

        // Render the response of the filter pipeline
        if (result is not null)
        {
            await GeneratedRouteBuilderExtensionsCore.ExecuteReturnAsync(result, httpContext, objectJsonTypeInfo);
        }
    }

    // If there were any filters, use RequestHandlerFiltered, otherwise use RequestHandler
    RequestDelegate targetDelegate = filteredInvocation is null 
                                        ? RequestHandler 
                                        : RequestHandlerFiltered;
    var metadata = inferredMetadataResult?.EndpointMetadata ?? ReadOnlyCollection<object>.Empty;
    return new RequestDelegateResult(targetDelegate, metadata);
})
```

`BuildFilterDelegate()`나 `ExecuteReturnAsync()`와 같이 소스에서 생성된 메서드를 추가로 살펴볼 수 있지만, 프로젝트에서 (소스로 이동하여) 쉽게 볼 수 있고 (일반적인 미니멀 API에서 사용되는 `Reflection.Emit`에 비해) 읽기 쉽다는 점을 고려할 때 여기서 다룰 가치는 크지 않다고 생각합니다. 그래도 미니멀 API 앱을 AOT 친화적으로 만들기 위해 뒤에서 어떤 일이 벌어지고 있는지 간단히 살펴보는 데 도움이 되셨기를 바랍니다!

## 요약

이 글에서는 AOT 컴파일과 함께 사용할 수 있도록 .NET 8에 도입된 미니멀 API 소스 생성기를 살펴봤습니다. .NET 8 이전에는 미니멀 API에서 `Reflection.Emit`을 사용하여 모델 바인딩 및 직렬화를 위해 최적화된 코드를 생성했지만, 이 접근 방식은 AOT에서 지원되지 않습니다. 대신 .NET 8 이전 미니멀 API에서 사용되는 런타임 코드와 유사한 코드를 생성하는 새로운 소스 생성기가 도입되었습니다. 소스 생성기는 런타임이 아닌 컴파일 시간에 코드를 생성하며, 이 코드를 AOT 링커가 정적으로 분석하여 앱을 올바르게 트리밍할 수 있습니다. AOT 없이 소스 생성기를 활성화할 수도 있으므로 첫 번째 요청 시 앱이 런타임에 수행해야 할 작업이 줄어듭니다. 이는 큰 차이를 만들지는 못하지만, 예를 들어 서버리스 앱에서 풀 AOT를 사용할 수 없는 경우에 유용할 수 있습니다!