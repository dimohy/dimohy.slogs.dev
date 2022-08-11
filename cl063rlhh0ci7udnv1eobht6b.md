## 소스 생성기 만들기 2부 - 스냅샷 테스트로 증분 생성기 테스트

> **이 글은 Andrew Lock님의 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/) 연재를 번역한 글입니다.**


이것은 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/) 시리즈의 두 번째 게시물입니다.

이전 게시물에서 소스 생성기를 만드는 방법을 자세히 설명했지만 테스트라는 매우 중요한 단계를 놓쳤습니다. 이 게시물에서는 알려진 문자열에 대해 수동으로 소스 생성기를 실행하고 출력을 평가하여 소스 생성기를 테스트하는 방법 중 하나를 설명합니다. 스냅샷 테스트는 생성기가 계속 작동하는지 확인하는 좋은 방법을 제공하며, 이 게시물에서는 뛰어난 `Verify` 라이브러리를 사용합니다. 

## 요약: EnumExtensions 생성기

간단히 요약하자면, [이전 게시물](https://andrewlock.net/creating-a-source-generator-part-1-creating-an-incremental-source-generator/)에서 `enum`에서 `ToString()`을 호출하는 문제에 대해 논의했으며(느림), 소스 생성기를 사용하여 동일한 기능을 제공하는 100배 빠른 확장 메서드를 만드는 방법을 설명했습니다.

따라서 다음과 같은 간단한 `enum`의 경우: 

```csharp
public enum Colour
{
    Red = 0,
    Blue = 1,
}
```

다음과 같은 확장 메서드를 생성합니다. 

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

보시다시피 이 구현은 간단한 스위치 표현식으로 구성되며 `nameof` 키워드를 사용하므로 `Colour.Red.ToStringFast()`를 수행하면 예상대로 "Red"가 반환됩니다. 

나는 이 포스트에서 생성기의 구현에 대해 다루지 않을 것이고, 그것이 당신이 원하는 것이라면 [이전 포스트](https://andrewlock.net/creating-a-source-generator-part-1-creating-an-incremental-source-generator/)를 다시 참조하십시오. 

대신 이 게시물에서는 소스 생성기가 올바른 코드를 생성하는지 테스트하는 방법을 살펴보겠습니다. 내가 선호하는 접근 방식은 "스냅샷 테스트"를 사용하는 것입니다. 


## 소스 생성기에 대한 스냅샷 테스트 

이전에 스냅샷 테스트에 대해 쓴 적이 없으며 이 게시물은 여기에서 자세히 설명하지 않아도 충분히 길지만 개념은 매우 간단합니다: 하나 또는 두 개의 속성에 대해 주장하는 대신 스냅샷 테스트는 전체 개체(또는 다른 파일)가 예상 결과와 동일하다고 주장합니다. 그것보다 더 많은 것이 있지만 지금은 해야 합니다! 

> 운 좋게도 [Dan Clarke](https://twitter.com/dracan)는 최근 [.NET Advent Calendar](https://dotnet.christmas/2021/11)에 대한 기여로 [스냅샷 테스트에 대한 훌륭한 소개](https://www.danclarke.com/snapshot-testing-with-verify)를 작성했습니다! 

결과적으로 소스 생성기는 스냅샷 테스트에 매우 적합합니다. 소스 생성기는 주어진 입력(소스 코드)에 대한 결정론적 출력을 생성하는 것이며 우리는 항상 그 출력이 정확히 동일하기를 원합니다. 필요한 출력의 "스냅샷"을 만들고 실제 출력과 비교하여 소스 생성기가 올바르게 작동하는지 확인할 수 있습니다. 

이제 이 모든 작업을 수동으로 수행하는 코드를 작성할 수 있지만 [Simon Cropp](https://github.com/SimonCropphttps://github.com/SimonCropp)이 작성한 [Verify](https://github.com/VerifyTests/Verify)라는 훌륭한 라이브러리가 있으므로 필요하지 않습니다. 이 라이브러리는 직렬화 및 비교를 처리하고, 파일 이름 지정을 처리하고, diff-tools와 통합하여 테스트가 실패할 때 객체 간의 차이점을 시각화하여 쉽게 비교할 수 있도록 합니다. 

또한 Verify에는 메모리 내 개체, EF Core 쿼리, 이미지, Blazor 구성 요소, HTML, XAML, WinForms UI와 같은 거의 모든 스냅샷 테스트를 위한 확장이 있습니다. [목록은 끝이 없어 보입니다](https://github.com/orgs/VerifyTests/repositories)! 우리가 관심 있는 확장은 [Verify.SourceGenerators](https://github.com/VerifyTests/Verify.SourceGenerators)입니다. 

> 최근까지 Verify에 테스트 생성기 지원 기능이 내장되어 있다는 사실을 몰랐습니다. 이전에는 Verify를 "수동으로" 사용했지만 [처리되지 않은 예외 팟캐스트](https://unhandledexceptionpodcast.com/posts/0029-snapshottesting/)에서 [Simon](https://twitter.com/SimonCropp)이 Dan Clark과 이야기하는 것을 들었을 때 시도해야 했습니다! 

Verify.SourceGenerators에서 제공하는 확장 및 도우미는 "원래" 소스 생성기(`ISourceGenerator`) 및 증분 소스 생성기 `IIncrementalGenerator` 모두에서 작동하며 이전에 사용했던 "수동" 접근 방식에 비해 두 가지 주요 이점이 있습니다:

- 컴파일에 추가되는 여러 생성 파일을 자동으로 처리합니다.
- 컴파일에 추가된 모든 진단을 정상적으로 처리합니다. 

그런 이유로 나는 그의 라이브러리를 사용해야 하는 소스 생성기를 살펴보고 업데이트할 것입니다! 


## 1. 테스트 프로젝트 생성

솔루션에 `NetEscapades.EnumGenerators`라는 단일 프로젝트가 있는 지난 시간에 중단한 부분부터 계속하겠습니다. 이 프로젝트에는 소스 생성기가 포함되어 있습니다. 

다음 스크립트에서 다음을 수행합니다:

- xunit 테스트 프로젝트 생성
- 솔루션에 추가
- 테스트 프로젝트에서 src 프로젝트에 대한 참조 추가
- 테스트 프로젝트에 필요한 패키지를 추가합니다.
   - Microsoft.CodeAnalysis.CSharp 및 Microsoft.CodeAnalysis.Analyzers에는 메모리에서 소스 생성기를 실행하고 출력을 검사하기 위한 메서드가 포함되어 있습니다.
   - Verify.XUnit에는 xunit에 대한 스냅샷 테스트 통합 확인이 포함되어 있습니다. 다른 테스트 프레임워크에 대해 동등한 어댑터가 있습니다.
   - Verify.SourceGenerators에는 특히 소스 생성기로 작업하기 위해 확인하는 확장이 포함되어 있습니다. 이것은 필수는 아니지만 작업을 훨씬 쉽게 만듭니다! 

```shell
dotnet new xunit -o ./tests/NetEscapades.EnumGenerators.Tests
dotnet sln add ./tests/NetEscapades.EnumGenerators.Tests
dotnet add ./tests/NetEscapades.EnumGenerators.Tests reference ./src/NetEscapades.EnumGenerators
# Add some helper packages to the test project
dotnet add ./tests/NetEscapades.EnumGenerators.Tests package Microsoft.CodeAnalysis.CSharp
dotnet add ./tests/NetEscapades.EnumGenerators.Tests package Microsoft.CodeAnalysis.Analyzers
dotnet add ./tests/NetEscapades.EnumGenerators.Tests package Verify.SourceGenerators
dotnet add ./tests/NetEscapades.EnumGenerators.Tests package Verify.XUnit
```

위의 스크립트를 실행한 후 테스트 프로젝트의 .csproj 파일은 다음과 같아야 합니다. 

```xaml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
    <ImplicitUsings>true</ImplicitUsings>
  </PropertyGroup>

  <!-- 👇 기본 템플릿에 추가 -->
  <ItemGroup>
    <PackageReference Include="Verify.XUnit" Version="14.7.0" />
    <PackageReference Include="Verify.SourceGenerators" Version="1.2.0" />
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.2" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.1" PrivateAssets="all" />
  </ItemGroup>

  <!-- 👇 생성기 프로젝트에 참조 추가  -->
  <ItemGroup>
    <ProjectReference Include="..\..\src\NetEscapades.EnumGenerators\NetEscapades.EnumGenerators.csproj" />
  </ItemGroup>

  <!-- 👇 모든 기본 템플릿의 일부  -->
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="16.11.0" />
    <PackageReference Include="xunit" Version="2.4.1" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.4.3">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="coverlet.collector" Version="3.1.0">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>

</Project>
```

이제 모든 종속성이 설치되었으므로 테스트를 작성할 수 있습니다! 


## 2. 간단한 스냅샷 테스트 만들기 

소스 생성기를 테스트하려면 약간의 설정이 필요하므로 문자열에서 컴파일을 생성하고 소스 생성기를 실행한 다음 스냅샷 테스트를 사용하여 출력을 테스트하는 도우미 클래스를 만들 것입니다. 

그 전에 테스트가 어떻게 생겼는지 봅시다:

```csharp
using VerifyXunit;
using Xunit;

namespace NetEscapades.EnumGenerators.Tests;

[UsesVerify] // 👈 XUnit에 Verify를 위한 후크 추가
public class EnumGeneratorSnapshotTests
{
    [Fact]
    public Task GeneratesEnumExtensionsCorrectly()
    {
        // 테스트 할 소스코드
        var source = @"
using NetEscapades.EnumGenerators;

[EnumExtensions]
public enum Colour
{
    Red = 0,
    Blue = 1,
}";

        // 소스 코드를 도우미에 전달하고 스냅샷 테스트 출력
        return TestHelper.Verify(source);
    }
}
```

TestHelper는 여기에서 모든 작업을 수행하므로 좀 더 구체적으로 살펴보기 전에 다음 단계를 설명하기 위해 주석이 달린 초기 구현을 보여줍니다. 

```csharp
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using VerifyXunit;

namespace NetEscapades.EnumGenerators.Tests;

public static class TestHelper
{
    public static Task Verify(string source)
    {
        // 제공된 문자열을 C# 구문 트리로 구문 분석
        SyntaxTree syntaxTree = CSharpSyntaxTree.ParseText(source);

        // 구문 트리에 대한 Roslyn 컴파일 생성
        CSharpCompilation compilation = CSharpCompilation.Create(
            assemblyName: "Tests",
            syntaxTrees: new[] { syntaxTree });


        // EnumGenerator 증분 소스 생성기의 인스턴스 생성
        var generator = new EnumGenerator();

        // GeneratorDriver는 컴파일에 대해 생성기를 실행하는데 사용됨
        GeneratorDriver driver = CSharpGeneratorDriver.Create(generator);

        // 소스 생성기를 실행!
        driver = driver.RunGenerators(compilation);

        // 소스 생성기 출력을 스냅샷 테스트하려면 Verifier를 사용!
        return Verifier.Verify(driver);
    }
}
```

스냅샷 테스트를 실행하면 Verify가 `GeneratorDriver` 출력의 스냅샷을 기존 스냅샷과 비교하려고 시도합니다. 테스트를 처음 실행 했기때문에 테스트가 실패하므로 Verify는 자동으로 기본 diff 도구(제 경우에는 VS Code)를 엽니다. 그러나 diff는 아마도 당신이 기대하는 것을 보여주지 않을 것입니다! 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1646014903174/UlOUvzrnP.png)

기존 스냅샷이 없기 때문에 오른쪽 창은 비어 있습니다. 하지만 왼쪽에 소스 생성기 출력을 표시하는 대신 `{}`만 표시됩니다. 문제가 발생한 것 같습니다. 

좋아, 그것은 내가 문서를 읽지 않았기 때문에 밝혀졌습니다. [Verify.SourceGenerators](https://github.com/VerifyTests/Verify.SourceGenerators#initialize) 추가 정보에는 어셈블리에 대해 `VerifySourceGenerators.Enable();`를 한 번 호출하여 소스 생성기 출력을 처리하기 위해 변환기를 초기화해야 한다고 매우 명확하게 나와 있습니다. 

최신 C#에서 이를 수행하는 올바른 방법은 `[ModuleInitializer]` 특성을 사용하는 것입니다. 사양에 설명된 대로 이 코드는 어셈블리의 다른 코드보다 먼저 한 번 실행됩니다. 

`[ModuleInitializer]` 속성을 사용하여 프로젝트의 정적 void 메서드를 장식하여 모듈 이니셜라이저를 만들 수 있습니다. 우리의 경우 다음을 수행합니다. 

```csharp
using System.Runtime.CompilerServices;
using VerifyTests;

namespace NetEscapades.EnumGenerators.Tests;

public static class ModuleInitializer
{
    [ModuleInitializer]
    public static void Init()
    {
        VerifySourceGenerators.Enable();
    }
}
```

> 모듈 이니셜라이저는 C#9 기능이므로 이전 버전의 .NET을 대상으로 하는 경우에도 사용할 수 있습니다. 그러나 `[ModuleInitializer]` 특성은 .NET 5+에서만 사용할 수 있습니다. 이전 버전의 .NET을 대상으로 하는 경우 [이 게시물](https://andrewlock.net/exploring-dotnet-6-part-11-callerargumentexpression-and-throw-helpers/#argumentnullexception-throw-)에서 `[DoesNotReturn]` 특성에 대해 설명하는 접근 방식과 유사한 고유한 특성 구현을 만듭니다. 

이니셜라이저를 추가한 후 테스트를 다시 실행하면 조금 더 나은 결과를 얻을 수 있습니다. 소스 생성기의 일부로 컴파일에 추가한 사용자 지정 `[EnumExtensions]` 속성입니다. 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1646015200348/oZqol7s4h.png)

이 속성은 우리가 예상한 것과 같지만 여전히 잘못된 것이 있습니다. 다른 생성된 소스 코드가 없습니다. 소스 생성기가 속성을 추가했지만 `EnumExtensions` 클래스도 생성해야 합니다. 🤔 


## 3. 실패 디버깅: 참조 누락 

이와 같이 소스 생성기를 테스트할 때 좋은 점은 디버그하기가 매우 쉽다는 것입니다. 별도의 IDE 인스턴스를 시작할 필요가 없습니다. 말 그대로 단위 테스트의 컨텍스트에서 소스 생성기를 실행하고 있으므로 IDE의 테스트에서 "디버그"를 누르고(저는 JetBrains Rider를 사용하고 있습니다) 코드를 단계별로 실행할 수 있습니다! 

테스트에서 예외가 발생하지 않고 올바른 출력이 생성되지 않는다는 점을 감안할 때 소스 생성기의 어딘가에서 내 논리가 잘못되었음을 의심했습니다. [증분 생성기 파이프라인의 첫 번째 "변환" 메서드](https://andrewlock.net/creating-a-source-generator-part-1-creating-an-incremental-source-generator/#5-building-the-incremental-generator-pipeline) `GetSemanticTargetForGeneration()`에 중단점을 배치했습니다. 그런 다음 디버깅을 시작하고 중단점에 도달했는지 확인했습니다. 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1646015339889/puVMMCFrH.png)

위에서 볼 수 있듯이 `GetSemanticTargetForGeneration()`에서 중단점에 도달했고 `enumDeclarationSyntax` 변수에는 테스트 코드의 `Color` 열거형이 포함되어 있으므로 지금까지는 모든 것이 좋아 보입니다. `enum` 선언의 속성을 반복하는 메서드를 단계별로 살펴보고 `[EnumExtensions]` 속성을 찾으려고 했습니다. 그러나 이상하게도 `[EnumExtensions]` 구문의 `Symbol`에 액세스하기 위해 `SemanticModel`을 사용하려는 시도가 `null`을 반환하여 탈출했습니다! 이것은 소스 생성기가 어떻게 실패했는지 설명합니다. 다음 질문은, 왜일까요?

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1646015445251/g5-iicATx.png)

디버깅을 중단하기 전에 즉시 창을 사용하여 `context.SemanticModel.GetSymbolInfo(attributeSyntax).CandidateSymbols`의 값을 확인했습니다. 이것은 단일 값을 반환했으므로 실패는 모호성 또는 유사한 문제로 인한 것이 아닙니다. `context.SemanticModel.GetSymbolInfo(attributeSyntax).CandidateReason`을 확인하면 `NotAnAttributeType`이 반환되었습니다. 

어? `NotAnAttributeType`? 

약간의 삽질 후에 나는 문제가 기본적으로 컴파일에 참조가 없다는 것을 깨달았습니다. 즉, `System.Attribute`를 찾을 수 없으므로 `[EnumExtensions]` 특성을 올바르게 만들 수 없습니다. 해결책은 올바른 dll에 대한 참조를 추가하도록 `TestHelper`를 업데이트하는 것이었습니다. 개체를 포함하는 어셈블리(여기서는 `System.Private.CoreLib`)에 대한 참조를 만들고 이를 컴파일에 추가했습니다. 전체 `TestHelper` 클래스는 다음과 같습니다. 

```csharp
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using VerifyXunit;

namespace NetEscapades.EnumGenerators.Tests;

public static class TestHelper
{
    public static Task Verify(string source)
    {
        SyntaxTree syntaxTree = CSharpSyntaxTree.ParseText(source);
        // 필요한 어셈블리에 대한 참조 생성
        // 필요한 경우 여러 참조를 추가할 수 있습니다.
        IEnumerable<PortableExecutableReference> references = new[]
        {
            MetadataReference.CreateFromFile(typeof(object).Assembly.Location)
        };

        CSharpCompilation compilation = CSharpCompilation.Create(
            assemblyName: "Tests",
            syntaxTrees: new[] { syntaxTree },
            references: references); // 👈 컴파일에 대한 참조 전달

        EnumGenerator generator = new EnumGenerator();

        GeneratorDriver driver = CSharpGeneratorDriver.Create(generator);

        driver = driver.RunGenerators(compilation);

        return Verifier
            .Verify(driver)
            .UseDirectory("Snapshots");
    }
}
```

이 변경을 수행하고 테스트를 실행한 후 Verify가 diff-tool을 다시 엽니다. 이번에는 두 개의 diff를 포함합니다. 이전과 같은 `[EnumExtensions]` 속성과 생성된 `EnumExtensions` 클래스:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1646015777541/8fmrbKFHX.png)

이 시점에서 우리는 확인된 파일 diff를 수락할 수 있습니다. 그러면 디스크에 저장됩니다. 수동으로 한 쪽에서 다른 쪽으로 diff를 복사하거나 Verify가 터미널의 클립보드에 넣는 명령을 실행할 수 있습니다. 

```shell
cmd /c del "C:\repo\sourcegen\tests\NetEscapades.EnumGenerators.Tests\EnumGeneratorSnapshotTests.GeneratesEnumExtensionsCorrectly.00.verified.txt"
cmd /c del "C:\repo\sourcegen\tests\NetEscapades.EnumGenerators.Tests\EnumGeneratorSnapshotTests.GeneratesEnumExtensionsCorrectly.02.verified.cs"
cmd /c del "C:\repo\sourcegen\tests\NetEscapades.EnumGenerators.Tests\EnumGeneratorSnapshotTests.GeneratesEnumExtensionsCorrectly.verified.cs"
cmd /c del "C:\repo\sourcegen\tests\NetEscapades.EnumGenerators.Tests\EnumGeneratorSnapshotTests.GeneratesEnumExtensionsCorrectly.verified.txt"
cmd /c move /Y "C:\repo\sourcegen\tests\NetEscapades.EnumGenerators.Tests\EnumGeneratorSnapshotTests.GeneratesEnumExtensionsCorrectly.00.received.cs" "C:\repo\sourcegen\tests\NetEscapades.EnumGenerators.Tests\EnumGeneratorSnapshotTests.GeneratesEnumExtensionsCorrectly.00.verified.cs"
cmd /c move /Y "C:\repo\sourcegen\tests\NetEscapades.EnumGenerators.Tests\EnumGeneratorSnapshotTests.GeneratesEnumExtensionsCorrectly.01.received.cs" "C:\repo\sourcegen\tests\NetEscapades.EnumGenerators.Tests\EnumGeneratorSnapshotTests.GeneratesEnumExtensionsCorrectly.01.verified.cs"
```

이제 스냅샷을 업데이트했으므로 테스트를 다시 실행하면 스냅샷 테스트가 통과합니다! 🎉 


## 더 많은 테스트

이제 소스 생성기에 대한 단일 스냅샷 테스트를 작성했으므로 더 추가하는 것은 간단합니다. 나는 다음과 같은 경우를 테스트하기로 결정했습니다. 

- 속성이 없는 `enum` — 확장 메서드를 생성하지 않습니다.
- 올바른 네임스페이스 가져오기가 누락된 `enum` — 확장 메서드를 생성하지 않음
- 파일에 두 개의 `enum` — 두 `enum`에 대한 확장자를 생성합니다.
- 속성이 없는 두 개의 `enum` - 속성이 있는 `enum`에 대한 확장만 생성합니다. 

[GitHub에서 이러한 예제의 소스 코드](https://github.com/andrewlock/blog-examples/tree/master/NetEscapades.EnumGenerators2)를 찾을 수 있지만 기존 테스트와 거의 동일합니다. 변경되는 유일한 것은 테스트 소스 코드와 스냅샷입니다. 


## 진단 테스트

아직 살펴보지 않은 소스 생성기의 한 측면은 진단입니다. 소스 생성기는 분석기 역할도 하므로 사용자 소스 코드의 문제를 보고할 수 있습니다. 이것은 사용자에게 예를 들어 어떤 식으로든 생성기를 잘못 사용하고 있음을 알려야 하는 경우에 유용합니다. 

우리의 소스 생성기에는 진단 기능이 없지만 스냅샷 테스트와 잘 작동한다는 것을 보여주기 위해 더미 생성기를 추가합니다! 

먼저 소스 생성기에서 `enum`에 대한 진단을 생성하는 도우미 메서드를 만듭니다. 

```csharp
static Diagnostic CreateDiagnostic(EnumDeclarationSyntax syntax)
{
    var descriptor = new DiagnosticDescriptor(
        id: "TEST01",
        title: "A test diagnostic",
        messageFormat: "A description about the problem",
        category: "tests",
        defaultSeverity: DiagnosticSeverity.Warning,
        isEnabledByDefault: true);

    return Diagnostic.Create(descriptor, syntax.GetLocation());
}
```

다음으로 해당 메서드를 호출하여 소스 생성기의 `Execute()` 메서드에서 진단을 만들고 메서드에 제공된 `SourceProductionContext`를 사용하여 출력에 등록합니다. 

```csharp
static void Execute(Compilation compilation, ImmutableArray<EnumDeclarationSyntax> enums, SourceProductionContext context)
{
    if (enums.IsDefaultOrEmpty)
    {
        return;
    }

    // 더미 진단 추가
    context.ReportDiagnostic(CreateDiagnostic(enums[0]));

    // ...
}
```

> 이것은 단지 스냅샷 테스트를 보여주기 위한 것이며 무작위 진단이 나타나는 것을 원하지 않는다는 것을 기억하십시오! 

테스트를 다시 실행하면 이제 실패가 발생합니다. Verify는 컴파일 및 진단에 추가된 추가 소스 코드를 모두 추출합니다. 진단은 C# 개체이므로 다음과 같은 JSON 형식 문서로 직렬화됩니다. 

```json
{
  Diagnostics: [
    {
      Id: TEST01,
      Title: A test diagnostic,
      Severity: Warning,
      WarningLevel: 1,
      Location: : (3,0)-(8,1),
      MessageFormat: A description about the problem,
      Message: A description about the problem,
      Category: tests
    }
  ]
}
```

Verify는 diff 도구를 한 번 더 실행하고 이제 테스트, 진단을 위한 추가 파일이 있음을 보여줍니다. 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1646016410190/5xQgGauxV.png)

소스 생성기는 일반적으로 주어진 입력에 대해 원하는 매우 구체적이고 결정적인 출력이 있다는 점을 감안할 때 스냅샷 테스트를 위한 거의 완벽한 사용 사례처럼 보입니다. 분명히 더 세분화된 단위 테스트를 위해 소스 생성기를 설계할 수 있지만 대부분 필요한 경우 약간의 디버깅으로 스냅샷 테스트가 필요한 모든 것을 제공합니다! 


## 요약

이 게시물에서는 스냅샷 테스트를 사용하여 이전 게시물에서 만든 소스 생성기를 테스트하는 방법을 보여주었습니다. 스냅샷 테스트에 대해 간략하게 소개한 다음, Verify.SourceGenerators를 사용하여 생성기 출력을 테스트하는 방법을 보여주었습니다. 우리는 몇 가지 문제를 디버깅하고 마침내 Verify가 소스 생성기가 생성하는 진단 및 구문 트리를 모두 처리한다는 것을 시연했습니다. 


## 원문
Andrew Lock's [Series: Creating a source generator](https://andrewlock.net/series/creating-a-source-generator/) - [Part 2 - Testing an incremental generator with snapshot testing](https://andrewlock.net/creating-a-source-generator-part-2-testing-an-incremental-generator-with-snapshot-testing/)