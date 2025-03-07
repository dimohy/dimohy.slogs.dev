---
title: "소스 생성기 만들기 1부 - 증분 생성기 만들기"
datePublished: Tue Feb 22 2022 07:40:07 GMT+0000 (Coordinated Universal Time)
cuid: ckzxti98n02ok03nv8ovye5oq
slug: creating-a-source-generator-part-1-creating-an-incremental-source-generator
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1646028210562/9HMrme7GS.webp
tags: dotnet

---

> **이 글은 Andrew Lock님의 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/) 연재를 번역한 글입니다.**

이 게시물에서는 증분 소스 생성기를 만드는 방법을 설명합니다. 사례 연구로 `ToStringFast()`라는 열거형에 대한 확장 메서드를 생성하기 위한 소스 생성기를 설명합니다. 이 방법은 내장 `ToString()`에 상응하는 것보다 훨씬 빠르며 소스 생성기를 사용하는 것은 사용하기 쉽다는 것을 의미합니다! 

> 이것은 내가 최근에 만든 NetEscapades.EnumGenerators라는 소스 생성기를 기반으로 합니다. [GitHub](https://github.com/andrewlock/NetEscapades.EnumGenerators/) 또는 [NuGet](https://www.nuget.org/packages/NetEscapades.EnumGenerators/)에서 찾을 수 있습니다. 

소스 생성기에 대한 약간의 배경 지식을 제공하고 열거형에서 `ToString()`을 호출하는 문제부터 시작하겠습니다. 나머지 게시물에서는 증분 생성기를 만드는 과정을 단계별로 설명합니다. 최종 결과는 동작하는 소스 생성기이지만 게시물 끝에서 제한 사항을 설명합니다. 

1. 소스 생성기 프로젝트 만들기
1. 열거형에 대한 세부정보 수집
1. 마커 특성 추가
1. 증분 소스 생성기 만들기
1. 증분 생성기 파이프라인 구축
1. 파이프라인 단계 구현
1. EnumToGenerate를 생성하기 위해 EnumDeclarationSyntax 구문 분석
1. 소스 코드 생성
1. 제한 사항 

## 배경: 소스 생성기

소스 생성기는 .NET 5의 기본 제공 기능으로 추가되었습니다. 소스 생성기는 컴파일 타임에 코드 생성을 수행하여 프로젝트에 소스 코드를 자동으로 추가할 수 있는 기능을 제공합니다. 이것은 [광대한 가능성의 영역](https://github.com/amis92/csharp-source-generators)을 열어주지만, 소스 생성기를 사용하여 리플렉션을 사용하여 수행해야 하는 작업을 대체하는 기능은 가장 선호되는 기능입니다. 

나는 이미 소스 생성기에 대한 많은 게시물을 작성했습니다. 예를 들면 다음과 같습니다.
- [소스 생성기를 사용하여 Blazor WebAssembly 앱에서 라우팅 가능한 모든 구성 요소 찾기](https://andrewlock.net/using-source-generators-to-find-all-routable-components-in-a-webassembly-app/)
- [소스 생성기로 로깅 성능 향상](https://andrewlock.net/exploring-dotnet-6-part-8-improving-logging-performance-with-source-generators/)
- [소스 생성기 업데이트: 증분 생성기 ](https://andrewlock.net/exploring-dotnet-6-part-9-source-generator-updates-incremental-generators/)

소스 생성기를 완전히 처음 사용하는 경우 .NET Conf에서 제공한 [Jason Bock의 소스 생성기 소개](https://www.youtube.com/watch?v=4DVV7FXukC8&list=PLdo4fOcmZ0oVFtp9MDEBNbA2sSqYvXSXO&index=77&t=71s)를 추천합니다. 그것은 단지 30분이며(그는 더 긴 버전의 연설도 가지고 있습니다) 당신을 빠르게 시작하고 실행할 수 있습니다.

.NET 6에서는 "증분 생성기"를 만들기 위한 새로운 API가 도입되었습니다. 이들은 .NET 5의 소스 생성기와 기능이 대체로 동일하지만 캐싱을 활용하여 성능을 크게 향상시켜 IDE 속도가 느려지지 않도록 설계되었습니다! 증분 생성기의 주요 단점은 .NET 6 SDK에서만 지원된다는 것입니다(VS 2022에서만 지원됨).

## 범위: 열거형 및 ToString()

C#의 간단한 열거형은 옵션 선택을 나타내는 편리한 작은 아이디어입니다. 내부적으로는 숫자 값(일반적으로 `int`)으로 표시되지만 코드에서 `0`이 "빨간색"을 나타내고 `1`이 "파란색"을 나타냄을 기억하는 대신 해당 정보를 보유하는 열거형을 사용할 수 있습니다. 

```csharp
public enum Colour // 예, 저는 영국인 입니다
{
    Red = 0,
    Blue = 1,
}
```

코드에서 열거형 `Colour`의 인스턴스를 전달하지만 무대 뒤에서 런타임은 실제로 `int`를 사용합니다. 문제는 때때로 색상의 이름을 알고 싶어한다는 것입니다. 이를 수행하는 기본 제공 방법은 `ToString()`을 호출하는 것입니다. 

```csharp
public void PrintColour(Colour colour)
{
    Console.Writeline("You chose "+ colour.ToString()); // 빨간색 선택
}
```

아마 이 글을 보시는 분들은 다 아시는 내용일 것입니다. 그러나 이것이 느리다는 것은 덜 일반적인 지식일 수 있습니다. 곧 얼마나 느린지 살펴보겠지만 먼저 최신 C#을 사용하여 빠른 구현을 살펴보겠습니다. 

```csharp
public static class EnumExtensions
{
    public string ToStringFast(this Colour colour)
        => colour switch
        {
            Colour.Red => nameof(Colour.Red),
            Colour.Blue => nameof(Colour.Blue),
            _ => colour.ToString(),
        }
    }
}
```

이 간단한 switch 문은 `Colour`의 알려진 값 각각을 확인하고 `nameof`를 사용하여 `enum`의 텍스트 표현을 반환합니다. 알 수 없는 값이면 기본 값이 문자열로 반환됩니다. 

> 이러한 알 수 없는 값에 대해 항상 주의해야 합니다. 예를 들어 이것은 유효한 C# 입니다. `PrintColour((Colour)123)`

알려진 색상에 대한 이 간단한 switch 문과 기본 `ToString()` 구현을 [BenchmarkDotNet](https://benchmarkdotnet.org/)로 비교했을 때 우리의 구현이 얼마나 빠른지를 알 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1645505904623/NpMi8ohXq.png)

먼저 .NET 6의 `ToString()`이 .NET Framework의 메서드보다 30배 이상 빠르며 바이트의 1/4만 할당한다는 점을 지적할 가치가 있습니다! 하지만 "빠른" 버전과 비교하면 여전히 매우 느립니다!

아무리 빨라도 `ToStringFast()` 메서드를 만드는 것은 약간의 고통입니다. 열거형이 변경될 때 최신 상태로 유지해야 하기 때문입니다. 운 좋게도 이것은 소스 생성기의 완벽한 사용 사례입니다! 

> 커뮤니티에서 몇 가지 열거형 생성기인 [이것](https://github.com/Spinnernicholas/EnumFastToStringDotNet)과 [이것](https://github.com/meziantou/Meziantou.Framework/blob/main/src/Meziantou.Framework.FastEnumToStringGenerator/EnumToStringSourceGenerator.cs)을 알고 있지만 둘 다 내가 원하는 대로 되지 않았기 때문에 [직접 만들었습니다!](https://github.com/andrewlock/NetEscapades.EnumGenerators) 

이 게시물에서는 .NET 6 SDK에서 지원되는 새로운 증분 소스 생성기를 사용하여 `ToStringFast()` 메서드를 생성하는 소스 생성기를 만드는 과정을 살펴보겠습니다. 

## 1. 소스 생성기 프로젝트 생성

시작하려면 C# 프로젝트를 만들어야 합니다. 소스 생성기는 `netstandard2.0`을 대상으로 해야 하며 소스 생성기 유형에 액세스하려면 몇 가지 표준 패키지를 추가해야 합니다.

클래스 라이브러리를 생성하여 시작합니다. 다음은 sdk를 사용하여 현재 폴더에 솔루션 및 프로젝트를 생성합니다. 

```shell
dotnet new sln -n NetEscapades.EnumGenerators
dotnet new classlib -o ./src/NetEscapades.EnumGenerators
dotnet sln add ./src/NetEscapades.EnumGenerators
```

NetEscapades.EnumGenerators.csproj의 내용을 다음으로 교체합니다. 주석에서 각 속성이 수행하는 작업을 설명했습니다. 

```xaml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <!-- 👇 소스 생성기는 netstandard 2.0을 대상으로 해야 합니다. -->
    <TargetFramework>netstandard2.0</TargetFramework> 
    <!-- 👇 소비하는 프로젝트에서 소스 생성기 dll을 직접 참조하고 싶지 않습니다. -->
    <IncludeBuildOutput>false</IncludeBuildOutput> 
    <!-- 👇 새로운 프로젝트, 왜 안돼! -->
    <Nullable>enable</Nullable>
    <ImplicitUsings>true</ImplicitUsings>
    <LangVersion>Latest</LangVersion>
  </PropertyGroup>

  <!-- 다음 라이브러리에는 필요한 소스 생성기 인터페이스 및 유형이 포함되어 있습니다. -->
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.2" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.1" PrivateAssets="all" />
  </ItemGroup>

  <!-- `dotnet pack`을 사용할 때 라이브러리가 소스 생성기로 패키징 되도록 합니다. -->
  <ItemGroup>
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" 
        PackagePath="analyzers/dotnet/cs" Visible="false" />
  </ItemGroup>
</Project>
```

이것은 현재로서는 거의 모든 상용구이므로 코드를 시작하겠습니다. 

## 2. 열거형에 대한 세부 정보 수집 

생성기 자체를 빌드하기 전에 생성하려는 확장 방법을 고려해 보겠습니다. 최소한 다음 사항을 알아야 합니다. 

- `enum`의 전체 `Type` 이름
- 모든 값의 이름

이정도로 충분합니다. 더 나은 사용자 경험을 위해 수집할 수 있는 더 많은 정보가 있지만 지금은 이 정보를 사용하여 작동하도록 하겠습니다. 이를 감안할 때 발견한 열거형에 대한 세부 정보를 보관할 간단한 유형을 만들 수 있습니다. 

```csharp
public readonly struct EnumToGenerate
{
    public readonly string Name;
    public readonly List<string> Values;

    public EnumToGenerate(string name, List<string> values)
    {
        Name = name;
        Values = values;
    }
}
```

## 3. 마커 특성 추가

또한 확장 메서드를 생성할 열거형을 선택하는 방법에 대해서도 생각해야 합니다. 우리는 프로젝트의 모든 열거형에 대해 이를 수행할 수 있지만 이는 다소 불필요한 것 같습니다. 대신 "마커 특성"을 사용할 수 있습니다. 마커 특성은 기능이 없고 다른 것(이 경우 소스 생성기)이 유형을 찾을 수 있도록 존재하는 단순한 특성입니다. 사용자는 특성으로 열거형을 장식하므로 그것으로 확장 메서드를 생성하는 법을 알게 됩니다.

```csharp
[EnumExtensions] // 마커 특성
public enum Colour
{
    Red = 0,
    Blue = 1,
}
```

아래와 같이 간단한 마커 특성을 생성할 것이지만 코드에서 이 특성을 직접 정의하지는 않을 것입니다. 대신 `[EnumExtensions]` 마커 특성에 대한 C# 코드가 포함된 문자열을 생성합니다. 특성을 사용할 수 있도록 소스 생성기가 런타임 시 소비 프로젝트의 컴파일에 이것을 자동으로 추가하도록 할 것입니다. 

```csharp
public static class SourceGenerationHelper
{
    public const string Attribute = @"
namespace NetEscapades.EnumGenerators
{
    [System.AttributeUsage(System.AttributeTargets.Enum)]
    public class EnumExtensionsAttribute : System.Attribute
    {
    }
}";
}
```

나중에 이 `SourceGenerationHelper` 클래스에 더 많은 것을 추가할 것이지만 지금은 실제 생성기 자체를 생성할 시간입니다. 

## 4. 증분 소스 생성기 생성

증분 소스 생성기를 만들려면 3가지 작업을 수행해야 합니다.

1. 프로젝트에 Microsoft.CodeAnalysis.CSharp 패키지를 포함합니다. 증분 생성기는 버전 4.0.0에서 도입되었으며 .NET 6/VS 2022에서만 지원됩니다.
1. `IIIncrementalGenerator`를 구현하는 클래스 만들기
1. `[Generator]` 특성으로 클래스 꾸미기 

```csharp
namespace NetEscapades.EnumGenerators;

[Generator]
public class EnumGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // 마커 특성을 컴파일 과정에 추가합니다.
        context.RegisterPostInitializationOutput(ctx => ctx.AddSource(
            "EnumExtensionsAttribute.g.cs", 
            SourceText.From(SourceGenerationHelper.Attribute, Encoding.UTF8)));

        // TODO: 소스 생성기의 나머지 구현
    }
}
```

`IIIncrementalGenerator`는 `Initialize()`라는 단일 메서드만 구현하면 됩니다. 이 방법에서는 "정적" 소스 코드(예: 마커 특성)를 등록할 수 있을 뿐만 아니라 관심 있는 구문을 식별하고 해당 구문을 소스 코드로 변환하기 위한 파이프라인을 구축할 수 있습니다. 

위의 구현에서 마커 특성을 컴파일에 등록하는 코드를 이미 추가했습니다. 다음 섹션에서는 마커 특성으로 장식된 열거형을 식별하는 코드를 작성할 것입니다. 

## 5. 증분 생성기 파이프라인 구축

소스 생성기를 빌드할 때 기억해야 할 핵심 사항 중 하나는 소스 코드를 작성할 때 많은 변경 사항이 발생한다는 것입니다. 사용자가 변경할 때마다 소스 생성기가 다시 실행될 수 있으므로 효율적이어야 합니다. 그렇지 않으면 사용자의 IDE 경험이 중단될 수 있습니다.

> 이것은 단순한 일화가 아니라 `[LoggerMessage]` 생성기의 미리 보기 버전에서 정확히 [이 문제가 발생](https://github.com/dotnet/runtime/issues/56702)했습니다.

[증분 생성기의 설계](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md)는 변환 및 필터의 "파이프라인"을 생성하여 변경 사항이 없는 경우 작업을 다시 수행하지 않도록 각 레이어의 결과를 메모하는 것입니다. 파이프라인의 단계는 표면적으로 모든 소스 코드 변경에 대해 많이 호출될 것이기 때문에 매우 효율적이라는 것이 중요합니다. 나중 레이어는 효율적으로 유지되어야 하지만 더 많은 여지가 있습니다. 파이프라인을 잘 설계했다면 사용자가 중요한 코드를 편집할 때만 이후 레이어가 호출됩니다. 

> [최근 블로그 게시물](https://andrewlock.net/exploring-dotnet-6-part-9-source-generator-updates-incremental-generators/)에서 이 디자인에 대해 썼습니다. 

이를 염두에 두고(그리고 `[LoggerMessage]` 생성기에서 영감을 받아) 다음을 수행하는 간단한 생성기 파이프라인을 생성합니다.

- **하나 이상의 특성이 있는 열거형으로만 구문을 필터링합니다.** 이것은 매우 빨라야 하며 관심 있는 모든 열거형을 포함합니다.
- **`[EnumExtensions]` 특성이 있는 열거형으로만 구문을 필터링합니다.** 이것은 시맨틱 모델(단순한 구문이 아님)을 사용하기 때문에 첫 번째 단계보다 약간 더 비용이 많이 들지만 여전히 그다지 비싸지 않습니다.
- **`Compilation`을 사용하여 필요한 모든 정보를 추출합니다.** 이것은 가장 비용이 많이 드는 단계이며 프로젝트에 대한 `Compilation`을 이전에 선택한 `enum` 구문과 결합합니다. 여기에서 `EnumToGenerate` 컬렉션을 만들고 소스를 생성하고 소스 생성기 출력으로 등록할 수 있습니다. 

코드에서 파이프라인은 아래와 같습니다. 위의 세 단계는 각각 `IsSyntaxTargetForGeneration()`, `GetSemanticTargetForGeneration()` 및 `Execute()` 메서드에 해당하며 다음 섹션에서 보여줍니다. 

```csharp
namespace NetEscapades.EnumGenerators;

[Generator]
public class EnumGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // 마커 특성 추가
        context.RegisterPostInitializationOutput(ctx => ctx.AddSource(
            "EnumExtensionsAttribute.g.cs", 
            SourceText.From(SourceGenerationHelper.Attribute, Encoding.UTF8)));

        // 열거형에 대한 간단한 필터 수행
        IncrementalValuesProvider<EnumDeclarationSyntax> enumDeclarations = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: static (s, _) => IsSyntaxTargetForGeneration(s), // select enums with attributes
                transform: static (ctx, _) => GetSemanticTargetForGeneration(ctx)) // sect the enum with the [EnumExtensions] attribute
            .Where(static m => m is not null)!; // filter out attributed enums that we don't care about

        // 선택한 열거형을 `Compilation`과 결합
        IncrementalValueProvider<(Compilation, ImmutableArray<EnumDeclarationSyntax>)> compilationAndEnums
            = context.CompilationProvider.Combine(enumDeclarations.Collect());

        // Compilation 및 열거형을 사용하여 소스 생성
        context.RegisterSourceOutput(compilationAndEnums,
            static (spc, source) => Execute(source.Item1, source.Item2, spc));
    }
}
```

파이프라인의 첫 번째 단계는 `CreateSyntaxProvider()`를 사용하여 들어오는 구문 토큰 목록을 필터링합니다. 조건자 `IsSyntaxTargetForGeneration()`은 필터링의 첫 번째 계층을 제공합니다. 변환 `GetSemanticTargetForGeneration()`은 구문 토큰을 변환하는 데 사용할 수 있지만 이 경우에는 술어 뒤에 추가 필터링을 제공하는 데만 사용합니다. 후속 `Where()` 절은 LINQ처럼 보이지만 실제로는 두 번째 필터링 계층을 수행하는 `IncrementalValuesProvider`의 메서드입니다.

파이프라인의 다음 단계는 단순히 첫 번째 단계에서 내보낸 `EnumDeclarationSyntax` 컬렉션을 현재 `Compilation`과 결합합니다. 

마지막으로 `(Compilation, ImmutableArray<EnumDeclarationSyntax>)` 결합된 튜플을 사용하여 `Execute()` 메서드를 사용하여 `EnumExtensions` 클래스의 소스 코드를 실제로 생성합니다. 

이제 각각의 방법을 살펴보겠습니다.

## 6. 파이프라인 단계 구현 

파이프라인의 첫 번째 단계는 매우 빨라야 하므로 전달된 `SyntaxNode`에 대해서만 작동하고 최소한 하나의 특성이 있는 `EnumDeclarationSyntax` 노드만 선택하도록 필터링합니다. 

```chsarp
static bool IsSyntaxTargetForGeneration(SyntaxNode node)
    => node is EnumDeclarationSyntax m && m.AttributeLists.Count > 0;
```

보시다시피 이것은 매우 효율적인 술어입니다. 간단한 패턴 매치를 사용하여 노드의 유형을 확인하고 특성을 확인합니다. 

> C# 10에서는 `node is EnumDeclarationSyntax { AttributeLists.Count: > 0 }`이라고 작성할 수도 있지만 개인적으로 전자를 선호합니다. 

이 효율적인 필터링이 실행된 후에는 좀 더 중요해질 수 있습니다. 우리는 어떤 특성도 원하지 않고 특정 마커 특성만 원합니다. `GetSemanticTargetForGeneration()`에서 우리는 이전 테스트를 통과한 각 노드를 반복하고 마커 특성을 찾습니다. 노드에 특성이 있으면 추가 생성에 참여할 수 있도록 노드를 반환합니다. 열거형에 마커 특성이 없으면 `null`을 반환하고 다음 단계에서 필터링합니다. 

```csharp
private const string EnumExtensionsAttribute = "NetEscapades.EnumGenerators.EnumExtensionsAttribute";

static EnumDeclarationSyntax? GetSemanticTargetForGeneration(GeneratorSyntaxContext context)
{
    // IsSyntaxTargetForGeneration 덕분에 노드가 EnumDeclarationSyntax임을 압니다.    
    var enumDeclarationSyntax = (EnumDeclarationSyntax)context.Node;

    // 메서드의 모든 특성을 반복합니다.
    foreach (AttributeListSyntax attributeListSyntax in enumDeclarationSyntax.AttributeLists)
    {
        foreach (AttributeSyntax attributeSyntax in attributeListSyntax.Attributes)
        {
            if (context.SemanticModel.GetSymbolInfo(attributeSyntax).Symbol is not IMethodSymbol attributeSymbol)
            {
                // 이상합니다. 기호를 가져올 수 없습니다. 무시하십시오.
                continue;
            }

            INamedTypeSymbol attributeContainingTypeSymbol = attributeSymbol.ContainingType;
            string fullName = attributeContainingTypeSymbol.ToDisplayString();

            // 특성이 [EnumExtensions] 특성입니까?
            if (fullName == "NetEscapades.EnumGenerators.EnumExtensionsAttribute")
            {
                // return the enum
                return enumDeclarationSyntax;
            }
        }
    }

    // 찾고자 하는 특성을 찾지 못했습니다.
    return null;
}   
```

> 우리는 여전히 가능한 한 효율적으로 노력하고 있으므로 LINQ 대신 foreach 루프를 사용하고 있습니다. 

파이프라인의 이 단계를 실행하면 `[EnumExtensions]` 특성이 있는 `EnumDeclarationSyntax` 컬렉션이 생성됩니다. `Execute` 메서드에서 `EnumToGenerate`를 만들어 각 열거형에서 필요한 세부 정보를 보유하고 이를 `SourceGenerationHelper` 클래스에 전달하여 소스 코드를 생성하고 컴파일 출력에 추가합니다.

```csharp
static void Execute(Compilation compilation, ImmutableArray<EnumDeclarationSyntax> enums, SourceProductionContext context)
{
    if (enums.IsDefaultOrEmpty)
    {
        // 아직 할 일이 없음
        return;
    }

    // 이것이 실제로 필요한지 확실하지 않지만 `[LoggerMessage]`가 수행하므로 좋은 생각인 것 같습니다!
    IEnumerable<EnumDeclarationSyntax> distinctEnums = enums.Distinct();

    // 각 EnumDeclarationSyntax를 EnumToGenerate로 변환
    List<EnumToGenerate> enumsToGenerate = GetTypesToGenerate(compilation, distinctEnums, context.CancellationToken);

    // EnumDeclarationSyntax에 오류가 있는 경우 EnumToGenerate를 생성하지 않으므로 생성할 항목이 있는지 확인합니다. 
    if (enumsToGenerate.Count > 0)
    {
        // 소스 코드를 생성하고 출력에 추가
        string result = SourceGenerationHelper.GenerateExtensionClass(enumsToGenerate);
        context.AddSource("EnumExtensions.g.cs", SourceText.From(result, Encoding.UTF8));
    }
}
```

이제 가까워지고 있습니다. `GetTypesToGenerate()` 및 `SourceGenerationHelper.GenerateExtensionClass()`라는 두 가지 메서드가 더 있습니다. 

## 7. EnumToGenerate 생성을 위한 EnumDeclarationSyntax 구문 분석

`GetTypesToGenerate()` 메서드는 Roslyn 작업과 관련된 대부분의 일반적인 작업이 발생하는 곳입니다. 필요한 세부 정보를 얻기 위해 구문 트리와 의미론적 `Compilation`의 조합을 사용해야 합니다. 

- `enum`의 전체 유형 이름
- `enum`에 있는 모든 값의 이름 

다음 코드는 각 `EnumDeclarationSyntax`를 반복하고 해당 데이터를 수집합니다. 

```csharp
static List<EnumToGenerate> GetTypesToGenerate(Compilation compilation, IEnumerable<EnumDeclarationSyntax> enums, CancellationToken ct)
{
    // 출력을 저장할 목록을 만듭니다.
    var enumsToGenerate = new List<EnumToGenerate>();
    // 마커 특성의 의미론적 표현을 얻습니다.
    INamedTypeSymbol? enumAttribute = compilation.GetTypeByMetadataName("NetEscapades.EnumGenerators.EnumExtensionsAttribute");

    if (enumAttribute == null)
    {
        // 이것이 null이면 Compilation에서 마커 특성 유형을 찾을 수 없습니다.
        // 이는 무언가 매우 잘못되었음을 나타냅니다. 구제..
        return enumsToGenerate;
    }

    foreach (EnumDeclarationSyntax enumDeclarationSyntax in enums)
    {
        // 우리가 요청하면 중지
        ct.ThrowIfCancellationRequested();

        // 열거형 구문의 의미론적 표현 얻기 
        SemanticModel semanticModel = compilation.GetSemanticModel(enumDeclarationSyntax.SyntaxTree);
        if (semanticModel.GetDeclaredSymbol(enumDeclarationSyntax) is not INamedTypeSymbol enumSymbol)
        {
            // 뭔가 잘못되었습니다, 구제  
            continue;
        }

        // 열거형의 전체 유형 이름을 가져옵니다. e.g. Colour,
        // 또는 OuterClass<T>.Colour가 제네릭 형식에 중첩된 경우(예)
        string enumName = enumSymbol.ToString();

        // 열거형의 모든 멤버 가져오기 
        ImmutableArray<ISymbol> enumMembers = enumSymbol.GetMembers();
        var members = new List<string>(enumMembers.Length);

        // 열거형에서 모든 필드를 가져오고 해당 이름을 목록에 추가합니다. 
        foreach (ISymbol member in enumMembers)
        {
            if (member is IFieldSymbol field && field.ConstantValue is not null)
            {
                members.Add(member.Name);
            }
        }

        // 생성 단계에서 사용할 EnumToGenerate 생성 
        enumsToGenerate.Add(new EnumToGenerate(enumName, members));
    }

    return enumsToGenerate;
}
```
남은 것은 `List<EnumToGenerate>`에서 소스 코드를 실제로 생성하는 것뿐입니다! 

## 8. 소스 코드 생성

마지막 메서드 `SourceGenerationHelper.GenerateExtensionClass()`는 `EnumToGenerate` 목록을 가져오고 `EnumExtensions` 클래스를 생성하는 방법을 보여줍니다. 이것은 단지 문자열을 구축하기 때문에 개념적으로 비교적 간단합니다(시각화하기는 조금 어렵습니다!). 

```csharp
public static string GenerateExtensionClass(List<EnumToGenerate> enumsToGenerate)
{
    var sb = new StringBuilder();
    sb.Append(@"
namespace NetEscapades.EnumGenerators
{
    public static partial class EnumExtensions
    {");
    foreach(var enumToGenerate in enumsToGenerate)
    {
        sb.Append(@"
                public static string ToStringFast(this ").Append(enumToGenerate.Name).Append(@" value)
                    => value switch
                    {");
        foreach (var member in enumToGenerate.Values)
        {
            sb.Append(@"
                ").Append(enumToGenerate.Name).Append('.').Append(member)
                .Append(" => nameof(")
                .Append(enumToGenerate.Name).Append('.').Append(member).Append("),");
        }

        sb.Append(@"
                    _ => value.ToString(),
                };
");
    }

    sb.Append(@"
    }
}");

    return sb.ToString();
}
```

그리고 우리는 끝났습니다! 이제 완전히 작동하는 소스 생성기가 있습니다. 게시물 시작 부분에서 `Colour` 열거형을 포함하는 프로젝트에 소스 생성기를 추가하면 다음과 같은 확장 메서드가 생성됩니다. 

```csharp
public static class EnumExtensions
{
    public string ToStringFast(this Colour colour)
        => colour switch
        {
            Colour.Red => nameof(Colour.Red),
            Colour.Blue => nameof(Colour.Blue),
            _ => colour.ToString(),
        }
    }
}
```

## 제한 사항 

소스 생성기가 완료되면 `dotnet pack -c Release`를 실행하여 패키지화하고 NuGet에 업로드할 수 있습니다. 

잠깐만요, 실제로 그러지 마세요.

이 코드에는 많은 제한이 있습니다. 특히 아직 실제로 테스트하지 않았다는 사실도 그렇습니다. 머리에서 떠오르는 대로 : 

- `EnumExtensions` 클래스는 항상 동일한 것으로 호출되며 항상 동일한 네임스페이스에 있습니다. 사용자가 제어할 수 있으면 좋을 것입니다.
- `enum`의 가시성을 고려하지 않았습니다. `enum`이 `internal`인 경우 생성된 코드는 `public` 확장 메서드이므로 컴파일되지 않습니다.
- 코드 형식이 프로젝트 규칙과 일치하지 않을 수 있으므로 코드를 자동 생성된 것으로 표시하고 `#nullable enable`로 활성화해야 합니다.
- 우리는 그것을 테스트하지 않았으므로 실제로 작동하는지 모릅니다!
- 컴파일에 직접 마커 특성을 추가하는 것은 때때로 문제가 될 수 있습니다. 이에 대한 자세한 내용은 이후 게시물에서 설명합니다. 

즉, 이것이 여전히 유용하기를 바랍니다. 앞으로의 게시물에서 위의 많은 문제를 다룰 것이지만, 자신만의 증분 생성기를 만들려는 경우 이 게시물의 코드가 좋은 프레임워크를 제공해야 합니다. 

## 요약

이 게시물에서는 증분 생성기를 만드는 데 필요한 모든 단계를 설명했습니다. 프로젝트 파일을 만드는 방법, 컴파일에 마커 특성을 추가하는 방법, `IIncrementalGenerator`를 구현하는 방법, 생성기 소비자가 IDE에서 지연을 경험하지 않도록 성능을 염두에 두는 방법을 보여주었습니다. 결과 구현에는 많은 제한 사항이 있지만 기본 프로세스를 보여 줍니다. 이 시리즈의 향후 게시물에서 이러한 제한 사항 중 많은 부분을 다룰 것입니다. 

[GitHub에서 내 NetEscapades.EnumGenerators 프로젝트](https://github.com/andrewlock/NetEscapades.EnumGenerators)를 찾을 수 있으며, [내 블로그 샘플](https://github.com/andrewlock/blog-examples/tree/master/NetEscapades.EnumGenerators)에서 이 게시물에 사용된 기본 제거 버전의 소스 코드를 찾을 수 있습니다. 

## 원문

Andrew Lock's [Series: Creating a source generator](https://andrewlock.net/series/creating-a-source-generator/) - [Part 1 - Creating an incremental generator](https://andrewlock.net/creating-a-source-generator-part-1-creating-an-incremental-source-generator/)
