## Fusion 튜토리얼 - 4부: Replica 서비스

> 본 글은 [Stl.Fusion](https://github.com/servicetitan/Stl.Fusion)의 튜토리얼 [Part 4: Replica Services](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part04.md) 문서를 번역한 것입니다.

이것을 다룬 동영상:

%[https://youtu.be/_wFhi11Eb0o]

Replica 서비스는 동일한 웹 API 클라이언트보다 더 효율적으로 `IComputed<T>` 동작을 고려하는 Compute 서비스의 원격 프록시입니다.

다시 말해:
- 서버 측에서 일치하는 `IComputed<T>`를 모방하는 `IComputed<T>`를 사용하여 모든 호출에 대해 결과를 유사하게 백업합니다. 따라서 클라이언트 측 Replica 서비스는 다른 클라이언트 측 Compute 서비스에서 사용할 수 있습니다. 짐작할 수 있듯이 서버 측 종속성이 무효화되면 클라이언트 측 복제본(`IComputed<T>`도 무효화됨)이 무효화됩니다. 이를 사용하는 모든 클라이언트 측 계산을 무효화하게 됩니다.
- 마찬가지로 일관성 있는 복제본을 캐시합니다. 즉, Replica 서비스는 일관성 있는 복제본을 계속 사용할 수 있는 경우 원격 호출하지 않습니다. "계산"을 "RPC 호출"로 바꾸면 Compute 서비스와 동일한 동작입니다.

Replica 서비스가 복제본의 초기 버젼을 생성하는 방법은 다소 명확합니다. 그것은 단순히 HTTP 끝점을 호출해서 `IComputed<T>`의 복사본을 가져올 수 있습니다. 하지만 무효화 알림은 어떻게 받을까요?

무효화 및 업데이트 메시지는 현재 추가로 WebSocket 기반의 `Publisher`-`Replicator` 채널을 통해 전달됩니다. 연결은 클라이언트에서 첫 번째 복제본이 생성될 때 이루어집니다. 채널은 지정된 `Publisher` (~ 서버)에서 이러한 모든 알림을 추가로 제공하는데 사용됩니다.

복원력 (연결 해제 시 재연결, 재연결 시 모든 복제본의 상태 자동 새로 고침 등)은 구현과 프로토콜 모두에 번들로 제공됩니다. (예: 이것이 모든 `IComputed<T>`에 `Version` 속성이 있는 주된 이유입니다.)

마지막으로 Replica 서비스는 인터페이스일 뿐입니다. 일반적으로 "모방"하는 Compute 서비스의 모든 메서드를 선언합니다. 인터페이스는 메서드 호출이 해당 HTTP 끝점에 매핑되어야 하는 방법을 설명하기 위해서만 필요합니다.

Fusion은 현재 [RestEase](https://github.com/canton7/RestEase)와 자체 [Castle.DynamicProxy](http://www.castleproject.org/projects/dynamicproxy/) 기반 프록시에 의존하는 런타임에 Replica 서비스 인터페이스를 구현합니다.

아래 시퀀스 다이어그램은 일반 Web API 클라이언트 (예: 일반 RestEase 클라이언트)가 호출을 처리할 때 어떤 일이 발생하는지 보여줍니다. "Web API"는 기본 서비스(이 예에서는 `GreetingService`)에 대한 호출을 전달하는 컨트롤러입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1659279673622/oaPPjNzV2.png align="left")

그리고 이것은 Replica 서비스가 호출을 처리하고 나중에 값을 무효화 및 업데이트할 때 어떤 일이 생기는지 보여주는 유사한 다이어그램입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1659279722204/uxcA3DcGZ.png align="left")

> 6단계는 실제로 발생하지 않습니다. 우리는 WebSocket을 사용하므로 확인을 보낼 필요가 없습니다.

이 프로세스에 대한 Gant 차트는 다음과 같습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1659279795663/xDoVPapRh.png align="left")

자, 이제 어떻게 작동하는지 알아보기 위해 몇 가지 코드를 작성해 보겠습니다. 불행히도 이번에는 코드의 양이 약간 폭발할 것입니다. 이는 대부분 Compute 서비스 자체를 호스팅하는 웹 서버, 호출 가능한 끝점 등을 게시하는 컨트롤러 등이 필요하기 때문입니다.

- 공통 인터페이스 (아직 이 코드를 실행하지 마세요):

