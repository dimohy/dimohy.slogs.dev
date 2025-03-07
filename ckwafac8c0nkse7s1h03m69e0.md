---
title: ".NET Core 탐색 1부 - .NET 6의 ConfigurationManager 내부 보기"
datePublished: Mon Nov 22 2021 08:44:14 GMT+0000 (Coordinated Universal Time)
cuid: ckwafac8c0nkse7s1h03m69e0
slug: net-core-1-net-6-configurationmanager
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1637233840872/ruKU0vWYO.png
tags: dotnet

---

***본 시리즈는 Andrew Lock 님의 [.NET Core 6 탐색](https://andrewlock.net/series/exploring-dotnet-6/) 연재를 번역한 것입니다.***

이 시리즈에서는 .NET 6의 새로운 기능 중 일부를 살펴보겠습니다. .NET 및 ASP.NET 팀의 많은 게시물을 포함하여 이미 .NET 6에 작성된 많은 콘텐츠가 있습니다. 이 시리즈에서는 이러한 일부 기능 중 일부 뒤에 있는 코드를 살펴보겠습니다.

이 첫 번째 게시물은 왜 `ConfigurationManager` 클래스가 추가되었는지, 그리고 구현에 사용된 일부 코드를 살펴봅니다.


## 잠깐, ConfigurationManager이 무엇인가요?

여러분의 첫 번째 물음이 `ConfigurationManager`가 "무엇인가"라면 큰 발표를 놓친 것이 아니니, 걱정하지 마세요.!

`ConfigurationManager`는 ASP.NET Core 시작 코드를 단순화하는 데 사용되는 ASP.NET Core의 새로운 `WebApplication` 모델을 지원하기 위해 추가되었습니다. 그러나 `ConfigurationManager`는 구현 세부 사항에 가깝습니다. 이것은 특정 시나리오(곧 설명할 예정임)를 최적화 하기 위해 도입되었지만 대부분의 경우 사용 중인지 알 필요는 없습니다.

우리가 `ConfigurationManager`에 도달하기 전에 그것이 대체하는 것과 그 이유를 살펴볼 것입니다.


## .NET 구성

.NET 5는 구성과 관련된 여러 유형을 노출하지만 앱에서 직접 사용하는 두 가지 기본 유형은 다음과 같습니다.

- `IConfigurationBuilder` - 구성 요소를 추가하는데 사용됩니다. `Build()` 빌더 호출을 통해 각 구성 요소를 읽고 최종 구성을 빌드합니다.
- `IConfugrationRoot` - 최종 "빌드된" 구성을 나타냅니다.

`IConfigurationBuilder` 인터페이스는 대부분 구성 요소 목록을 둘러싼 래퍼입니다. 구성 공급자(Configuration Provider)는 일반적으로 확장 메소드(`AddJsonFile()`이나 `AddZureKeyVault()`와 같은)로 구성요소를 `Sources` 목록으로 포함합니다.

```csharp
public interface IConfigurationBuilder
{
    IDictionary<string, object> Properties { get; }
    IList<IConfigurationSource> Sources { get; }
    IConfigurationBuilder Add(IConfigurationSource source);
    IConfigurationRoot Build();
}
```
한편 `IConfigurationRoot`은 최종 "계층화된" 구성 요소를 나타내며 각 구성 요소의 모든 값을 결합하여 모든 구성 값의 최종 "평면" 보기를 제공합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1637233840872/ruKU0vWYO.png)
최신 구성 공급자(환경 변수)는 이전 구성 공급자(appsetting.json, sharedsettings.json)에서 추가한 값을 덮어씁니다. 내 책 [ASP.NET Core in Action, Second Edtion](https://www.manning.com/books/asp-net-core-in-action-second-edition?utm_source=aspnetcore-in-action&utm_medium=affiliate&utm_campaign=book_lock2_asp_5_19_20&a_aid=aspnetcore-in-action&a_bid=44c089ee)에서 가져옴

.NET 5 및 이전 버전에서 `IConfigurationBuilder` 및 `IConfigurationRoot` 인터페이스는 `ConfigurationBuilder` 및 `ConfigurationRoot` 각각에 의해 구현됩니다. 유형을 직접 사용하는 경우 다음과 같이 할 수 있습니다.

```csharp
var builder = new ConfigurationBuilder();

// 정적 변수를 추가함
builder.AddInMemoryCollection(new Dictionary<string, string>
{
    { "MyKey", "MyValue" },
});

// JSON 파일로부터 변수를 추가함
builder.AddJsonFile("appsettings.json");

// IConfigurationRoot 인스턴스를 생성함
IConfigurationRoot config = builder.Build();

string value = config["MyKey"]; // 변수값 읽음
IConfigurationSection section = config.GetSection("SubSection"); // 섹션을 얻음
```

일반적으로 ASP.NET Core 앱에서는 `ConfigurationBuilder`를 직접 만들거나 `Build()`를 호출하지 않을 것이지만 그렇지 않다면 이것이 배후에서 일어나는 일입니다. 두 유형 사이에는 명확한 구분이 있으며 대부분의 경우 구성 시스템이 잘 작동하므로 .NET 6에서 새로운 유형이 필요한 이유는 무엇일까요?


## .NET 5의 "부분 구성 빌드" 문제

이 디자인의 중요 문제는 구성을 "부분적"으로 빌드해야 할 때 입니다. 이는 Azure Key Vault와 같은 서비스 또는 데이터베이스 구성을 저장할 때 일반적인 문제입니다.

예를 들어 ASP.NET Core에서 `ConfigureAppConfiguration()` 내부의 [Azure Key Vault에서 시크릿을 읽는 제안된 방법](https://docs.microsoft.com/en-us/aspnet/core/security/key-vault-configuration?view=aspnetcore-5.0#use-application-id-and-x509-certificate-for-non-azure-hosted-apps)입니다.

```csharp
.ConfigureAppConfiguration((context, config) =>
{
    // "normal" configuration etc
    config.AddJsonFile("appsettings.json");
    config.AddEnvironmentVariables();

    if (context.HostingEnvironment.IsProduction())
    {
        IConfigurationRoot partialConfig = config.Build(); // build partial config
        string keyVaultName = partialConfig["KeyVaultName"]; // read value from configuration
        var secretClient = new SecretClient(
            new Uri($"https://{keyVaultName}.vault.azure.net/"),
            new DefaultAzureCredential());
        config.AddAzureKeyVault(secretClient, new KeyVaultSecretManager()); // add an extra configuration source
        // The framework calls config.Build() AGAIN to build the final IConfigurationRoot
    }
})
```

Azure Key Vault 공급자를 구성하려면 구성 값이 필요하므로 닭과 달걀의 문제가 발생합니다. 구성을 빌드할 때까지 구성 요소를 추가할 수 없습니다!

다음과 같은 해결책:
- '초기' 구성 값 추가합니다.
- `IConfigurationBuilder.Build()`를 호출해서 "부분적" 구성 결과를 빌드 합니다.
- `IConfigurationRoot` 결과에서 필요한 구성 값을 검색합니다.
- 이 값을 이용해서 나머지 구성 요소를 추가합니다.
- 프레임워크는 `IConfigurationBuilder.Build()`를 암시적으로 호출해서 최종 `IConfigurationRoot` 항목을 생성하고 최종 앱 구성에 사용합니다.

좀 산만하지만 그 자체로는 아무런 문제가 없습니다. 그렇다면 단점은 무엇인가요?

단점은 `Build()`를 두 번 호출해야 한다는 점입니다. 한번은 첫 번째 요소들만 이용해서 `IConfigurationRoot`를 빌드하고  다시 한번 모든 요소들을 이용해서 `IConfigurationRoot`을 빌드합니다.

기본 `ConfigurationBuilder` 구현에서 `Build()`를 호출하는 것은 모든 요소를 반복하고 공급자를 로딩하고 이를 `ConfigurationRoot`의 새 인스턴스에 전달합니다.

```csharp
public IConfigurationRoot Build()
{
    var providers = new List<IConfigurationProvider>();
    foreach (IConfigurationSource source in Sources)
    {
        IConfigurationProvider provider = source.Build(this);
        providers.Add(provider);
    }
    return new ConfigurationRoot(providers);
}
```

그런 다음 `ConfigurationRoot`은 각 공급자를 차례로 반복하고 구성 값을 로딩합니다.

```csharp
public class ConfigurationRoot : IConfigurationRoot, IDisposable
{
    private readonly IList<IConfigurationProvider> _providers;
    private readonly IList<IDisposable> _changeTokenRegistrations;

    public ConfigurationRoot(IList<IConfigurationProvider> providers)
    {
        _providers = providers;
        _changeTokenRegistrations = new List<IDisposable>(providers.Count);

        foreach (IConfigurationProvider p in providers)
        {
            p.Load();
            _changeTokenRegistrations.Add(ChangeToken.OnChange(() => p.GetReloadToken(), () => RaiseChanged()));
        }
    }
    // ... 나머지 구현
}
```

`Build()` 이 앱이 시작할 때 두 번 호출되면, 이 모든 것이 두 번 발생합니다.

일반적으로 구성 요소에서 데이터를 두 번 이상 가져오는 것은 아무런 해가 없지만 불필요한 작업이며 종종 (상대적으로 느린) 파일 읽기 등을 수반합니다.

이것은 일반적인 패턴으로, .NET 6에서는 이 "재구축"을 피하기 위해 새로운 유형 `ConfigurationManager` 가 도입되었습니다. 


## .NET 6의 구성 관리자(Configuration Manager)

.NET 6의 '단순화된' 애플리케이션 모델의 일부로 .NET 팀은 `ConfigurationManager`라는 새로운 구성 유형을 추가했습니다. 이 유형은 `IConfigurationBuilder`와 `IConfigurationRoot`을 모두 구현합니다. 두 가지 유형을 단일 유형으로 결합한 .NET 6은 이전 섹션에 표시된 일반적인 패턴을 최적화 할 수 있습니다.

`ConfigurationManager`와 함께 `IConfigurationSource`가 추가되면 (`AddJsonFile()`을 예를 들어 호출할 때) 공급자가 즉시 로딩 되고 구성이 업데이트 됩니다. 이렇게 하면 부분 빌드 시나리오에서 구성 요소를 두 번 이상 로딩 하지 않아도 됩니다.

요소들을 `IList<IConfigurationSource>`로 노출하는 `IConfigurationBuilder` 인터페이스로 인해서 이를 구현하는 것은 조금 더 어렵습니다.

```csharp
public interface IConfigurationBuilder
{
    IList<IConfigurationSource> Sources { get; }
    // .. 다른 멤버들
}
``` 

`ConfigurationManager`관점에서 이것의 문제는 `IList<>`가 `Add()` 및 `Remove()` 함수를 노출한다는 점입니다. 단순 `List<>`가 사용된 경우 소비자는 `ConfigurationManager`가 알지 못하는 상태에서 구성 공급자를 추가하거나 제거할 수 있습니다.

이 문제를 해결하기 위해 `ConfigurationManager`는 사용자 정의 `IList<>` 구현을 사용합니다. 여기에는 `ConfigurationManager` 인스턴스에 대한 참조가 포함되어 있어서 변경 사항이 구성에 반영될 수 있습니다.

```csharp
private class ConfigurationSources : IList<IConfigurationSource>
{
    private readonly List<IConfigurationSource> _sources = new();
    private readonly ConfigurationManager _config;

    public ConfigurationSources(ConfigurationManager config)
    {
        _config = config;
    }

    public void Add(IConfigurationSource source)
    {
        _sources.Add(source);
        _config.AddSource(source); // 요소를 ConfigurationManager에 추가
    }

    public bool Remove(IConfigurationSource source)
    {
        var removed = _sources.Remove(source);
        _config.ReloadSources(); // ConfigurationManager의 요소들을 초기화
        return removed;
    }

    // ... 추가적인 구현
}
```

사용자 `IList<>` 구현을 사용하면서 `ConfigurationManager`는 새 요소가 추가될 때마다 `AddSource()`를 호출하도록 보장합니다. 이것이 `ConfigurationManager`의 이점입니다. `AddSource()`를 호출하면 요소가 즉시 로드됩니다.

```csharp
public class ConfigurationManager
{

    private void AddSource(IConfigurationSource source)
    {
        lock (_providerLock)
        {
            IConfigurationProvider provider = source.Build(this);
            _providers.Add(provider);

            provider.Load();
            _changeTokenRegistrations.Add(ChangeToken.OnChange(() => provider.GetReloadToken(), () => RaiseChanged()));
        }

        RaiseChanged();
    }
}
```

이 메소드는 즉시 `IConfigurationSource`에서 `Build`를 호출해서 `IConfigurationProvider`를 만들고 공급자 목록에 추가합니다.

다음으로 이 메서드는 `IConfigurationProvider.Load()`를 호출합니다. 이렇게 하면 데이터가 공급자(예: 환경 변수, JSON파일 또는 Azure Key Vault)에 로드되며 이것이 "비싼" 단계 입니다! `IConfigurationBuilder`에 요소를 추가하고 여러 번 빌드해야 하는 "정상적인" 경에는 "최적의" 접근 방식이 제공됩니다. 요소는 한 번만 로드 됩니다.

`ConfigurationManager`에서`Build()`의 구현은 이제 단순히 자신을 반환하는 엉터리 입니다.

```csharp
IConfigurationRoot IConfigurationBuilder.Build() => this;
```

물론 소프트웨어 개발은 모두 트레이드 오프에 관한 것입니다. 원소를 추가할 때 원소를 점진적으로 구축하는 것은 원소만 추가하는 경우에 잘 작동합니다. 하지만 `Clear()`, `Remove()`또는 인덱서와 같은 다른 `IList<>`함수를 호출하는 경우 `ConfigurationManager`는 `ReloadSources()`를 호출해야 합니다.

```csharp
private void ReloadSources()
{
    lock (_providerLock)
    {
        DisposeRegistrationsAndProvidersUnsynchronized();

        _changeTokenRegistrations.Clear();
        _providers.Clear();

        foreach (var source in _sources)
        {
            _providers.Add(source.Build(this));
        }

        foreach (var p in _providers)
        {
            p.Load();
            _changeTokenRegistrations.Add(ChangeToken.OnChange(() => p.GetReloadToken(), () => RaiseChanged()));
        }
    }

    RaiseChanged();
}
```

보시다시피 요소 중 하나가 변경되면 `ConfigurationManager`는 모든 것을 제거하고 다시 시작해서 각 원소를 반복하고 다시 로드 해야 합니다. 구성 요소를 많이 조작하는 경우 비용이 빠르게 증가하고 `ConfigurationManager`의 원래 장점을 완전히 무효화 할 수 있습니다.

물론 원소를 제거하는 것은 매우 이례적인 일입니다. 일반적으로 공급자를 추가하는 것 외에 다른 작업을 수행할 이유가 없으므로 `ConfigurationManager`는 가장 일반적인 경우에 매우 최적화 되어 있습니다. 누가 짐작이나 했을까요? 😉

다음 표는 `ConfigurationBuilder`와 `ConfigurationManager`를 모두 사용하는 다양한 작업의 상대적 비용에 대한 최종 요약을 제공합니다.

|작업|ConfigurationBuilder|ConfigurationManager|
|----|---|---|
|요소 추가|저렴|보통|
|IConfigurationRoot 부분 구축|비쌈|매우 쌈(엉터리)|
|IConfigurationRoot 완전 구축|비쌈|매우 쌈(엉터리)|
|요소 제거|저렴|비쌈|
|요소 변경|저렴|비쌈|

## 그렇다면 ConfigurationManager에 관심을 가져야 합니까?

따라서 모든 방법을 읽었으므로 `ConfigurationManager` 나 `ConfigurationBuilder`에 관심을 가져야 합니까?

아마 아닐 것입니다.

.NET 6에 도입된 `WebApplicationBuilder`는 구성을 부분적으로 빌드해야 하는 위에서 설명한 사용 사례에 최적화된 `ConfigurationManager`를 사용합니다.

그러나 이전 버전의 ASP.NET Core에 도입된 `WebHostBuilder` 또는 `HostBuilder`는 여전히 .NET 6에서 매우 많이 지원되며 뒤에서 `ConfigurationBuilder` 및 `ConfigurationRoot` 유형을 계속 사용합니다.

내가 생각할 수 있는 유일한 상황은 `IConfigurationBuilder` 또는 `IConfigurationRoot`이 구체적인 유형인 `ConfigurationBuilder`또는 `ConfigurationRoot`인 경우 어딘가 에서 의존하고 있는 경우입니다. 그것은 나에게 매우 가능성이 없어 보이며, 당신이 그것에 의존한다면 그 이유를 알고 싶습니다!

하지만 그 틈새 예외를 제외하고는 "오래된" 유형이 사라지지 않으므로 걱정할 필요가 없습니다. "부분 빌드"를 수행해야 하고 새로운 `WebApplicationBuilder`를 사용하는 경우 앱의 성능이 조금 더 향상된다는 사실에 만족하세요! 


## 요약

이 게시물에서는 .NET 6에 도입되고 최소 API 예제에서 사용되는 새로운 `WebApplicationBuilder`에서 사용되는 새로운 `ConfigurationManager` 유형에 대해 설명했습니다. `ConfigurationManager`는 구성을 "부분적으로 빌드"해야 하는 일반적인 상황을 최적화 하기 위해 도입되었습니다. 이는 일반적으로 구성 공급자가 일부 구성 자체를 필요로 하기 때문입니다. 예를 들어 Azure Key Vault에서 시크릿을 로드 하려면 사용할 자격 증명 모음을 나타내는 구성이 필요합니다.

`ConfigurationManager`는 `Build()`를 호출할 때까지 기다리지 않고 요소가 추가되는 즉시 로드하여 이 시나리오를 최적화 합니다. 이렇게 하면 "부분 빌드" 시나리오에서 구성을 "재빌드"할 필요가 없습니다. 단점은 다른 작업(예: 요소 제거)의 비용이 많이 든다는 것입니다.


## 원문

[Part 1 - Looking inside ConfigurationManager in .NET 6](https://andrewlock.net/exploring-dotnet-6-part-1-looking-inside-configurationmanager-in-dotnet-6/) | Andrew Lock