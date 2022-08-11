## [Stl.Fusion] Replica 서비스 (feat. HelloBlazorHybrid)

[Stl.Fusion](https://github.com/servicetitan/Stl.Fusion)(이하 `Fusion`)은 실시간성 앱을 효율적으로 만들 수 있는 환경을 제공해주는 라이브러리입니다.

오늘 시간은 클라이언트에서 Replica 서비스를 이용해 Compute 서비스와 유사하게 무효화 및 캐시 기능을 사용하는 방법을 [HelloBlazorHybrid 샘플](https://github.com/servicetitan/Stl.Fusion)을 분석하면서 진행하도록 하겠습니다.

Replica 서비스의 기본적인 이해는 [Fusion 튜토리얼 - 4부: Replica 서비스](https://dimohy.slogger.today/fusion-4-replica)를 참조하세요.


## 개요

`Fusion`은 `분산 반응형 메모이제이션` 환경을 제공해주는 라이브러리 또는 프레임워크 입니다. 이것을 Fusion에서는 DREAM이라고 줄여서 말합니다.

> 좀 더 상세한 설명을 [Fusion 개요](https://dimohy.slogger.today/fusion-overview)를 참조하세요.

또한 `Fusion`은 Blazor Server와 Webassembly간 단일 코드베이스를 구축할 수 있는 환경을 제공해 Blazor를 이용해 서비스를 만드는데 둘 중 어느 것을 사용해야 하는지 고민하지 않아도 됩니다.

> `Fusion 튜토리얼 - 6부: Blazor 앱의 실시간 UI` 번역을 완료하면 이곳에 링크할 예정입니다.

Replica 서비스 (복제 서비스)를 이용하면 클라이언트로 노출해야 할 API를 `Web API` 형태로 그대로 사용할 수 있고 몇 가지 환경 구성을 통해 서버의 Compute 서비스를 클라이언트에서 DREAM의 이점을 그대로 활용하면서 사용할 수 있습니다.


## 샘플 정보

- GitHub 레파지토리 : https://github.com/servicetitan/Stl.Fusion.Samples
- HelloBlazorHybrid 샘플 : https://github.com/servicetitan/Stl.Fusion.Samples/tree/master/src/HelloBlazorHybrid
- 개발 환경
   - Visual Studio 2022, C#, .NET 6


## Replica 서비스

`Fusion`의 Replica 서비스는 Blazor Webassembly, 모바일 앱, 데스크톱 앱, 콘솔 등에서 Fusion으로 구성된 Compute 서비스를 클라이언트에서 동일하게 사용할 수 있도록 복제하는 서비스 입니다. 모든 환경 구성이 되면 사용법은 거의 동일하게 됩니다.

`HelloBlazorHybrid 샘플`을 분석하면서 Replica 서비스를 이해하도록 합시다.


### HelloBlazorHybrid 샘플

HelloBlazorHybrid 프로젝트는 Fusion이 제공하는 기능을 Blazor에서 확인할 수 있도록 구성된 샘플입니다. 프로젝트는 Blazor Server 및 Webassembly에서 모두 동작하도록 구성되어 있습니다.


#### 프로젝트 구성

- Abstrations

  인터페이스 및 구조입니다. Blazor Server 및 Webassembly에서 공통으로 필요 합니다.

- Server
  Blazor Server 및 Webassembly로 샘플을 호스팅합니다.

- Services
  서비스입니다. 호스팅 서비스 및 Compute 서비스가 있습니다.

- UI
  Blazor의 UI 입니다. Blazor Server 및 Webassembly에서 공통으로 사용합니다.


### Fusion 구성

Fusion을 사용하려면 Fusion 서비스를 추가해야 합니다.

| Server/Startup.cs, ConfigureServices()

```csharp
...
        // Fusion
        var fusion = services.AddFusion();
        var fusionServer = fusion.AddWebServer();
        fusion.AddFusionTime(); // IFusionTime is one of built-in compute services you can use
        services.AddScoped<BlazorModeHelper>();

        // Fusion services
        fusion.AddComputeService<ICounterService, CounterService>();
        fusion.AddComputeService<IWeatherForecastService, WeatherForecastService>();
        fusion.AddComputeService<IChatService, ChatService>();
        fusion.AddComputeService<ChatBotService>();
        // This is just to make sure ChatBotService.StartAsync is called on startup
        services.AddHostedService(c => c.GetRequiredService<ChatBotService>());
...
```

위의 구성으로 서버 측의 Compute 서비스가 등록이 됩니다.

> Compute 서비스 구현 및 사용법은 [Fusion 튜토리얼 - 4부: Replica 서비스](https://dimohy.slogger.today/fusion-4-replica)를 참조하세요.

```csharp
        // Shared UI services
        UI.Program.ConfigureSharedServices(services);
```

UI 관련 서비스를 초기화 합니다. 위의 경우는 Blazor Server에서 사용 합니다. Blazor Webassembly의 경우 `UI` 프로젝트의 `Program.cs`의 `Main()`에서 `ConfigureServices()` 에서 동일한 `UI.Program.ConfigureSharedServices(services)`를 호출함을 확인할 수 있습니다.

클라이언트(여기서는 웹브라우저에서 동작하는 Webassembly)에서 서버의 Compute 서비스를 호출하려면 Proxy 인터페이스를 통해 `Web API`를 호출하는 방법을 알아야 합니다.

| UI/Program.cs, ConfigureServices()

```csharp
...
        // Fusion service clients
        fusionClient.AddReplicaService<ICounterService, ICounterClientDef>();
        fusionClient.AddReplicaService<IWeatherForecastService, IWeatherForecastClientDef>();
        fusionClient.AddReplicaService<IChatService, IChatClientDef>();
...
```

`AddReplicaService()`를 통해 서비스 인터페이스와 `Web API`가 연결이 됩니다. 물론 실제로 저 인터페이스로 호출하려면 인터페이스를 구현한 구현체가 있어야 하는데 Fusion은 이 Proxy 구현체를 런타임에서 만들어 인스턴스를 생성해줍니다. 이제 인터페이스를 통해 원격 API를 호출할 수 있게 되는 것입니다.

그렇다면 단순히 Proxy를 통해 `Web API`를 호출하는 것과 어떤 차이가 있을까요?


### Replica 서비스의 동작

`fusionClient.AddReplicaService<TService, TClient>`에 의해 Replica 서비스가 등록되면 이제 마치 서비스 구현체의 인스턴스가 있는 것처럼 호출할 수 있게 됩니다.

| UI/Counter.razor
```razor
@inject ICounterService CounterService
```

razor에서 `@inject`으로 원하는 Replica 서비스의 인스턴스를 가져올 수 있습니다.

```csharp
CounterService.Increment();
```

인터페이스에 맞는 메소드를 호출 할 수 있으며 Fusion에 의해 생성된 Proxy 구현체의 인스턴스에 의해 `ICounterClientDef`인터페이스에 정의된 규칙으로 `Web API`를 호출합니다.

| Abstractions/Clients.cs
```csharp
[BasePath("counter")]
public interface ICounterClientDef
{
    [Post("increment")]
    Task Increment(CancellationToken cancellationToken = default);

    [Get("get")]
    Task<int> Get(CancellationToken cancellationToken = default);
}
```

특성에 의해 `/api/counter/increment`의 API가 호출이 되는데 이것은 다음의 컨트롤러에 의해 서버의 Compute 서비스를 사용합니다.

| Server/CounterController.cs
```csharp
[Route("api/[controller]/[action]")]
[ApiController, JsonifyErrors, UseDefaultSession]
public class CounterController : ControllerBase, ICounterService
{
    private readonly ICounterService _counter;

    public CounterController(ICounterService counter) 
        => _counter = counter;

    [HttpGet, Publish]
    public Task<int> Get(CancellationToken cancellationToken = default)
        => _counter.Get(cancellationToken);

    [HttpPost]
    public Task Increment(CancellationToken cancellationToken = default)
        => _counter.Increment(cancellationToken);
}
```

서비스의 `Get()`의 특성 중 `Publish`가 있는데 클라이언트가 게시를 요청할 경우 Get 출력이 게시 되도록 합니다.

Replica 서비스의 흥미로운 점은 Replica 서비스 또한 상태가 무효화 되기 전까지 값을 캐시 한다는 점입니다.  서버의 상태가 무효화되지 않는 한 클라이언트에서 값을 읽은 후 다시 읽으려 할 때 서버에 요청하지 않고 캐시된 값을 사용하게 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1659456193261/ofX_IzfIN.png align="left")

웹 브라우저에서 동일한 페이지를 여러 개 만들고 `Increment` 버튼을 눌렀을 때 `Count`가 동시에 올라가는 것을 볼 수 있습니다.

이번에는 `IWeatherForecastService`를 보시죠.

| Abstractions/IWeatherForcastService.cs
```csharp
public interface IWeatherForecastService
{
    [ComputeMethod]
    Task<WeatherForecast[]> GetForecast(DateTime startDate, CancellationToken cancellationToken = default);
}
```

시작 날짜로 예보를 구하는 Compute 서비스 입니다.

| Services/WeatherForecastService.cs
```csharp
public class WeatherForecastService : IWeatherForecastService
{
    private static readonly string[] Summaries = {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };

    [ComputeMethod(AutoInvalidationDelay = 1)]
    public virtual Task<WeatherForecast[]> GetForecast(
        DateTime startDate, CancellationToken cancellationToken = default)
    {
        var rng = new Random();
        return Task.FromResult(Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = startDate.AddDays(index),
            TemperatureC = rng.Next(-20, 55),
            Summary = Summaries[rng.Next(Summaries.Length)]
        }).ToArray());
    }
}
```

한 가지 다른 점이 있습니다. `ComputeMethod()` 특성에서  `AutoInvalidationDelay`가 1로 설정되어 있는데 1초마다 자동으로 무효화를 해주는 설정입니다. 이것으로 인해 매 초마다 예보 정보가 갱신 될 것입니다.

다음은 컨트롤러 입니다.

| Server/Controllers/WeatherForecastController.cs
```csharp
[Route("api/[controller]/[action]")]
[ApiController, JsonifyErrors, UseDefaultSession]
public class WeatherForecastController : ControllerBase, IWeatherForecastService
{
    private readonly IWeatherForecastService _forecast;

    public WeatherForecastController(IWeatherForecastService forecast) 
        => _forecast = forecast;

    [HttpGet, Publish]
    public Task<WeatherForecast[]> GetForecast(DateTime startDate,
        CancellationToken cancellationToken = default)
        => _forecast.GetForecast(startDate, cancellationToken);
}
```

이 컨트롤러에 의해 `/api/WeatherForecast/getForecast`의 `Web API`가 노출됩니다.

```csharp
[BasePath("weatherForecast")]
public interface IWeatherForecastClientDef
{
    [Get("getForecast")]
    Task<WeatherForecast[]> GetForecast(DateTime startDate, CancellationToken cancellationToken = default);
}
```

`fusionClient.AddReplicaService<IWeatherForecastService, IWeatherForecastClientDef>();`에 의해 해당 `Web API`가 Replica 서비스로 연결이 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1659457152979/LzY6ksVv2.png align="left")


## 정리

오늘은 `Stl.Fusion`에서 제공하는 `HelloBlazorHybrid 샘플`을 통해 Fusion의 Replica 서비스에 대해 알아 보았습니다. Replica 서비스를 이용하면 마치 내부 기능을 사용하는 것 처럼 서버의 Compute 서비스를 사용할 수 있게 되며 상태 무효화시 클라이언트까지 변경된 값이 잘 적용됨을 확인할 수 있습니다.