```csharp
// 이상적으로는 Replica 서비스가 해당 Compute 서비스와 정확히 동일하기를 원합니다.
// 이렇게 하는 좋은 방법은 Compute 서비스에서 구현해야 하는 인터페이스를 노출하고
// Fusion이 동일한 인터페이스를 통해 클라이언트를 "노출"하도록 지시하는 것입니다.
public interface ICounterService
{
    [ComputeMethod]
    Task<int> Get(string key, CancellationToken cancellationToken = default);
    Task Increment(string key, CancellationToken cancellationToken = default);
    Task SetOffset(int offset, CancellationToken cancellationToken = default);
}
```

- 웹 호스트 서비스 (아직 이 코드를 실행하지 마세요):

```csharp
public class CounterService : ICounterService
{
    private readonly ConcurrentDictionary<string, int> _counters = new ConcurrentDictionary<string, int>();
    private readonly IMutableState<int> _offset;

    public CounterService(IStateFactory stateFactory)
        => _offset = stateFactory.NewMutable<int>();

    [ComputeMethod] // 선택 사항: 이 속성은 인터페이스에서 상속됩니다.
    public virtual async Task<int> Get(string key, CancellationToken cancellationToken = default)
    {
        WriteLine($"{nameof(Get)}({key})");
        var offset = await _offset.Use(cancellationToken);
        return offset + (_counters.TryGetValue(key, out var value) ? value : 0);
    }

    public Task Increment(string key, CancellationToken cancellationToken = default)
    {
        WriteLine($"{nameof(Increment)}({key})");
        _counters.AddOrUpdate(key, k => 1, (k, v) => v + 1);
        using (Computed.Invalidate())
            _ = Get(key, default);
        return Task.CompletedTask;
    }

    public Task SetOffset(int offset, CancellationToken cancellationToken = default)
    {
        WriteLine($"{nameof(SetOffset)}({offset})");
        _offset.Value = offset;
        return Task.CompletedTask;
    }
}

// 서비스를 게시하려면 Web API 컨트롤러가 필요합니다.
[Route("api/[controller]/[action]")]
[ApiController, JsonifyErrors, UseDefaultSession]
public class CounterController : ControllerBase
{
    private ICounterService Counters { get; }

    public CounterController(ICounterService counterService)
        => Counters = counterService;

    // 게시는 클라이언트가 게시를 요청한 경우 Get 출력이 게시되도록 합니다.
    // - 게시 대상이 생성됩니다.
    // - ID는 응답 헤더에서 공유됩니다.
    [HttpGet, Publish]
    public Task<int> Get(string key)
    {
        key ??= ""; // 빈 값은 기본적으로 null 값에 바인딩 됩니다.
        WriteLine($"{GetType().Name}.{nameof(Get)}({key})");
        return Counters.Get(key, HttpContext.RequestAborted);
    }

    [HttpPost]
    public Task Increment(string key)
    {
        key ??= ""; // 빈 값은 기본적으로 null 값에 바인딩 됩니다.
        WriteLine($"{GetType().Name}.{nameof(Increment)}({key})");
        return Counters.Increment(key, HttpContext.RequestAborted);
    }

    [HttpPost]
    public Task SetOffset(int offset)
    {
        WriteLine($"{GetType().Name}.{nameof(SetOffset)}({offset})");
        return Counters.SetOffset(offset, HttpContext.RequestAborted);
    }
}
```

- 클라이언트 서비스 (아직 이 코드를 실행하지 마세요):

```csharp
// ICounterClientDef는 ICounterService 메서드가 HTTP 메서드에 매핑되는 방식을 알려줍니다.
// 앞으로 보게 되겠지만, 이것은 Replica 서비스(ICounterService 구현)에 의해 사용됩니다.
[BasePath("counter")]
public interface ICounterClientDef
{
    [Get("get")]
    Task<int> Get(string key, CancellationToken cancellationToken = default);
    [Post("increment")]
    Task Increment(string key, CancellationToken cancellationToken = default);
    [Post("setOffset")]
    Task SetOffset(int offset, CancellationToken cancellationToken = default);
}
```

- `CreateHost` 및 `CreateClientServices` 메서드 (아직 이 코드를 실행하지 마세요):

