---
title: ".NET 8의 웹 API 업데이트 | Christian Nagel"
datePublished: Thu Apr 20 2023 00:11:18 GMT+0000 (Coordinated Universal Time)
cuid: clgodaj75000509jrf1se2oxc
slug: net-8-api-christian-nagel
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1681947086226/89162c8b-c682-4886-9e7a-99ccff068a1d.jpeg
tags: dotnet

---

> Christian Nagel님의 [Web API Updates with .NET 8](https://csharp.christiannagel.com/2023/04/19/api-dotnet8/)를 번역하였습니다.

---

.NET 8의 미리보기 3에는 일기 예보 대신 TODO 서비스를 사용하여 API를 만드는 새로운 프로젝트 템플릿이 포함되어 있습니다. 이 템플릿의 생성된 코드를 살펴보면 슬림 빌더, AOT를 사용하여 네이티브 .NET 바이너리를 생성할 때 도움이 되는 JSON 소스 생성기 사용 등 더 많은 변경 사항이 있습니다. 이 글에서는 앞으로의 변경 사항을 살펴봅니다.

![Planets and light](https://csharpdotchristiannageldotcom.files.wordpress.com/2023/04/planetsandlight.jpg align="left")

## 일기 예보 대신 TODO 서비스

여전히 일기 예보 서비스를 생성하는 닷넷 새 웹 API 외에도 Todo 서비스를 생성하는 새 템플릿을 사용할 수 있습니다. 이 템플릿의 이름은 ASP.NET Core API이며(다른 하나는 ASP.NET Core Web API 템플릿입니다), `dotnet new api`로 사용할 수 있습니다.

이렇게 생성된 코드에는 몇 가지 흥미로운 부분이 있습니다. 간단한 부분부터 시작하겠습니다. 첫 번째 코드 스니펫은 `Todo` 클래스의 구현을 보여줍니다. .NET 6부터 이미 사용할 수 있는 `DateOnly` 유형을 사용하는 것 외에는 특별한 것이 없습니다.

```csharp
public class Todo
{
   public int Id { get; set; }
   public string? Title { get; set; }
   public DateOnly? DueBy { get; set; }
   public bool IsComplete { get; set; }
}
```

동일한 `Todo.cs` 파일에는 몇 가지 샘플 데이터를 생성하는 데 사용되는 `TodoGeneratorclass`가 있습니다. 이 `TodoGenerator`에는 세 개의 항목으로 구성된 배열인 `_parts`라는 필드가 포함되어 있습니다. 이 항목의 모든 항목은 `Prefixes`와 `Suffixes`라는 두 개의 배열로 구성된 튜플입니다. .NET에서는 점점 더 많은 **튜플**이 사용되며, 이는 튜플의 훌륭한 사용 사례입니다.

```csharp
private static readonly {string[] Prefixes, string[] Suffixes)[] _parts = new[]
{
   (new[] { "Walk the", "Feed the" }, new[] { "dog", "cat", "goat" }),
   (new[] { "Do the", "Put away the" }, new[] { "groceries", "dishes", "laundry" }),
   (new[] { "Clean the" }, new[] { "bathroom", "pool", "blinds", "cat" })
};
```

`GenerateTodos` 메서드는 염소 산책시키기(Walk the goat), 빨래하기(Do the laundry), 자동차 청소하기(Clean the car)와 같이 파트의 접두사와 접미사를 결합하여 임의의 조합을 생성합니다.

이를 위해 세 개의 값을 결합한 또 다른 튜플 배열인 `titleMap` 변수가 사용됩니다: `Row`, `Prefix`, `Suffix`. 3번을 반복 하면서 `titleMap` 배열은 행에 대해 가능한 모든 조합으로 채워집니다(세 개의 `_parts` 값이 있는 세 개의 행, 접두사(샘플 코드에는 행에 따라 하나 또는 두 개의 요소가 있음), 행의 접두사와 함께만 사용할 수 있는 접미사가 있습니다). 세 번의 반복이 완료되면 `titleMap` 배열에는 `(0, 0, 0)`, `(0, 0, 1)`, `(0, 0, 2)`, 최대 `(2, 0, 3)` 등의 값을 가진 16개의 항목이 포함됩니다.

다음으로 `Random.Shared.Shuffle`을 사용하면 새로운 .NET 8 API가 `titleMap` 배열을 섞는데 사용됩니다. 이렇게 하면 배열이 제자리에서 섞이고, 그 결과 `titleMap` 배열의 순서가 무작위로 바뀝니다. 그런 다음 `titleMap` 배열은 `yield` 문을 사용하여 이 배열의 첫 번째 항목으로 `Todo` 객체를 만드는 데 사용됩니다.

```csharp
internal static IEnumerable<Todo> GenerateTodos(int count = 5)
{
   var titleCount = _parts.Sum(row => row.Prefixes.Length * row.Suffixes.Length);
   var titleMap = new (int Row, int Prefix, int Suffix)[titleCount];
   var mapCount = 0;
   for (var i = 0; i < _parts.Length; i++)
   {
      var prefixes = _parts[i].Prefixes;
      var suffixes = _parts[i].Suffixes;
      for (var j = 0; j < prefixes.Length; j++)
      {
         for (var k = 0; k < suffixes.Length; k++)
         {
            titleMap[mapCount++] = (i, j, k);
         }
      }
   }

   Random.Shared.Shuffle(titleMap);

   for (var id = 1; id <= count; id++)
   {
      var (rowIndex, prefixIndex, suffixIndex) = titleMap[id];
      var (prefixes, suffixes) = _parts[rowIndex];
      yield return new Todo
      {
         Id = id,
         Title = string.Join(' ', prefixes[prefixIndex], suffixes[suffixIndex]),
         DueBy = Random.Shared.Next(-200, 365) switch
         {
            < 0 => null,
            var days => DateOnly.FromDateTime(DateTime.New.AddDays(days))
         }
      };
   }
}
```

## 슬림 빌더

이제 정말 흥미로운 부분으로 들어가 보겠습니다. `Program.cs` 파일에는 익숙한 `WebApplication`과 `WebApplicationBuilder`가 포함되어 있습니다. 하지만 `CreateBuilder`를 호출하는 대신 새로운 메서드인 `CreateSlimBuilder`가 사용됩니다. 이름에서 알 수 있듯이 이 메서드는 최소한의 기능만 포함하는 슬림한 빌더를 생성합니다. `CreateSlimBuilder` 메서드는 .NET 8에서 사용할 수 있습니다.

`CreateBuilder`와 `CreateSlimBuilder`의 차이점은 무엇인가요? .NET 8 미리보기 3에서 `CreateBuilder` 메서드는 종속성 주입 컨테이너에 93개의 서비스를 등록합니다. `CreateSlimBuilder`를 사용하면 65개의 서비스가 등록됩니다. 아마도 더 많은 변화가 있을 수 있지만 이것은 이야기의 일부일 뿐입니다. 이제 구성 및 로깅 부분에 대해 알아보겠습니다.

```csharp
var builder = WebApplication.CreateSlimBuilder(args);
builder.Logging.AddConsole();

var app = builder.Build();
```

### 구성 공급자

슬림 빌더는 메모리(`MemoryConfigurationSource`)와 환경 변수(`EnvironmentVariablesConfigurationSource`)에서 구성을 검색하는 구성 공급자를 추가합니다. JSON 파일에서 구성을 읽는 기능은 더 이상 기본적으로 포함되지 않습니다.

실제로 이전에는 메모리 및 환경 변수 구성 소스가 하나만 추가되었지만 이제는 메모리 구성 소스와 환경 변수 구성 소스가 여러 개 있습니다. 다른 소스는 구성 키에 다른 접두사를 사용합니다.

사용 중인 환경에서 `appsettings.json` 및 `appsettings.{environment}.json`을 사용하나요? 사용하는 호스팅에 따라 다를 수 있습니다. 애플리케이션이 컨테이너에서 실행되는 경우 환경 변수가 모두 필요할 수 있습니다. Azure 앱 구성 서비스를 사용하여 설정을 구성할 수도 있습니다. 이 경우 어떤 경우든 이를 공급자로 추가해야 합니다. JSON 파일을 사용하는 경우 이러한 구성 공급자를 빌더에 쉽게 추가할 수 있습니다.

### 로깅 공급자

`CreateBuilder` 메서드는 Windows 플랫폼에 네 개의 로거 공급자를 추가합니다: `ConsoleLoggerProvider`, `DebugLoggerProvider`, `EventLogLoggerProvider`, `EventSourceLoggerProvider`. `CreateSlimBuilder`를 사용하면 로깅 공급자가 추가되지 않습니다. 템플릿에서 생성된 코드만 로깅 빌더에 콘솔 로거 공급자를 추가합니다. 대신 다른 로깅 공급자가 필요한 경우 이 줄을 바꾸면 됩니다.

## JSON 직렬화기 소스 생성기

프로젝트 템플릿을 사용할 때(명령줄 옵션 `--publish-native-aot` 사용) 또는 Visual Studio에서 네이티브 AOT 게시 활성화 옵션을 선택하여 AOT를 사용하도록 설정한 경우 JSON 직렬화기 소스 생성기가 사용됩니다. 이 소스 생성기는 .NET 7부터 사용할 수 있으며, Todo 클래스에 대한 직렬화 코드를 생성하는 데 사용됩니다.

바이너리 AOT 코드를 생성하려면 런타임 중에 리플렉션 코드를 제거해야 합니다. .NET 개체를 직렬화할 때 직렬화기는 리플렉션을 사용하여 개체의 속성을 가져옵니다. 속성을 사용하여 직렬화 동작을 변경할 수 있습니다. 런타임을 사용하여 이 정보를 분석하면 성능이 저하되고 컴파일러의 트리밍 기능은 런타임 중에 필요한 코드를 제거하지 못하도록 제한됩니다. 이 문제를 해결하기 위해 소스 생성기를 사용하여 컴파일 타임에 직렬화기가 어떻게 동작해야 하는지 이 코드를 생성할 수 있습니다.

JSON 직렬화의 경우, 소스 코드 생성기는 .NET 7부터 사용할 수 있습니다. 생성된 템플릿에는 Todo 배열을 참조하는 `JsonSerializable` 속성이 적용된 이 `JsonSerializerContext` 파생 부분 클래스가 포함되어 있습니다. 이를 통해 컴파일러는 컴파일러가 생성한 코드로 `AppJsonSerializerContext` 클래스의 구현을 채웁니다.

```csharp
[JsonSerializable(typeof(Todo[]))]
internal partial class AppJsonSerializerContext : JsonSerializerContext
{
}
```

> 솔루션 탐색기에서 종속성을 사용하여 생성된 코드를 읽고 분석기를 연 다음 System.Text.Json.SourceGeneration을 선택할 수 있습니다. 여기에서 직렬화를 위한 `AppJsonSerializerContext`의 다양한 기능을 나타내는 여러 소스 코드 파일을 볼 수 있습니다.

소스 생성기에 대한 JSON 컨텍스트는 API `ConfigureHttpJsonOptions`를 사용하여 구성합니다. 이 구성을 사용하면 소스 생성 코드가 `Todo` 개체를 반환할 때 사용됩니다.

```csharp
builder.Services.ConfigureHttpJsonOptions(options =>
{
   options.SerializerOptions.AddContext<AppJsonSerializerContext>();
});
```

## 최소 API

`WebApplicationBuilder`를 완료하고 `WebApplication`을 빌드한 후 미들웨어가 구성됩니다. 최소 API를 사용하여 `/todos` 경로에 대한 그룹이 추가됩니다. HTTP GET 요청은 5개의 무작위 할일을 반환합니다. 하나의 할일을 반환하는 다른 API도 사용할 수 있습니다.

```csharp
var app = builder.Build();

var sampleTodos = TodoGenerator.GenerateTodos().ToArray();

var todoApi = app.MapGroup("/todos");
todosApi.MapGet("/", () => sampleTodos);
todosApi.MapGet("/{id}", (int id) =>
   sampleTodos.FirstOrDefault(a => a.Id == id) is { } todo
      ? Results.Ok(todo)
      : Results.NotFound());

app.Run();
```

## AOT 게시

Visual Studio 또는 `dotnet build` 및 `dotnet run`과 함께 CLI를 사용하여 애플리케이션을 빌드하고 실행하는 것은 평소와 같이 작동합니다. AOT에 대한 변경은 게시 단계를 통해 수행됩니다. 프로젝트 파일에는 `<PublishAot>true</PublishAot>` 항목이 포함됩니다. 이렇게 하면 AOT 컴파일러를 사용하여 애플리케이션을 게시할 수 있습니다.

이제 `dotnet publish`를 사용하면 바이너리 실행 파일이 생성됩니다. .NET 8 미리보기 3을 사용하면 파일 크기가 11MB를 약간 넘습니다. 애플리케이션을 실행하는 데 .NET 런타임이 필요하지 않습니다. `TodoAPI.exe`를 시작하면 애플리케이션이 시작되며, 이는 AOT가 없는 애플리케이션의 시작 시간보다 훨씬 빠릅니다. 시작 시간 및 메모리 사용량에 대한 차이점은 아래 링크를 통해 Microsoft Learn 설명서를 확인하십시오. Microsoft 문서에는 AOT를 사용하지 않을때와 사용할 때의 메모리 사용량이 86MB에서 40MB로, 시작 시간이 161ms에서 35ms로 변경된 것으로 나와 있습니다. 인상적입니다!

## 정리

.NET 8에는 많은 개선 사항이 포함되어 있으며, 여기에는 ASP.NET Core의 슬림 빌더 또는 `Random` 클래스의 `Shuffle` 메서드와 같은 몇 가지가 소개되어 있습니다. 물론 가장 큰 변화는 AOT입니다. .NET 7은 AOT 컴파일을 지원하면서 시작되었습니다. .NET 7을 사용하면 AOT는 C++ 응용 프로그램과 함께 사용할 수 있는 클래스 라이브러리를 만드는 등 몇 가지 시나리오에서만 작동합니다. .NET 8에서는 AOT를 사용할 수 있는 더 많은 시나리오에 대한 지원이 추가될 예정입니다. API를 만들기 위한 프로젝트 템플릿이 이에 대한 좋은 예입니다. 문서에서 (아직) 작동하지 않는 부분을 살펴보면(아래 링크 참조), 인증은 아직 지원되지 않지만 곧 지원될 JWT, Blazor Server, SignalR 및 MVC는 지원되지 않는 것으로 나열되어 있습니다. 이미 작동 중인 것은 gRPC, 최소 API, 응답 캐싱, 상태 확인, 웹 소켓 등입니다. .NET 8에서 더 많은 기능이 향상되기를 기대합니다.

학습과 프로그래밍을 즐기세요!

Christian

이 글이 마음에 드셨다면 커피 한 잔으로 응원해 주세요. 고마워요!

[![Buy Me A Coffee](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png align="left")](https://www.buymeacoffee.com/christiannagel)

## 자세한 정보

C# 프로그래밍에 대한 자세한 내용은 제 책과 워크샵에서 확인할 수 있습니다.

[**ASP.NET**](http://ASP.NET) [**Core updates in .NET 8 preview 3**](https://devblogs.microsoft.com/dotnet/asp-net-core-updates-in-dotnet-8-preview-3/?WT.mc_id=DT-MVP-10160)

[**Native AOT Compatibility**](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/native-aot?view=aspnetcore-8.0#aspnet-core-and-native-aot-compatibility)

[**Professional C# and .NET – 2021 Edition**](https://csharp.christiannagel.com/2021/09/13/professionalcsharp2021/)에서 C#에 대해 자세히 알아보세요.

[**Trainings**](https://www.cninnovation.com/training/)

[**Sample source code**](https://github.com/ProfessionalCSharp/ProfessionalCSharp2021)

상단 이미지는 DALL-E에서 제공하는 Microsoft Bing 이미지 크리에이터를 사용하여 만들었습니다.

---

[https://csharp.christiannagel.com/2023/04/19/api-dotnet8/](https://csharp.christiannagel.com/2023/04/19/api-dotnet8/)