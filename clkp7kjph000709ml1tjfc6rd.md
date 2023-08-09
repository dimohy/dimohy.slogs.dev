---
title: "새로운 구성 바인더 소스 생성기 사용"
datePublished: Sun Jul 30 2023 08:57:43 GMT+0000 (Coordinated Universal Time)
cuid: clkp7kjph000709ml1tjfc6rd
slug: exploring-the-dotnet-8-preview-using-the-new-configuration-binder-source-generatorwhy-do-we-need-more-source-generators
canonical: https://andrewlock.net/exploring-the-dotnet-8-preview-using-the-new-configuration-binder-source-generator/#why-do-we-need-more-source-generators-
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1690707538952/125bceba-383e-46e4-b55c-ea91f5426747.jpeg
tags: net, dotnet

---

> [Andrew Lock](https://andrewlock.net/)님의 [Using the new configuration binder source generator](https://andrewlock.net/exploring-the-dotnet-8-preview-using-the-new-configuration-binder-source-generator/#why-do-we-need-more-source-generators-)를 [DeepL](https://www.deepl.com/translator)의 도움을 받아 번역하였습니다.

이 글은 [.NET 8 미리 보기 살펴보기 시리즈](https://dimohy.hashnode.dev/series/exploring-dotnet8-preview)의 첫 번째 글입니다.

1부 - 새로운 구성 바인더 소스 생성기 사용(이 게시물)  
[2부 - 미니멀 API AOT 컴파일 템플릿](https://dimohy.hashnode.dev/the-minimal-api-aot-compilation-template)  
[3부 - WebApplication.CreateBuilder()와 새로운 CreateSlimBuilder() 메서드 비교하기](https://dimohy.hashnode.dev/webapplicationcreatebuilder-createslimbuilder)  
[4부 - 새로운 미니멀 API 소스 생성기 살펴보기](https://dimohy.hashnode.dev/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator)  
[5부 - 메서드 호출을 인터셉터로 대체하기](https://dimohy.hashnode.dev/exploring-the-dotnet-8-preview-changing-method-calls-with-interceptors)

이 글은 새 시리즈의 첫 번째 포스팅으로, .NET 8 프리뷰에 포함된 몇 가지 새로운 기능을 살펴봅니다. 이 게시물에서는 *Microsoft.Extensions.Configuration 구성 바인더*를 대상으로 도입된 새로운 소스 생성기를 살펴봅니다.

> 이 게시물은 모두 미리보기 빌드를 사용하므로 2023년 11월에 .NET 8이 최종 출시되기 전에 일부 기능이 변경(또는 제거)될 수 있습니다!

## 더 많은 소스 생성기가 필요한 이유는 무엇인가요?

소스 생성기는 컴파일 시 코드를 기반으로 추가 코드를 생성할 수 있는 기능으로 .NET 6에 도입되었습니다. 소스 생성기는 여러 가지 흥미로운 문제에 대한 해결책을 제공하기 때문에 [블로그에 여러 번 소스 생성기에 대한 글을 썼습니다](https://andrewlock.net/tag/source-generators/). 기본적으로 소스 생성기를 사용하면 수동으로 작성하기 어렵거나 번거로운 코드를 자동으로 생성할 수 있습니다.

제 [EnumExtensions 소스 생성기](https://andrewlock.net/netescapades-enumgenerators-a-source-generator-for-enum-performance/)를 예로 들어보겠습니다. 이 제너레이터는 리플렉션 기반 열거형 메서드에 대한 빠른 대안을 제공합니다. 예를 들어 다음과 같이 정의된 `enum이` 있다고 가정해 보겠습니다.

```csharp
[EnumExtensions]
public enum MyEnum
{
    First,
    Second,
}
```

소스 생성기는 다음과 같은 확장 메서드를 생성합니다.

```csharp
public static partial class MyEnumExtensions
{
    public static string ToStringFast(this MyEnum value)
        => value switch
        {
            MyEnum.First => nameof(MyEnum.First),
            MyEnum.Second => nameof(MyEnum.Second),
            _ => value.ToString(),
        };
}
```

정의된 값 중 하나를 사용하여 이 확장을 호출하면 "기본 제공" `ToString()` 메서드를 사용하는 것보다 훨씬 더 빠를 수 있습니다.

| **Method** | **FX** | **Mean** | **Error** | **StdDev** | **Ratio** | **Gen 0** | **Allocated** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ToString | `net48` | 578.276 ns | 3.3109 ns | 3.0970 ns | 1.000 | 0.0458 | 96 B |
| ToStringFast | `net48` | 3.091 ns | 0.0567 ns | 0.0443 ns | 0.005 | \- | \- |
| ToString | `net6.0` | 17.9850 ns | 0.1230 ns | 0.1151 ns | 1.000 | 0.0115 | 24 B |
| ToStringFast | `net6.0` | 0.1212 ns | 0.0225 ns | 0.0199 ns | 0.007 | \- | \- |

이 확장 기능은 손으로 할 수 없는 작업을 수행하지는 않지만, 중요한 점은 `ToStringFast()` 메서드를 자동으로 업데이트한다는 것입니다. `MyEnum`에 새 멤버를 추가하면 `ToStringFast()` 확장이 자동으로 업데이트되므로 사용자가 직접 업데이트하는 것을 기억할 필요가 없습니다!

소스 생성기의 또 다른 큰 장점은 리플렉션에 대한 앱의 런타임 의존성을 제거할 수 있다는 것입니다. 이는 성능상의 이점이 있을 수 있지만, 대부분의 경우 [리플렉션을 사용하면 일회성 비용으로 줄일 수 있습니다](https://andrewlock.net/benchmarking-4-reflection-methods-for-calling-a-constructor-in-dotnet/). 미리 컴파일(AOT)의 더 중요한 측면은 소스 생성을 사용하면 코드를 정적으로 분석할 수 있다는 것입니다.

AOT 컴파일의 핵심 부분은 트리밍(트리 쉐이킹이라고도 함)으로, 실제로 사용되지 않는 앱의 모든 부분을 최종 바이너리에서 제거합니다. 이는 AOT 앱을 작게 유지하는 데 중요합니다. 앱이 런타임 리플렉션을 사용하는 경우 컴파일러는 앱의 어떤 부분이 사용되거나 사용되지 않는지 쉽게 알 수 없으므로 AOT에 문제가 발생합니다. 리플렉션을 소스 생성 코드로 대체하면 앱이 더 AOT 친화적으로 바뀝니다.

[이것이 바로 구성 바인더 소스 생성기를 도입한 주된 이유입니다.](https://github.com/dotnet/runtime/issues/44493) AOT는 .NET 8의 ASP.NET Core 앱에 대한 우선 순위이며(현재는 최소한의 API 및 gRPC 앱만 지원될 예정임), 그 작업의 일부에는 AOT 친화적인 앱을 더 쉽게 만드는 것이 포함됩니다. 현재 .NET의 구성 바인딩 시스템은 리플렉션에 의존하고 있는데, 소스 생성기를 도입하면 이를 AOT 친화적인 생성 코드로 대체할 수 있습니다.

## 구성 바인딩은 어떻게 사용되나요?

ASP.NET Core는 ["옵션" 패턴](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-7.0)에 크게 의존합니다. 이 주제는 미묘한 차이와 주의해야 할 점이 많기 때문에 [제 블로그에서 광범위하게 다룬 적이 있는 또 다른 주제](https://andrewlock.net/tag/configuration/)입니다. 크게 보면 옵션 패턴에는 두 가지 개념이 포함됩니다:

* 다계층 구성. JSON 파일, XML 파일, Azure Key Vault, 환경 변수 등 여러 소스에서 구성 값을 로드할 수 있으며, 이러한 구성 값은 문자열 키-값 쌍의 사전으로 압축됩니다.
    
* C# 개체를 구성에 바인딩하기.
    

두 번째 요점은 구성 작업을 즐겁게 만드는 데 중요합니다. 다음과 같은 작업을 수행하는 대신 구성에서 값을 수동으로 구문 분석할 필요가 없습니다.

```csharp
public App(IConfiguration configuration)
{
    var rawValue = configuration.GetSection("AppFeatures")["RateLimit"];
    if (!string.IsNullOrEmpty(rawValue))
    {
        _rateLimit = int.Parse(rawValue);
    }
    else
    {
        _rateLimit = 100; // default
    }
}
```

앱 설정 코드에서 이와 같은 작업을 수행할 수 있습니다.

```csharp
var builder = WebApplication.CreateBuilder(args);

var configSection = builder.Configuration.GetSection("AppFeatures");
builder.Services.Configure<AppFeaturesSettings>(configSection);

public class AppFeaturesSettings
{
    public int? RateLimit { get; set; }
}
```

`AppFeaturesSettings` 클래스는 구성 섹션에 "바인딩"되어 구성에서 값 구문 분석을 자동으로 처리합니다. 그런 다음 `AppFeaturesSettings` 객체를 앱에 삽입할 수 있습니다.

```csharp
public App(IOptions<AppFeaturesSettings> settings)
{
    _rateLimit = settings.Value.RateLimit ?? 100;
}
```

> 네, 저도 앱의 `IOptions<>` 종속성이 마음에 들지 않지만, [제 책에서 설명한 것처럼](https://livebook.manning.com/book/asp-net-core-in-action-third-edition/chapter-10/v-10/241) 이를 해결할 수 있는 잘 알려진 방법이 있습니다!

`Configure<T>` 메서드는 궁극적으로 옵션 유형 `T`에서 바인딩 가능한 모든 속성을 찾고, 구성에서 값을 구문 분석하고, 설정하는 번거로운 작업을 처리하는 Microsoft.Extensions.Configuration의 확장 메서드 집합인 `ConfigurationBinder`를 호출하는 확장 메서드입니다.

## 현재 구성 바인딩은 내부적으로 어떻게 작동하나요?

짐작하셨겠지만, 현재 `ConfigurationBinder`는 이 프로세스를 처리하기 위해 리플렉션을 사용합니다. 어떤 `Configure<T>` 확장 메서드를 사용하든(또는 `Bind()`를 직접 호출하든), 결국에는 [`BindInstance()` 메서드](https://github.com/dotnet/runtime/blob/4c23ac2badc7d839a95ef485149e4fbb53da4695/src/libraries/Microsoft.Extensions.Configuration.Binder/src/ConfigurationBinder.cs#L276)로 호출하게 됩니다.

```csharp
private static void BindInstance(
    [DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.All)] Type type,
    BindingPoint bindingPoint,
    IConfiguration config,
    BinderOptions options)
{
    // ...
}
```

이 메서드에는 바인딩할 `Type`, 바인딩할`IConfiguration`, 그리고 몇 가지 추가 옵션(지금은 무시하겠습니다)을 받습니다. 이 메서드는 제공된 타입을 검사하고, 해당 타입이 문자열에서 직접 바인딩할 수 있는 "프리미티브" 타입이므로 직접 바인딩할 수 있는지 확인합니다.

그렇지 않고 `Type`이 이전의 `AppFeaturesSettings`와 같은 복잡한 객체인 경우, `BindInstance`는 유형에서 [리플렉션을 사용하여 바인딩 가능한 모든 속성을 찾은 다음](https://github.com/dotnet/runtime/blob/4c23ac2badc7d839a95ef485149e4fbb53da4695/src/libraries/Microsoft.Extensions.Configuration.Binder/src/ConfigurationBinder.cs#L216), `BindInstance`를 재귀적으로 호출하여 속성을 바인딩합니다.

이 동작을 사용하면 유형(및 중첩된 유형)의 모든 속성을 재귀적으로 바인딩하고 구성에서 값을 적절히 구문 분석하여 옵션 객체에 설정할 수 있습니다.

안타깝게도 앞서 지적했듯이 리플렉션을 사용하기 때문에 이 방법은 AOT 컴파일에 적합하지 않습니다. 이것이 바로 소스 생성기가 필요한 이유입니다.

## 구성 바인더 소스 생성기 설치 및 활성화하기

이 섹션에서는 애플리케이션에 구성 바인더 소스 생성기를 설치하고 활성화하는 방법을 보여드리겠습니다. 소스 생성기는 .NET 8 프리뷰 3에 도입되었지만 실제로는 .NET 7 앱에서 테스트했습니다.

.NET 6, .NET 7 또는 .NET 8(미리 보기) 앱에 구성 바인더 소스 생성기를 설치하려면 Microsoft.Extensions.Configuration.Binder 패키지를 설치하면 됩니다.

```plaintext
dotnet add package Microsoft.Extensions.Configuration.Binder --version 8.0.0-preview.3.23174.8
```

> 나중에 설명하는 것처럼 미리보기 4 및 5 패키지에서 문제를 발견했기 때문에 여기서는 미리보기 3 패키지를 설치합니다.

이 패키지에는 소스 생성기가 포함되어 있지만 기본적으로 비활성화되어 있습니다. 활성화하려면 프로젝트에서 MSBuild 프로퍼티를 설정해야 합니다.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net7.0</TargetFramework>
    <!-- 👇 Required, as you may get namespace issues without it currently-->
    <ImplicitUsings>enable</ImplicitUsings>
    <!-- 👇 Enable generator in Preview 3-->
    <EnableMicrosoftExtensionsConfigurationBinderSourceGenerator>true</EnableMicrosoftExtensionsConfigurationBinderSourceGenerator>
    <!-- 👇 Enable generator in Preview 4+-->
    <EnableConfigurationBindingGenerator>true</EnableConfigurationBindingGenerator>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Configuration.Binder" Version="8.0.0-preview.3.*" />
  </ItemGroup>
</Project>
```

미리 보기 3과 미리 보기 4 사이에서 설정할 MSBuild 속성이 변경되었습니다:

* 미리 보기 3에서는 `EnableMicrosoftExtensionsConfigurationBinderSourceGenerator`를 사용합니다.
    
* 미리보기 4 이상에서는 `EnableConfigurationBindingGenerator`를 사용합니다. 물론 최종 릴리스 전에 다시 변경될 수 있습니다!
    

프로젝트에 이 기능을 활성화하면 완료입니다! 프로젝트에서 코드를 변경할 필요가 없으며, `Configure<T>` 및 `Bind` 호출은 마술처럼 소스에서 생성된 코드를 사용합니다! IDE에서 `Configure<>` 호출에서 F12를 누르면 확인할 수 있습니다. 소스 생성기가 실행 중이면 생성된 코드로 바로 이동합니다! 소스 생성기가 실행 중이 아니라면 [`OptionsConfigurationServiceCollectionExtensions` 클래스](https://github.com/dotnet/runtime/blob/ddd748e16417bd1ddcf1094b0a63793a60408cfb/src/libraries/Microsoft.Extensions.Options.ConfigurationExtensions/src/OptionsConfigurationServiceCollectionExtensions.cs)에서 디컴파일된 소스 링크 코드를 보게 될 것입니다!

> 컴파일러가 생성된 코드를 디스크로 내보내도록 할 수도 있으므로 작업하기가 더 쉬워질 수 있습니다. [이전 블로그 게시물](https://andrewlock.net/creating-a-source-generator-part-6-saving-source-generator-output-in-source-control/)에서 이 방법을 사용하는 방법을 설명했습니다.

소스 제너레이터가 어떻게 콜사이트 코드가 원래 함수 대신 제너레이터를 강제로 호출하도록 하는 깔끔한 트릭을 구현했는지 궁금해서 생성된 코드를 살짝 들여다보았습니다.

## 생성된 코드 보기

더 이상 고민하지 않고 장난감 예제에 대해 생성된 코드의 예를 살펴보겠습니다. 다음 예제에서는 `Configure<>()`를 호출하여 `AppFeaturesSettings`를 바인딩하고 있습니다.

```csharp
var builder = WebApplication.CreateBuilder(args);

var configSection = builder.Configuration.GetSection("AppFeatures");
builder.Services.Configure<AppFeaturesSettings>(configSection); // 👈 Calls the source generator

public class AppFeaturesSettings
{
    public int? RateLimit { get; set; }
}
```

소스 생성기를 활성화하면 다음과 같은 코드가 생성됩니다(대략적으로 읽기 쉽도록 일반적인 `using` 문을 추출하여 정리했습니다).

```csharp
// <auto-generated/>
#nullable enable

using System;
using System.Linq;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

internal static class GeneratedConfigurationBinder
{
    public static IServiceCollection Configure<T>(this IServiceCollection services, IConfiguration configuration)
    {
        if (typeof(T) == typeof(AppFeaturesSettings))
        {
            return services.Configure<AppFeaturesSettings>(obj =>
            {
                BindCore(configuration, ref obj);
            });
        }

        throw new NotSupportedException($"Unable to bind to type '{typeof(T)}': 'Generator parser did not detect the type as input'");
    }

    private static void BindCore(IConfiguration configuration, ref AppFeaturesSettings obj)
    {
        if (obj is null)
        {
            throw new ArgumentNullException(nameof(obj));
        }

        if (configuration["RateLimit"] is string stringValue1)
        {
            obj.RateLimit = int.Parse(stringValue1);
        }

    }

    public static bool HasChildren(IConfiguration configuration)
    {
        foreach (IConfigurationSection section in configuration.GetChildren())
        {
            return true;
        }
        return false;
    }
}
```

간단한 예제에서는 생성된 코드가 비교적 간단합니다. `Configure<T>` 메서드는 제공된 유형이 소스가 생성된 유형인지 확인한 다음(항상 그래야 합니다) `BindCore()`를 호출하여 바인딩을 수행합니다. `BindCore()`는 앞서 설명한 "수동" 바인딩과 구문 분석만 수행하므로 실제로 마법이 있는 것은 아닙니다.

정말 깔끔한 비결은 코드를 변경하지 않고도 기존 코드가 소스 생성기를 호출하도록 만드는 것입니다! 제너레이터가 기존 호출을 어떻게 "가로채는" 걸까요?

```csharp
// Why does this 👇 suddenly call the source generator instead of the existing extension method
builder.Services.Configure<AppFeaturesSettings>(configSection);
```

메소드 서명을 주의 깊게 살펴보면 답을 찾을 수 있습니다. 라이브러리 메서드 서명은 다음과 같습니다.

```csharp
namespace Microsoft.Extensions.DependencyInjection;

public static class OptionsConfigurationServiceCollectionExtensions
{
    public static global::Microsoft.Extensions.DependencyInjection.IServiceCollection Configure<T>(
        this global::Microsoft.Extensions.DependencyInjection.IServiceCollection services, 
        global::Microsoft.Extensions.Configuration.IConfiguration configuration)
    where T : class
    {
        // ...
    }
}
```

소스에서 생성된 서명이 있는 동안

```csharp
internal static class GeneratedConfigurationBinder
{
    public static global::Microsoft.Extensions.DependencyInjection.IServiceCollection Configure<T>(
        this global::Microsoft.Extensions.DependencyInjection.IServiceCollection services, 
        global::Microsoft.Extensions.Configuration.IConfiguration configuration)
    {
        // ...
    }
}
```

실제 차이점은 단 3가지뿐입니다.

* 확장 메서드를 포함하는 클래스가 다릅니다.
    
* 라이브러리 메서드에는 추가 `class` 제네릭 제약 조건이 있습니다.
    
* 라이브러리 클래스는 `Microsoft.Extensions.DependencyInjection` 네임스페이스에 정의되는 반면, 생성된 클래스는 `global` 네임스페이스에 정의됩니다.
    

제가 알기로는 소스 생성 코드의 "재정의" 동작의 핵심은 생성된 코드가 전역 네임스페이스에 배치된다는 것입니다. 메서드 조회 우선 순위는 `global` 네임스페이스를 선호하므로 소스에서 생성된 확장 메서드가 `OptionsConfigurationServiceCollectionExtensions`의 메서드 대신 선택됩니다!

참고로 소스 생성기를 설명하는 [원래 github 이슈](https://github.com/dotnet/runtime/issues/44493)에 다음과 같이 언급되어 있습니다.

> 다음 단계로 Roslyn에서 개발 중인 사이트 교체 기능이라고 불리우는 것을 사용하여 사용자 호출을 생성된 호출로 직접 교체하고자 합니다.

조금 더 알아본 결과, 이는 [여기에서 논의된 인터셉터 제안](https://github.com/dotnet/csharplang/issues/7009)을 가리키는 것 같습니다. 프로토타입은 있지만 아직 구현되지는 않았기 때문에 어떻게 될지 계속 지켜보고 있습니다!

마지막 질문은 소스 생성기를 사용할 준비가 되었나요? 작동하지 않는 것이 있나요?

## 현재 작동하지 않는 항목은 무엇인가요?

.NET 8 프리뷰 3에 구현된 소스 생성기는 "초안"에 불과합니다. [이 이슈](https://github.com/dotnet/runtime/issues/79527)에 설명된 대로 몇 가지 미흡한 점이 있으며, 이는 다음 몇 번의 프리뷰에서 보완될 예정입니다. 이러한 이슈의 대부분은 리플렉션 구현과 최대한 동등하게 만드는 것에 관한 것입니다.

> 한 가지 문제가 발생할 수 있는 점은 [TypeConverter가 어떻게 처리될 것인가](https://github.com/dotnet/runtime/issues/83599) 하는 점입니다. 구성 바인딩에 `TypeConverter`를 사용하는 경우 이 문제에 대해 알아두는 것이 좋습니다.

다양한 옵션 유형으로 간단한 테스트를 해본 결과 바인딩 결과가 다른 경우는 단 한 가지뿐이었습니다.

```csharp
public class BindableOptions
{
    public IEnumerable<SubClass> IEnumerable { get; set; }
}
```

리플렉션 기반 바인더는 `IEnumerable` 프로퍼티를 기꺼이 바인딩하지만, 소스 생성기는 프리뷰 3에서 이를 건너뜁니다. 이 문제는 사실 [이번 PR](https://github.com/dotnet/runtime/pull/86285)에서 이미 수정되었지만, 테스트하려면 적어도 프리뷰 6까지 기다려야 합니다!

더 큰 문제는 프리뷰 4와 프리뷰 5 버전 모두 [컴파일되지 않는 코드를 생성](https://github.com/dotnet/runtime/issues/86348)한다는 것입니다 😱 다음은 프리뷰 4 출력을 보여주며, 프리뷰 5는 약간 다른 방식으로 깨집니다 😅.

```csharp
// <auto-generated/>
#nullable enable

internal static class GeneratedConfigurationBinder
{
    public static global::Microsoft.Extensions.DependencyInjection.IServiceCollection Configure<T>(this global::Microsoft.Extensions.DependencyInjection.IServiceCollection services, global::Microsoft.Extensions.Configuration.IConfiguration configuration)
    {
        if (configuration is null)
        {
            throw new global::System.ArgumentNullException(nameof(configuration));
        }

        if (typeof(T) == typeof(global::AppFeaturesSettings))
        {
            return services.Configure<global::AppFeaturesSettings>(obj =>
            {
                if (!global::Microsoft.Extensions.Configuration.Binder.SourceGeneration.Helpers.HasValueOrChildren(configuration))
                {
                    // 👇  Error CS8030 : Anonymous function converted to a void returning delegate cannot return a value
                    return default;
                }

                global::Microsoft.Extensions.Configuration.Binder.SourceGeneration.Helpers.BindCore(configuration, ref obj);
            });
        }

        throw new global::System.NotSupportedException($"Unable to bind to type '{typeof(T)}': 'Generator parser did not detect the type as input'");
    }
}
// ... additional generated code not shown
```

좋은 소식은 이제 [이 문제가 해결](https://github.com/dotnet/runtime/issues/86348)되었으므로 미리보기 6이 완벽하게 작동하길 바라지만, 그 동안에는 미리보기 3이 훌륭하게 작동한다는 것입니다 😆.

## 요약

이 글에서는 .NET 프리뷰 3에 도입된 새로운 구성 바인딩 소스 생성기를 살펴봤습니다. 기본 구성 바인더에서 사용하는 리플렉션을 대체하려면 .NET 8의 ASP.NET Core를 대상으로 하는 AOT 컴파일이 필요합니다. 구성 바인딩 소스 생성기는 메서드 확인 규칙을 사용하여 `Configure<>` 및 `Bind()` 호출의 호출 위치를 소스 생성된 버전으로 재정의합니다.