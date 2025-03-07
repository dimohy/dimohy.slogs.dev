---
title: "소스 생성기 만들기 7부 - 소스 생성기 '마커 특성' 문제 해결 - 1부"
datePublished: Thu Mar 17 2022 12:42:59 GMT+0000 (Coordinated Universal Time)
cuid: cl0uzgc530077k7nv5nf91eo6
slug: creating-a-source-generator-part-7-solving-the-source-generator-marker-attribute-problem-part1
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1647518756506/nGqtXA8on.png
tags: dotnet

---

> **이 글은 Andrew Lock님의 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/) 연재를 번역한 글입니다.**

이것은 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/)의 일곱 번째 게시물 입니다.

이 게시물에서 나는 소스 생성기와 관련하여 씨름해 온 문제에 대해 설명합니다. 소스 생성기를 구동하는 '마커 특성'을 어디에 둘 것인가 입니다. 이 포스트에서 나는 마커 특성이 무엇인지, 왜 그것이 소스 생성기에 유용한지, 그리고 그것들을 어디에 둘지 결정하는 것이 왜 문제가 될 수 있는지 설명합니다. 마지막으로, 다음 포스트에서 제가 결정한 솔루션에 대해 설명합니다. 이 솔루션은 아마도 가장 좋은 방법으로 보입니다.


## 마커 특성 및 소스 생성기 

저는 C# 소스 생성기의 팬이며 애플리케이션에서 소스 생성기를 사용하는 방법에 대해 [여러 게시물](https://andrewlock.net/tag/source-generators/)을 작성했습니다. 나는 최근에 [StronglyTypedId](https://www.nuget.org/packages/StronglyTypedId/1.0.0-beta02)라고 하는 강력한 형식의 ID를 생성하기 위한 [라이브러리를 업데이트](https://andrewlock.net/rebuilding-stongly-typed-id-as-a-source-generator-1-0-0-beta-release/)하여 사용자 지정 Roslyn 작업이 아닌 .NET의 기본 제공 소스 생성기 지원을 사용했습니다. 

대부분의 소스 생성기의 핵심 단계 중 하나는 코드 생성에 참여해야 하는 애플리케이션의 구문을 식별하는 것입니다. 이는 전적으로 소스 생성기의 목적에 따라 다르지만 매우 일반적인 접근 방식은 특성을 사용하여 코드 생성 프로세스에 참여해야 하는 코드를 장식하는 것입니다. 

예를 들어 .NET 6에서 Microsoft.Extensions.Logging 라이브러리의 일부인 [`LoggerMessage` 소스 생성기](https://andrewlock.net/exploring-dotnet-6-part-8-improving-logging-performance-with-source-generators/#the-net-6-loggermessage-source-generator)는 `[LoggerMessage]` 특성을 사용하여 생성될 코드를 정의합니다:

```csharp
using Microsoft.Extensions.Logging;

public partial class TestController
{
    // 여기에 특성을 추가하면 LogHelloWorld를 나타냅니다.
    // 메서드는 코드를 생성해야 합니다.
    [LoggerMessage(0, LogLevel.Information, "Writing hello world response to {Person}")]
    partial void LogHelloWorld(Person person);
}
```

마찬가지로 StronglyTypedId 패키지에서 `struct`에 적용된 `[StronglyTypedId]` 특성을 사용하여 유형이 `StronglyTypedId`가 되기를 원한다는 것을 나타냅니다:

```csharp
using StronglyTypedIds;

[StronglyTypedId]
public partial struct MyCustomId { }
```

이 두 경우 모두 특성 자체는 소스 생성기에 무엇을 생성할지 알려주기 위해 컴파일 타임에 사용되는 마커일 뿐입니다. 최종 컴파일된 출력에 있을 필요는 없지만 일반적으로 컴파일 되어도 문제가 되지 않습니다. 

이 게시물에서 다루고 있는 질문은 다음과 같습니다. 이러한 마커 특성은 어디에 정의되어야 합니까? 


## 마커 특성 정의

어떤 경우에는 사소한 대답이 있습니다. 생성기가 사용자가 필요로 하는 일부 기능이 있는 기존 라이브러리의 개선 사항인 경우 생성기를 해당 라이브러리와 함께 간단히 패키지할 수 있습니다.

예를 들어, `LoggerMessage` 생성기는 Microsoft.Extensions.Logging.Abstractions 라이브러리의 일부입니다. 사람들이 어쨌든 설치할 동일한 NuGet 패키지에 패키지되어 있으며 마커 특성은 참조된 dll에 포함되어 있으므로 항상 존재합니다. 이것은 마커 특성에 관한 한 "가장 좋은" 시나리오입니다. 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1647518756506/nGqtXA8on.png)