```csharp
public static IHost CreateHost()
{
    var builder = Host.CreateDefaultBuilder();
    builder.ConfigureHostConfiguration(cfg =>
        cfg.AddInMemoryCollection(new Dictionary<string, string>() { { "Environment", "Development" } }));
    builder.ConfigureLogging(logging =>
        logging.ClearProviders().SetMinimumLevel(LogLevel.Information).AddDebug());
    builder.ConfigureServices((b, services) =>
    {
        var fusion = services.AddFusion();
        fusion.AddWebServer();
        // Registering Compute Service
        fusion.AddComputeService<ICounterService, CounterService>();
        services.AddRouting();
        // And its controller
        services.AddControllers().AddApplicationPart(Assembly.GetExecutingAssembly());
    });
    builder.ConfigureWebHost(b =>
    {
        b.UseKestrel();
        b.UseUrls("http://localhost:50050/");
        b.Configure((ctx, app) =>
        {
            app.UseWebSockets();
            app.UseRouting();
            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
                endpoints.MapFusionWebSocketServer();
            });
        });
    });
    return builder.Build();
}

public static IServiceProvider CreateClientServices()
{
    var services = new ServiceCollection();
    var baseUri = new Uri($"http://localhost:50050/");
    var apiBaseUri = new Uri($"{baseUri}api/");

    var fusion = services.AddFusion();
    var fusionClient = fusion.AddRestEaseClient();
    fusionClient.ConfigureHttpClient((c, name, options) => {
        // Replica 서비스는 IHttpClientFactory를 사용하여 HttpClient를 구성하므로
        // 모든 HttpClient가 기본적으로 BaseAddress = apiBaseUri를 갖도록 만드는 올바른 방법입니다.
        options.HttpClientActions.Add(client => client.BaseAddress = apiBaseUri);
    });
    fusionClient.ConfigureWebSocketChannel(c => new () {
        BaseUri = baseUri,
    });
    // Registering replica service
    fusionClient.AddReplicaService<ICounterService, ICounterClientDef>();

    return services.BuildServiceProvider();
}
```

마지막으로 Replica 서비스를 사용해 볼 준비가 되었습니다.

```csharp
using var host = CreateHost();
await host.StartAsync();
WriteLine("Host started.");

            using var stopCts = new CancellationTokenSource();
var cancellationToken = stopCts.Token;

async Task Watch<T>(string name, IComputed<T> computed)
{
    for (; ; )
    {
        WriteLine($"{name}: {computed.Value}, {computed}");
        await computed.WhenInvalidated(cancellationToken);
        WriteLine($"{name}: {computed.Value}, {computed}");
        computed = await computed.Update(cancellationToken);
    }
}

var services = CreateClientServices();
var counters = services.GetRequiredService<ICounterService>();
var aComputed = await Computed.Capture(_ => counters.Get("a"));
_ = Task.Run(() => Watch(nameof(aComputed), aComputed));
var bComputed = await Computed.Capture(_ => counters.Get("b"));
_ = Task.Run(() => Watch(nameof(bComputed), bComputed));

await Task.Delay(200);
await counters.Increment("a");
await Task.Delay(200);
await counters.SetOffset(10);
await Task.Delay(200);

stopCts.Cancel();
await host.StopAsync();
```

출력:
```
Host started.
CounterController.Get(a)
Get(a)
aComputed: 0, ReplicaClientComputed`1(Intercepted:ICounterServiceProxy.Get(a, System.Threading.CancellationToken) @4f, State: Consistent)
CounterController.Get(b)
Get(b)
bComputed: 0, ReplicaClientComputed`1(Intercepted:ICounterServiceProxy.Get(b, System.Threading.CancellationToken) @6j, State: Consistent)
CounterController.Increment(a)
Increment(a)
aComputed: 0, ReplicaClientComputed`1(Intercepted:ICounterServiceProxy.Get(a, System.Threading.CancellationToken) @4f, State: Invalidated)
Get(a)
aComputed: 1, ReplicaClientComputed`1(Intercepted:ICounterServiceProxy.Get(a, System.Threading.CancellationToken) @2m, State: Consistent)
CounterController.SetOffset(10)
SetOffset(10)
bComputed: 0, ReplicaClientComputed`1(Intercepted:ICounterServiceProxy.Get(b, System.Threading.CancellationToken) @6j, State: Invalidated)
aComputed: 1, ReplicaClientComputed`1(Intercepted:ICounterServiceProxy.Get(a, System.Threading.CancellationToken) @2m, State: Invalidated)
Get(a)
Get(b)
aComputed: 11, ReplicaClientComputed`1(Intercepted:ICounterServiceProxy.Get(a, System.Threading.CancellationToken) @2n, State: Consistent)
bComputed: 10, ReplicaClientComputed`1(Intercepted:ICounterServiceProxy.Get(b, System.Threading.CancellationToken) @29, State: Consistent)
```

이제 Replica 서비스가 제 역할을 수행합니다 - 기본 Compute 서비스를 완벽하게 모방합니다!

`CounterController` 메소드는 주어진 인수 세트에 대해 한 번만 호출됩니다 - 이는 일부 복제본이 존재하는 동안 Replica 서비스가 이를 사용하여 값을 업데이트하기 때문입니다. 즉, 업데이트가 WebSocket 채널을 통해 요청되고 전달됩니다.

