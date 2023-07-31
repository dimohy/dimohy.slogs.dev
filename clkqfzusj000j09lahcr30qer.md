---
title: "최소 Api Aot 컴파일 템플릿"
datePublished: Mon Jul 31 2023 05:41:20 GMT+0000 (Coordinated Universal Time)
cuid: clkqfzusj000j09lahcr30qer
slug: the-minimal-api-aot-compilation-template
canonical: https://andrewlock.net/exploring-the-dotnet-8-preview-the-minimal-api-aot-template/
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1690782016501/f696afef-03b6-4d33-8e60-c869b74ee3f8.jpeg
tags: net, dotnet

---

> Andrew Lock님의 [The minimal API AOT compilation template](https://andrewlock.net/exploring-the-dotnet-8-preview-the-minimal-api-aot-template/)를 DeepL의 도움을 받아 번역하였습니다.

이 글은 [.NET 8 미리 보기 살펴보기](https://dimohy.hashnode.dev/series/exploring-dotnet8-preview) 시리즈의 두 번째 포스팅입니다.

[1부 - 새로운 구성 바인더 소스 생성기 사용](https://dimohy.hashnode.dev/7ioi66gc7jq0ioq1royessdrsjtsnbjrjzqg7iam7iqkioydneyeseq4scdsgqzsmqk)  
2부 - 최소한의 API AOT 컴파일 템플릿(이 게시물)  
3부 - WebApplication.CreateBuilder()와 새로운 CreateSlimBuilder() 메서드 비교하기  
4부 - 새로운 최소 API 소스 생성기 살펴보기 5부 - 메서드 호출을 인터셉터로 대체하기

.NET 8의 가장 큰 초점 중 하나는 Ahead of Time(AOT) 컴파일입니다. 이 글에서는 .NET 8 SDK 미리보기 릴리스에서 제공되는 새로운 "AOT 지원" 템플릿을 살펴보고, 몇 가지 흥미로운 기능을 짚어보고, AOT의 주요 이점 중 하나인 빠른 시작 시간을 시연해 보겠습니다.

> 이 게시물은 모두 프리뷰 빌드를 사용하고 있으므로 2023년 11월에 .NET 8이 최종 출시되기 전에 일부 기능이 변경(또는 제거)될 수 있습니다!

## AOT 컴파일이란 무엇인가요?

Ahead of Time(AOT) 컴파일은 Microsoft의 ASP.NET 팀에서 .NET 8을 위해 작업 중인 주요 기능 중 하나입니다. 이 기능이 무엇인지, 그리고 왜 이 기능이 중요한지 이해하기 위해 잠시 시간을 내어 .NET에서 일반적으로 어떻게 작동하는지, 그리고 AOT와 어떻게 다른지 살펴보겠습니다.

> 동영상을 선호하신다면 최근 데미안 에드워즈와 데이비드 파울러가 이 주제에 대해 이야기하는 [훌륭한 커뮤니티 스탠드업](https://www.youtube.com/watch?v=km9CnGYBafI)이 있었습니다.

전통적으로 .NET에서 사용하는 언어(C#, F#, VB.NET 등)가 무엇이든 컴파일러를 사용하여 중간 언어(IL) 바이트 코드를 생성합니다. 최신 .NET에서는 프로젝트에서 `dotnet build`를 실행하여 IL과 전체 메타데이터가 포함된 실행 파일을 생성하는 방식으로 이 단계를 수행합니다.

> IL이 어떻게 보이는지 궁금하다면 [https://sharplab.io](https://sharplab.io) 에서 실험해 볼 수 있습니다. 여기에서 [C# 스니펫에 대한 IL을 출력](https://sharplab.io/#v2:EYLgtghglgdgNAFxFANgHwAICYCMBYAKAwGYACbUgYVIG9DSHyyMAWUgWQAoBKW+xgRhwBOTgCIAEgFMUKAPakA7nIBOKACYBCMdwDc/BgF9ChoA)하도록 선택할 수 있으며, 이는 저수준 또는 성능 작업을 수행할 때 유용할 수 있습니다.

간단한 예로 다음과 같은 함수를 들 수 있습니다.

```csharp
public static void Main(string name)
    => Console.WriteLine("Hello " + name + "!");
```

이렇게 입력하면 컴파일러는 다음과 같은 IL을 생성합니다.

```csharp
IL_0000: ldstr "Hello "
IL_0005: ldarg.0
IL_0006: ldstr "!"
IL_000b: call string [System.Runtime]System.String::Concat(string, string, string)
IL_0010: call void [System.Console]System.Console::WriteLine(string)
IL_0015: ret
```

여기서 세부 사항은 중요하지 않으며, 중요한 점은 IL이 여전히 상대적으로 높은 수준이라는 것입니다. 이러한 명령어를 가져와서 CPU에서 직접 실행할 수는 없습니다. 이를 위해서는 IL을 어셈블리 코드로 변환하는 또 다른 컴파일 단계가 필요합니다. .NET에서는 일반적으로 런타임에 Just-in-time(JIT) 컴파일러를 사용하여 .NET 런타임에서 이 작업을 수행합니다. 결과 명령어는 다음과 같이 보일 수 있습니다.

```csharp
L0000: push ebp
L0001: mov ebp, esp
L0003: push edi
L0004: push esi
L0005: push ebx
L0006: mov esi, ecx
L0008: test esi, esi
L000a: je short L0059
L000c: mov edi, [esi+4]
L000f: test edi, edi
L0011: je short L0059
L0013: lea ecx, [edi+7]
L0016: call System.String.FastAllocateString(Int32)
L001b: mov ebx, eax
L001d: push dword ptr [0x8b086e8]
L0023: mov ecx, ebx
L0025: xor edx, edx
L0027: call dword ptr [0x64b15a0]
L002d: push esi
L002e: mov ecx, ebx
L0030: mov edx, 6
L0035: call dword ptr [0x64b15a0]
L003b: lea edx, [edi+6]
L003e: push dword ptr [0x8ad5530]
L0044: mov ecx, ebx
L0046: call dword ptr [0x64b15a0]
L004c: mov ecx, ebx
L004e: call dword ptr [0x10b271e0]
L0054: pop ebx
L0055: pop esi
L0056: pop edi
L0057: pop ebp
L0058: ret
L0059: mov ecx, 7
L005e: call System.String.FastAllocateString(Int32)
L0063: mov ebx, eax
L0065: cmp dword ptr [ebx+4], 6
L0069: jl short L0096
L006b: lea ecx, [ebx+8]
L006e: mov edx, [0x8b086e8]
L0074: add edx, 8
L0077: push 0xc
L0079: call dword ptr [0x6a99fc0]
L007f: push dword ptr [0x8ad5530]
L0085: mov ecx, ebx
L0087: mov edx, 6
L008c: call dword ptr [0x64b15a0]
L0092: mov ecx, ebx
L0094: jmp short L004e
L0096: mov ecx, 0x9c595b4
L009b: call 0x05f0300c
L00a0: mov esi, eax
L00a2: mov ecx, esi
L00a4: call dword ptr [0x9c61af8]
L00aa: mov ecx, esi
L00ac: call 0x62fcef50
L00b1: int3
```

이는 CPU가 실제로 실행하는 명령어입니다. AOT 컴파일을 사용하면 중간 IL 단계를 완전히 건너뛰고 최종 CPU에서 실행되는 어셈블리 코드를 직접 생성할 수 있습니다.

당연한 질문은 왜 AOT를 사용하고 싶지 않을까요? AOT와 JIT의 장단점은 무엇인가요?

## AOT 컴파일의 장단점

AOT 컴파일은 어셈블리 코드 명령어를 생성하므로 프로그램이 실행될 때 런타임에 JIT 컴파일이 필요하지 않습니다. 따라서 시작 시간이 크게 단축된다는 한 가지 큰 장점이 있습니다.

> [다른 잠재적인 이점](https://devblogs.microsoft.com/dotnet/asp-net-core-updates-in-dotnet-8-preview-3/#benefits-of-using-native-aot-with-asp-net-core) \*\*—**AOT는 전체 디스크 공간(모든 파일의 총 크기)과 메모리 사용량을 줄일 수 있습니다**—\*\*하지만 여기서는 시작 시간 단축에 초점을 맞추겠습니다.

일반적으로 .NET 런타임은 메서드를 실행하기 전에 메서드의 IL에 대해 JIT 컴파일러를 실행하여 실행할 어셈블리 코드를 생성합니다. 앱이 시작되면 일반적으로 JIT 컴파일해야 하는 클래스와 메서드가 많이 있습니다. 이 모든 것이 합쳐지면 일반적으로 .NET 앱을 시작하는 데 시간이 오래 걸릴 수 있습니다.

한 번만 시작하고 몇 시간 또는 며칠 동안 계속 실행되는 기존 서버 애플리케이션에서는 시작하는 데 시간이 오래 걸리는 것은 크게 문제가 되지 않습니다. 하지만 AWS Lambda 또는 Azure 함수를 사용하는 애플리케이션의 경우 시작 시간이 중요합니다. 이러한 앱은 요청에 대한 응답으로 스핀업되고 한 번 실행된 후 종료됩니다. 밀리초 단위로 요금을 청구할 수 있는 이러한 앱의 경우 시작 시간이 매우 중요합니다.

이 특정 시나리오는 .NET용 AOT 컴파일이 정말 빛을 발하는 곳입니다. 이것이 바로 .NET 8이 집중하는 이유입니다. 하지만 AOT가 JIT 컴파일을 사용하는 것보다 보편적으로 "더 나은" 것은 아닙니다. 단점도 많기 때문에 AOT가 올바른 선택이 아닌 경우가 많으며, 사용 사례에 따라서는 JIT 접근 방식이 더 나을 수도 있습니다.

> AOT에 적합한 .NET 프로그램을 만드는 것은 쉬운 일이 아니므로, [미니멀 API 앱과 gRPC 앱만 .NET 8에서 AOT와 호환될 것으로 예상됩니다](https://devblogs.microsoft.com/dotnet/asp-net-core-updates-in-dotnet-8-preview-3/#asp-net-core-and-native-aot-compatibility).

AOT의 주요 문제 중 하나는 머신 코드가 IL보다 훨씬 크다는 것입니다(이전 섹션에서 볼 수 있듯이): 15개의 IL 명령어와 인수는 177개의 어셈블리 코드 명령어를 생성합니다.) 따라서 순진하게도 모든 프레임워크를 포함하여 ASP.NET Core 앱에서 AOT를 수행하면 결과 파일 크기가 엄청나게 커집니다.

관리 가능한 크기를 만드는 유일한 방법은 "앱 트리밍"(일부 다른 언어에서는 트리 셰이킹이라고도 함)을 수행하는 것입니다. 여기에는 앱에서 실제로 사용되지 않는 애플리케이션과 프레임워크의 모든 부분을 제거하는 작업이 포함됩니다. 이렇게 하면 결과 바이너리의 크기를 크게 줄일 수 있으며 AOT를 실용적으로 만들 수 있습니다.

> AOT 없이 독립형 애플리케이션을 트리밍할 수 있지만 그 반대의 경우도 마찬가지이며, 모든 실용적인 목적을 위해 AOT는 트리밍이 필요합니다.

트리밍은 AOT의 전제 조건이지만 어려움이 시작되는 곳이기도 합니다. 트리밍이 올바르게 작동하려면 컴파일러가 애플리케이션에서 실제로 사용되는 클래스, 필드 및 메서드를 정확히 파악할 수 있어야 합니다. 그 외에는 모두 제거해야 합니다.

문제는 .NET에는 리플렉션과 동적 디스패치가 있기 때문에 일반적으로 모든 .NET 애플리케이션을 정적으로 완전히 분석하는 것은 근본적으로 불가능하다는 것입니다. 간단한 예로, 클래스 이름을 입력으로 받아 해당 유형의 인스턴스를 생성하는 콘솔 애플리케이션을 상상해 보세요. 사용자가 어떤 유형을 요청할지 미리 알 수 없으므로 최종 프로그램에서 해당 유형이 실제로 유지되는지 확인할 수 없습니다.

또 다른 예로는 dll을 동적으로 로드하는 '플러그인' 애플리케이션이 있습니다. 이러한 어셈블리에 어떤 유형이 필요한지 미리 알 수 있는 방법이 없습니다.

> 플러그인 스타일 애플리케이션은 실제로 더 큰 문제가 있습니다. AOT에는 JIT가 없으므로 로드하려는 dll에 포함된 IL을 컴파일할 방법조차 없습니다!

트리밍은 .NET 앱을 AOT와 호환되게 만드는 것이 어려운 근본적인 이유입니다. .NET 8에서 AOT를 지원하기 위해 많은 노력을 기울인 부분은 대부분 컴파일러에서 프레임워크 구성 요소를 정적으로 분석할 수 있도록 만드는 것입니다. ASP.NET Core의 설계를 고려할 때 이는 전혀 쉬운 일이 아닙니다!

또 다른 한계는 AOT가 Windows x64 또는 Linux arm64와 같은 특정 플랫폼용 어셈블리 코드를 생성한다는 점입니다. 생성된 코드는 일반적으로 설계상 크로스 플랫폼인 IL과 달리 지정된 플랫폼에서만 실행할 수 있습니다.

간혹 간과되는 한 가지 점은 JIT 컴파일러는 코드가 실행되는 머신에 대해 AOT 컴파일러보다 더 많은 정보를 알고 있다는 점입니다(해당 머신에서 실행 중이기 때문에!). 즉, JIT 컴파일러는 잠재적으로 AOT 컴파일러보다 더 최적화된 코드를 생성할 수 있습니다. 예를 들어, JIT 컴파일러는 사용 가능한 하드웨어 내재성을 알고 있는 반면, AOT 컴파일러는 동일한 가정을 할 수 없습니다. 즉, 일부 상황에서는 JIT 앱의 정상 상태 성능이 AOT 앱보다 우수할 수 있습니다.

지금까지 이론에 대해 설명했으니 이제 새로운 템플릿을 살펴보고 AOT의 영향을 살펴볼 차례입니다!

## 템플릿 살펴보기

.NET 미리 보기에는 새로운 미니멀 API 애플리케이션을 생성하는 `dotnet new api` [새 템플릿이 포함](https://learn.microsoft.com/en-gb/aspnet/core/fundamentals/native-aot?view=aspnetcore-8.0#the-api-template)되어 있습니다. 이 템플릿은 .NET 7 `webapi`와 `empty` 템플릿 사이의 어딘가에 있습니다. 이 템플릿은 자동 생성된 할 일 모델을 가져오기 위한 API 엔드포인트가 있는 기본 할 일 목록 애플리케이션입니다.

저희 목적상 정말 흥미로운 점은 `--aot` 옵션이 포함되어 있다는 점입니다. 다음을 사용하여 템플릿을 생성할 수 있습니다.

```plaintext
dotnet new api --aot
```

`--aot` 옵션은 생성된 코드에 두 가지를 추가합니다.

* .NET 6 JSON 소스 생성기를 구성하고 `JsonSerializerContext` 구현을 추가합니다.
    
* 이 옵션은 *.csproj* 파일에서 MSBuild 속성을 `PublishAot=true`로 설정합니다.
    

소개는 이것으로 충분하니 이제 코드를 살펴보겠습니다! Program.cs 파일은 아래와 같으며, 몇 가지 흥미로운 점을 강조 표시했습니다.

```csharp
using System.Text.Json.Serialization;

// 👇 Note Slim builder - new in .NET 8
var builder = WebApplication.CreateSlimBuilder(args); 

// 👇 Only added with --aot, configures the JSON source generator
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonSerializerContext.Default);
});

var app = builder.Build();

// 👇 Generates an array of `Todo` objects
var sampleTodos = TodoGenerator.GenerateTodos().ToArray();

// 👇 The actual API configuration
var todosApi = app.MapGroup("/todos");
todosApi.MapGet("/", () => sampleTodos);
todosApi.MapGet("/{id}", (int id) =>
    sampleTodos.FirstOrDefault(a => a.Id == id) is { } todo
        ? Results.Ok(todo)
        : Results.NotFound());

app.Run();

// 👇 The serialization context required for source generation
[JsonSerializable(typeof(Todo[]))]
internal partial class AppJsonSerializerContext : JsonSerializerContext
{

}
```

여기에는 몇 가지 흥미로운 점이 있습니다.

* 새로운 `CreateSlimBuilder()` 메서드를 사용하고 있다는 점입니다—자세한 내용은 이후 포스트에서 설명합니다!
    
* JSON 소스 생성기를 사용하도록 앱을 구성합니다(AOT에 필요)
    
* 다소 과도한 (제 생각에는) 생성기를 사용하여 `Todo` 개체 배열을 생성합니다(이 게시물에는 표시되지 않음)
    

> 템플릿을 올바르게 만드는 것이 항상 어려운 일이라는 것을 알고 있지만, 제 생각에는 정적 배열도 유용할 수 있는데 30줄의 제너레이터 코드(템플릿 전체 코드의 30% 이상)는 약간 지나친 것 같습니다 😉 특히 데모가 아닌 시나리오에서는 즉시 삭제될 코드이기 때문에 더욱 그렇습니다.

일반적인 appsettings.json 파일 등이 있습니다. *.csproj* 파일은 비교적 표준적인 파일이지만 몇 가지 속성을 구성합니다.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <!-- 👇 Disables server GC to reduce memory consumption -->
    <ServerGarbageCollection>false</ServerGarbageCollection>
    <!-- 👇 Using invariant globalization reduces app sizes -->
    <InvariantGlobalization>true</InvariantGlobalization>
    <!-- 👇 Enables always publishing as AOT -->
    <PublishAot>true</PublishAot>
  </PropertyGroup>
</Project>
```

이제 템플릿이 완성되었으니 직접 사용해 볼 시간입니다!

## 템플릿 테스트

AOT를 사용하여 앱을 게시하려면

```plaintext
dotnet publish
```

프로젝트 파일의 `PublishAot` 설정 덕분에 앱은 AOT 컴파일 체인을 사용하여 자동으로 게시됩니다.

> AOT의 경우 Visual Studio 2022(또는 [여기에 설명](https://learn.microsoft.com/en-gb/dotnet/core/deploying/native-aot/?tabs=net7#prerequisites)된 필수 구성 요소)의 "데스크톱 개발(C++를 사용한 데스크톱 개발)" 워크로드를 설치해야 한다는 점에 유의하세요. 그렇지 않으면 `오류 : 플랫폼 링커를 찾을 수 없습니다.` 또는 `치명적 오류 LNK1181: 입력 파일 'advapi32.lib'를 열 수 없습니다.` 의 오류가 표시될 수 있습니다. 후자의 경우, 현재 사용 중인 컴퓨터와 일치하는 구성 요소 `Windows 10 SDK(10.0.19041.0)`를 설치했습니다.

앱을 게시하고 나면 바로 실행할 수 있으며, 다른 ASP.NET Core 미니멀 API 앱과 마찬가지로 작동합니다! 하지만 이 글에서는 AOT를 사용했을 때와 사용하지 않았을 때의 시작 시간을 비교하는 것이 가장 흥미로웠습니다.

### 시작 시간 측정

시작 시간을 측정하기 위해 간단한 조정을 하기로 했습니다. `app.Run()`을 호출하여 실행하고 실행을 차단하는 대신 다음과 같이 변경했습니다.

```csharp
var task = app.RunAsync();

await app.StopAsync();
```

이렇게 하면 앱이 실행되기 시작하고 즉시 중지됩니다. 앱을 시작하고 중지하는 데 걸리는 총 시간을 측정하기 위해 [동료 중 한 명](https://twitter.com/tonywanhjor)이 작성한 `timeit` 작업의 포트인 [TimeItSharp](https://github.com/tonyredondo/timeitsharp) 도구를 사용했습니다. 그는 Datadog .NET 추적기 작업의 일환으로 애플리케이션의 지속 시간을 측정하기 위해 비슷한 목적으로 이 코드를 작성했습니다.

글로벌 .NET 도구를 설치하려면, 실행하세요.

```plaintext
dotnet tool install --global TimeItSharp
```

TimeItSharp는 [구성 파일을 사용](https://github.com/tonyredondo/timeitsharp#sample-configuration)하여 앱을 워밍업으로 실행할 횟수와 앱을 실행할 횟수를 구성합니다. 앱에 다음과 같은 간단한 JSON 파일인 timeit.json을 추가했습니다.

```json
{
  "warmUpCount": 10,
  "count": 100,
  "scenarios": [{"name": "Default"}],
  "processName": "aottest.exe",
  "workingDirectory": "$(CWD)/",
  "processTimeout": 15
}
```

그런 다음 앱을 게시했습니다.

```plaintext
> dotnet publish
MSBuild version 17.7.0-preview-23281-03+4ce2ff1f8 for .NET
  Determining projects to restore...
  All projects are up-to-date for restore.
  aottest -> C:\aottest\bin\Release\net8.0\win-x64\aottest.dll
  aottest -> C:\aottest\bin\Release\net8.0\win-x64\publish\
```

그리고 테스트를 실행했습니다!

```plaintext
cd C:\aottest\bin\Release\net8.0\win-x64\publish\
dotnet timeit timeit.json
```

이제 결과를 살펴봅시다!

### 결과 비교

TimeItSharp는 다양한 메트릭을 생성할 수 있지만, 간단하게 보여드리기 위해 축소된 버전의 결과를 보여드리겠습니다. 먼저 AOT를 제외한 결과를 살펴보겠습니다.

```plaintext
C:\aottest\bin\Release\net8.0\publish> dotnet timeit timeit.json
TimeIt (v. 0.0.8.0) by Tony Redondo

Warmup count: 10
Count: 100
Number of Scenarios: 1
Exporters: ConsoleExporter, JsonExporter, Datadog

Scenario: Default
  Warming up ..........
    Duration: 7.4238972s
  Run ....................................................................................................
    Duration: 37.0493903s
```

`WebApplication`을 빌드하고, 앱을 시작하고, 종료하는 등 AOT가 아닌 앱을 실행하는 데 걸리는 평균 시간은 **350밀리초**였습니다.

| **Name** | **Mean** | **StdDev** | **StdErr** | **Min** | **Max** | **P95** | **P90** | **Outliers** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Default | 363.9ms | 6.268ms | 0.6299ms | 350.5ms | 382.9ms | 375.9ms | 373.6ms | 1 |

> 이 수치는 모두 상대적인 수치이며, 비교적 오래된 Windows 노트북에서 실행한 것이므로 다른 컴퓨터에서는 다른 수치가 나올 수 있습니다!

AOT를 활성화한 상태에서 동일한 작업을 실행하면 전체 프로세스가 **45ms**밖에 걸리지 않습니다!

| **Name** | **Mean** | **StdDev** | **StdErr** | **Min** | **Max** | **P95** | **P90** | **Outliers** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Default | 44.75ms | 3.755ms | 0.3773ms | 39.98ms | 58.00ms | 52.58ms | 50.27ms | 1 |

동일한 앱을 실행하는 데 걸리는 시간이 7배 이상 빨라졌습니다! 이것이 바로 AOT의 힘입니다 😃.

이 글 전체에서 설명했듯이 AOT가 모든 문제를 해결해 주지는 않으며 새로운 문제를 야기할 수도 있지만 .NET 앱 시작 시간을 개선하는 것은 확실합니다!

## 요약

이 글에서는 .NET 앱에서 AOT 컴파일과 JIT 컴파일의 차이점을 설명하고, AOT 컴파일의 장단점을 살펴봤습니다. 그런 다음 .NET 8 미리 보기에 포함된 새로운 AOT 호환 `api` 템플릿을 살펴봤습니다. AOT를 사용하지 않은 앱과 AOT 버전의 앱 시작 시간을 비교한 결과, AOT를 사용하지 않은 경우 시작 및 종료에 걸리는 시간이 평균 350밀리초였지만 AOT를 사용하면 45밀리초로 줄어드는 등 AOT의 이점이 명확하게 드러났습니다! 앱 시작 시간 단축이라는 AOT의 진정한 장점이 바로 이 부분에서 빛을 발합니다.