그러나 소스 생성기일 뿐인 라이브러리가 있다면 어떨까요? 여전히 해당 특성을 참조해야 하므로 표면적으로는 3가지 주요 옵션이 있습니다. 

- 소스 생성기를 사용하여 특성을 컴파일에 자동으로 추가합니다.
- 사용자에게 특성 자체를 컴파일에 추가하도록 요청합니다.
- 외부 dll에 특성을 포함하고 프로젝트가 이를 참조하는지 확인하십시오. 

이들 각각은 장단점이 있으므로 이 포스트에서는 각각의 장단점을 살펴보고 어떤 것이 가장 좋다고 생각하는지 이야기해 보겠습니다. 


## 1. 사용자 컴파일에 특성 추가

소스 생성자는 소비 프로젝트에 소스 코드를 추가할 수 있습니다. 일반적으로 소스 생성기는 컴파일에 추가한 코드에 액세스할 수 없으므로 전체 재귀 문제를 피할 수 있습니다. 한 가지 예외가 있습니다. 소스 생성기는 일부 고정 소스를 컴파일에 추가할 수 있는 "초기화 후" 후크를 등록할 수 있습니다. 

.NET 6의 증분 생성기 API의 경우 이 후크를 `RegisterPostInitializationOutput()`이라고 합니다. 이 시점에서 사용자 코드에 대한 액세스 권한이 없으므로 고정 코드를 추가하는 데만 유용하지만 사용자가 이를 참조할 수 있으며 소스 생성기에서 이를 참조하는 코드를 사용할 수 있습니다. 예를 들어 

```csharp
[Generator]
public class HelloWorldGenerator : IIncrementalGenerator
{
    /// <inheritdoc />
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // 특성 소스 등록
        context.RegisterPostInitializationOutput(i =>
        {
            var attributeSource = @"
            namespace HelloWorld
            {
                public class MyExampleAttribute: System.Attribute {} 
            }";
            i.AddSource("MyExampleAttribute.g.cs", attributeSource);
        });

        // ... 생성기 구현
    }
}
```