짐작하듯이 여기에서 사용한 컨트롤러는 일반 Web API 컨트롤러입니다. Fusion없이 메서드를 호출할 수 있는지 궁금하다면 그렇습니다. 따라서 모든 Fusion 엔드포인트는 일반 Web API 엔드포인트이기도 합니다!

증거:

%[https://www.youtube.com/watch?v=jYVe5yd0xuQ&t=4173s]

이제 클라이언트 측 `LiveState<T>`가 복제 서비스를 사용하여 서버 측 Compute 서비스의 출력을 "관찰"할 수 있음을 보여 드리겠습니다. 아래 코드는 `LiveState<T>`를 보여주는 이전 부분에서 본 것과 거의 동일하지만 Computed 서비스 대신 Replica 서비스를 사용합니다.

```csharp
using var host = CreateHost();
await host.StartAsync();
WriteLine("Host started.");

var services = CreateClientServices();
var counters = services.GetRequiredService<ICounterService>();
var stateFactory = services.StateFactory();
using var state = stateFactory.NewComputed(
    new ComputedState<string>.Options() {
        UpdateDelayer = new UpdateDelayer(UICommandTracker.None, 1.0), // 1초 업데이트 지연
        EventConfigurator = state1 => {
            // 3개의 이벤트 핸들러를 연결하는 바로가기: Invalidated, Updating, Updated
            state1.AddEventHandler(StateEventKind.All,
                (s, e) => WriteLine($"{DateTime.Now}: {e}, Value: {s.Value}, Computed: {s.Computed}"));
        },
    },
    async (state, cancellationToken) =>
    {
        var counter = await counters.Get("a", cancellationToken);
        return $"counters.Get(a) -> {counter}";
    });
await state.Update(); // 상태가 최신 값을 얻도록 합니다.
await counters.Increment("a");
await Task.Delay(2000);
await counters.SetOffset(10);
await Task.Delay(2000);

await host.StopAsync();
```

출력:
```
Host started.
10/2/2020 6:27:48 AM: Updated, Value: , Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @26, State: Consistent)
10/2/2020 6:27:48 AM: Invalidated, Value: , Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @26, State: Invalidated)
10/2/2020 6:27:48 AM: Updating, Value: , Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @26, State: Invalidated)
CounterController.Get(a)
Get(a)
10/2/2020 6:27:48 AM: Updated, Value: counters.Get(a) -> 0, Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @4a, State: Consistent)
CounterController.Increment(a)
Increment(a)
10/2/2020 6:27:48 AM: Invalidated, Value: counters.Get(a) -> 0, Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @4a, State: Invalidated)
10/2/2020 6:27:49 AM: Updating, Value: counters.Get(a) -> 0, Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @4a, State: Invalidated)
Get(a)
10/2/2020 6:27:50 AM: Updated, Value: counters.Get(a) -> 1, Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @6h, State: Consistent)
CounterController.SetOffset(10)
SetOffset(10)
10/2/2020 6:27:50 AM: Invalidated, Value: counters.Get(a) -> 1, Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @6h, State: Invalidated)
10/2/2020 6:27:51 AM: Updating, Value: counters.Get(a) -> 1, Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @6h, State: Invalidated)
Get(a)
10/2/2020 6:27:51 AM: Updated, Value: counters.Get(a) -> 11, Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @ap, State: Consistent)
10/2/2020 6:27:52 AM: Invalidated, Value: counters.Get(a) -> 11, Computed: StateBoundComputed`1(FuncLiveState`1(#38338487) @ap, State: Invalidated)
```

짐작할 수 있듯이 이것은 Blazor 샘플이 실시간으로 UI를 업데이트하는 데 사용하는 논리입니다. 또한 동일한 공통 인터페이스를 구현하기 위해 Compute 서비스 및 Replica 서비스를 유사하게 만듭니다. 이것이 바로 WASM 및 Server 측 Blazor 모드에서 작동하는 동일한 UI 구성 요소를 사용할 수 있도록 하는 것입니다:

- UI 구성 요소가 서버 측에서 렌더링될 때 호스트의 `IServiceProvider`에서 `IWhateverService`의 구현으로 서버 측 Compute 서비스를 선택합니다. 모든 것이 로컬이기 때문에 복제본이 필요하지 않습니다.
- 그리고 동일한 UI 구성 요소가 클라이언트에서 렌더링될 때 클라이언트 측 IoC 컨테이너에서 `IWhateverService`로 Replica 서비스를 선택하고, 이것이 `IState<T>`가 실시간으로 업데이트되도록 하고 UI 구성 요소가 다시 렌더링되도록 합니다.

이상입니다. 이제 Fusion의 모든 주요 기능을 배웠습니다. 물론 세부 사항이 있으며 나머지 튜토리얼에서는 대부분 이에 관한 것입니다.