이 후크는 나중에 생성기에서 사용할 수 있는 사용자의 컴파일에 마커 특성을 추가하기 위해 맞춤 제작된 것 같습니다. 사실, 이 시나리오는 [소스 생성기 쿡북에서 마커 특성을 사용하는 "방법"으로 명시적으로 언급](https://github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md#augment-user-code)되어 있습니다. 

그리고 대부분의 경우 이것은 완벽하게 작동합니다. 

문제가 발생하는 지점은 사용자가 둘 이상의 프로젝트에서 소스 생성기를 참조하는 경우입니다. `MyExampleAttribute` 클래스는 `HelloWorld` 네임스페이스의 두 프로젝트에 추가됩니다. 프로젝트 중 하나가 다른 프로젝트를 참조하는 경우 `CS0436` 경고와 다음 행을 따라 빌드 경고가 표시됩니다. 

```
warning CS0436: The type 'MyExampleAttribute' in 'HelloWorldGenerator\MyExampleAttribute.g.cs' conflicts with the imported type 'MyExampleAttribute' in 'MyProject, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null'.
```

문제는 두 개의 다른 프로젝트에서 동일한 유형을 정의했으며 컴파일러가 둘을 구별할 수 없다는 것입니다. 어떻게 해결할 수 있습니까? 

명백한 해결책은 특성을 `public`이 아닌 `internal`로 만드는 것입니다. 그렇게 하면 각 프로젝트는 해당 특정 프로젝트에 추가된 `MyExampleAttribute`만 참조합니다. 그리고 그것은 효과가 있을 것입니다 🎉

그러나 누군가 `[InternalsVisibleTo]`를 사용하는 경우에는 작동하지 않습니다. 그 시점에서 사실상 모든 `internal`유형은 `public`이 되므로  다시 처음으로 돌아갑니다. 

이제 "사람들이 `[InternalsVisibleTo]`를 실제로 사용하지 않습니까?"라고 생각할 수도 있습니다. 글쎄, 나는 원래 `StronglyTypedId`에서 이 접근 방식을 취했으며 [예, 예, 그렇습니다](https://github.com/andrewlock/StronglyTypedId/issues/38#issuecomment-924674974). 하지만 저는 판단할 사람이 아닙니다. 제 하루 작업에 대한 [`AssemblyInfo.cs` 파일에는 22개의 `[InternalsVisibleTo]` 특성이 포함](https://github.com/DataDog/dd-trace-dotnet/blob/master/tracer/src/Datadog.Trace/AssemblyInfo.cs)되어 있습니다! 

큰 문제는 여기에 사용자를 위한 해결 방법이 없다는 것입니다. 그들은 그 시나리오에서 깨졌습니다. 다른 옵션을 살펴보겠습니다. 


## 2. 사용자에게 직접 생성하도록 요청 

다음 옵션은 사용자에게 특성을 직접 추가하도록 요청하는 것입니다. 이것이 어떻게 또는 왜 도움이 되는지 궁금할 수 있지만 핵심은 사용자가 한 번만 추가하면 전체 솔루션에서 동일한 특성을 사용할 수 있다는 것입니다. 모든 프로젝트에 소스 생성기를 추가하는 대신 사용자는 "도메인 도우미" 클래스(예를 들어)에 `MyExampleAttribute`를 만듭니다. 

이 접근 방식은 실제로는 겉으로 보이는 것처럼 이상하거나 후진적이지 않습니다. 사실, 정확히 이 접근 방식을 사용하는 많은 C# 기능이 있습니다. [최근 게시물](https://andrewlock.net/exploring-dotnet-6-part-11-callerargumentexpression-and-throw-helpers/#argumentnullexception-throw-)에서 `[DoesNotReturn]` 특성을 사용하여 언급할 때 그러한 경우를 언급했습니다. 이 특성은 무엇보다도 nullable 흐름 분석에 사용되지만 .NET Core 3용 BCL에만 정의되어 있습니다. 즉, .NET Core 2.x 또는 .NET Standard 권한을 대상으로 하는 경우 사용할 수 없습니다. 그런가요?

아닙니다! C# 컴파일러는 "직접 추가" 접근 방식을 사용합니다. 특성이 어딘가에 정의되어 있는 한 특성이 어디에 정의되어 있는지는 중요하지 않습니다. 즉, 자신의 프로젝트에 추가할 수 있으며(올바른 네임스페이스를 사용해야 함) C# 컴파일러는 "마법처럼" 이를 "원본"과 동일하게 취급합니다. 

```csharp
#if !NETCOREAPP3_0_OR_GREATER
namespace System.Diagnostics.CodeAnalysis
{
    [AttributeUsage(AttributeTargets.Method)]
    public class DoesNotReturnAttribute: Attribute { }
}
#endif
```

소스 생성기에 대해 정확히 동일한 접근 방식을 취할 수 있습니다. 그러나 사용자에게 이 작업을 수행하도록 요청하는 것은 약간 힘든 작업처럼 느껴집니다. 또한 `[DoesNotReturn]`과 같은 기본 특성은 괜찮지만 `[StronglyTypedId]`와 같은 복잡한 특성은 어떻습니까? 

```csharp
using System;

namespace StronglyTypedIds
{
    [AttributeUsage(AttributeTargets.Struct, Inherited = false, AllowMultiple = false)]
    [System.Diagnostics.Conditional("STRONGLY_TYPED_ID_USAGES")]
    public sealed class StronglyTypedIdAttribute : Attribute
    {
        public StronglyTypedIdAttribute(
            StronglyTypedIdBackingType backingType = StronglyTypedIdBackingType.Default,
            StronglyTypedIdConverter converters = StronglyTypedIdConverter.Default,
            StronglyTypedIdImplementations implementations = StronglyTypedIdImplementations.Default)
        {
            BackingType = backingType;
            Converters = converters;
            Implementations = implementations;
        }

        public StronglyTypedIdBackingType BackingType { get; }
        public StronglyTypedIdConverter Converters { get; }
        public StronglyTypedIdImplementations Implementations { get; }
    }
}
```

사용자에게 추가하도록 요청하고 모든 것을 정확하게 수정하여 생성기가 손상되지 않도록 하는 것은 나에게 초보자처럼 보입니다. 게다가 사용자가 프로젝트를 업데이트할 때마다 이 코드를 업데이트해야 하므로 API를 발전시킬 수 있는 능력을 잃게 됩니다. 지원 요청을 위한 레시피인 것 같습니다... 

그래서 우리에게 남은 선택지는 하나뿐입니다. 


## 3. 외부 dll에서 마커 특성 참조 

이 접근 방식을 사용하면 생성기가 마커 특성 자체를 추가하지 않으며 사용자도 이를 컴파일에 추가하지 않습니다. 대신 소스 생성기는 사용자의 프로젝트에서 참조하는 dll에 정의된 특성에 의존합니다. 

많은 옵션이 있기 때문에 해당 dll이 어떻게 또는 어디서 왔는지에 대해 의도적으로 주의를 기울이고 있습니다. 예를 들어 `[LoggerMessage]` 생성기는 생성기를 포함하는 Microsoft.Extensions.Logging.Abstractions NuGet 패키지에 있는 특성에 의존합니다. 이것은 생성자가 특성을 항상 사용할 수 있고 그 반대도 마찬가지임을 확인할 수 있기 때문에 특히 편리합니다. 특성이 사용 가능한 경우 생성기도 사용 가능합니다. 

생성기가 "기본" dll에 대한 "추가 옵션"인 경우 이 접근 방식이 완벽합니다. 일부 프로젝트의 분석기에 대해 수행되는 방식과 유사하게 "메인" 패키지가 종속성을 취하는 별도의 패키지에 생성기를 포함하는 것과 유사한 주장을 할 수 있습니다. 소스 생성기는 정말 멋진 분석기와 같으므로 동일한 패턴이 많이 적용되어야 합니다. 예를 들어, 기본 xunit 패키지는 xunit.analyzers 패키지에 종속됩니다. 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1647520086523/5uHpaFzAH.png)

이 접근 방식은 생성기가 기본 패키지에 "추가된 항목"인 경우 의미가 있습니다. 이러한 방식으로 종속성 체인을 유지하면 마커 특성이 있는 경우(예: xunit 패키지) 생성기가 항상 참조됩니다. 

> 기본 xunit 패키지 없이 생성기 패키지(예: xunit.analyzers)를 설치할 수 있지만 마커 특성을 사용하려고 하면 컴파일 오류가 발생하는 동작이 예상됩니다. 

그러나 원래 문제로 돌아가서 "독립 실행형" 생성기가 있다면 그것은 소스 생성기일 뿐입니다. 이 문제를 해결하기 위해 특성만 포함하는 NuGet 패키지를 도입할 필요가 없습니다. 

또 다른 가능성은 소스 생성기 dll 자체에 특성을 포함하는 것입니다. 기본적으로 소스 생성기를 포함하는 dll은 사용자의 컴파일에 포함되지 않지만 포함되기 할 수 있습니다. 일할 만큼 미친거야?

[StronglyTypedId 생성기 프로젝트](https://github.com/andrewlock/StronglyTypedId)에서 문제를 해결하기 위해 여러 가지 접근 방식을 시도했습니다. 그리고 바로 해결 방법으로 넘어가기 보다는 다음 포스트에서 제가 시도한 몇 가지 접근 방식, 어떻게 실패했는지, 궁극적으로 제가 결정한 솔루션에 대해 이야기하면서 저와 함께 여러분을 고통스럽게 만들 것입니다. 


## 요약

이 게시물에서 나는 소스 생성기의 맥락에서 "마커 특성"이 무엇인지, 그리고 어떻게 그것들이 코드 생성을 주도하는 데 도움이 될 수 있는지 설명했습니다. 그런 다음 특성을 컴파일에 추가하는 방법에 대한 질문에 대해 논의했습니다. 

일반적으로 소스 생성기 자체를 사용하여 컴파일에 추가하지만 사용자가 `[InternalsVisibleTo]` 특성을 사용할 때 문제가 발생할 수 있습니다. 해결 방법으로 C# 컴파일러가 경우에 따라 수행하는 것처럼 사용자에게 특성 자체를 추가하도록 요청할 수 있습니다. 또는 dll에 특성을 추가하고 해당 dll을 어떻게든 참조할 수 있습니다. 이를 달성하는 방법에 대한 다양한 옵션이 있습니다. 다음 게시물에서 나는 이들 중 일부를 탐색하고 내가 결정한 솔루션을 설명할 것입니다. 


## 원문
Andrew Lock's [Series: Creating a source generator](https://andrewlock.net/series/creating-a-source-generator/) - [Part 7 - Solving the source generator 'marker attribute' problem - Part 1](https://andrewlock.net/creating-a-source-generator-part-7-solving-the-source-generator-marker-attribute-problem-part1/)