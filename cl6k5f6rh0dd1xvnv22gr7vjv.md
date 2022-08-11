## Fusion 치트 시트

## 컴퓨팅(Compute) 서비스

컴퓨팅 서비스 인터페이스:
```csharp
// IComputeService는 선택적 태깅 인터페이스일 뿐입니다.
// 그럼에도 불구하고 "구현"하는 것이 좋습니다. 이를 통해 .GetServices() 및
// .GetCommander() 와 같은 몇 가지 확장 메서드를 사용할 수 있습니다.
public interface ICartService : IComputeService
{
    // 인터페이스 메소드에 적용할 때 이 속성은 구현에 의해 "상속"됩니다.
    [ComputeMethod]
    Task<List<Order>> GetOrders(long cartId, CancellationToken cancellationToken = default);
}
```

컴퓨팅 서비스 구현:
```csharp
public class CartService : ICartService 
{
    // 메서드는 virtual 이어야 하며 Task<T>를 반환해야 합니다.
    public virtual async Task<List<Order>> GetOrders(long cartId, CancellationToken cancellationToken)
    {
        // 여기에 구현
    }
}  
```

무효화 로직을 추가하세요 (아래 using 블록 내에서 무효화된 내용에 대한 결과를 변경하는 코드)
```csharp
using (Computed.Invalidate()) {
// 이 블록 내에서 수행하는 모든 계산 메서드 호출은 호출을 무효화합니다.
// 실제 메서드 코드를 실행하는 대신 이 호출의 결과를 무효화합니다.
// 항상 동기식으로 완료되며 여기에서 CancellationToken 대신 "default"를 전달할 수 있습니다.
    _ = GetOrders(cartId, default);
}
```

컴퓨팅 서비스 등록:
```csharp
fusion = services.AddFusion(); // services는 IServiceCollection
fusion.AddComputeService<IOrderService, OrderService>();
```


## 복제(Replica) 서비스

컨트롤러 추가:
```csharp
[Route("api/[controller]/[action]")]
[ApiController, JsonifyErrors, UseDefaultSession]
public class CartController : ControllerBase, ICartService
{
    private readonly ICartService _cartService;
    private readonly ICommander _commander;

    public CartController(ICartService service, ICommander commander) 
    {
        _service = service;
        _commander = commander;
    }    

    [HttpGet, Publish]
    public Task<List<Order>> GetOrders(long cartId, CancellationToken cancellationToken)
        => _service.GetOrders(cartId, cancellationToken);
}
```

클라이언트 정의 추가:
```csharp
[BasePath("cart")]
public interface ICartClientDef
{
    [Get(nameof(GetOrders))]
    Task<List<Order>> GetOrders(long cartId, CancellationToken cancellationToken);
}
```

Fusion 클라이언트 구성(클라이언트 측 `IServiceProvider`를 구성하는 코드에서 한 번만 수행해야 함):
```csharp
var baseUri = new Uri("http://localhost:5005");
var apiBaseUri = new Uri($"{baseUri}api/");

var fusion = services.AddFusion();
fusion.AddRestEaseClient(
    client => {
        client.ConfigureWebSocketChannel(_ => new() { BaseUri = baseUri });
        client.ConfigureHttpClient((_, name, o) => {
            var isFusionClient = (name ?? "").StartsWith("Stl.Fusion");
            var clientBaseUri = isFusionClient ? baseUri : apiBaseUri;
            o.HttpClientActions.Add(httpClient => httpClient.BaseAddress = clientBaseUri);
        });
    });
```

복제 서비스 등록:
```csharp
// "var fusionClient = ..." 이후
fusionClient.AddReplicaService<ITodoService, ITodoClientDef>();
```

복제 서비스 사용:
```csharp
// 기존과 동일하게 그냥 호출하면 됩니다.
// 동일한 인수를 사용하여 이전 호출과 동일한 결과를 생성할 것으로 
// 예상되는 모든 호출은 로컬로 캐시된 IComputed를 통해 해결됩니다.
// IComputed가 서버에서 무효화되면 Fusion은 모든 클라이언트에서
// 해당 복제본을 무효화합니다.
```


## 명령 처리자(Commander)

명령 유형 선언:
```csharp
// record를 사용할 필요는 없지만 계산 방법의 명령 및 출력에 변경 불가능한 유형을 사용하는 것이 좋습니다.
public record UpdateCartCommand(long CartId, Dictionary<long, long?> Updates) 
    : ICommand<Unit> // 단위는 명령의 반환 유형입니다. 다른 것을 사용할 수 있습니다
{
    // 호환성: Newtonsoft.Json 은 record를 역직렬화하려면 이 생성자가 필요합니다.
    public UpdateCartCommand() : this(0, null!) { }
}
```

컴퓨팅 서비스 인터페이스에 명령 처리기를 추가:
```csharp
public interface ICartService : IComputeService
{
    // ...
    [CommandHandler] // 이 속성은 또한 impl에 의해 "상속"됩니다.
    Task<Unit> UpdateCart(UpdateCartCommand command, CancellationToken cancellationToken = default);
```

명령 처리기 구현 추가:
```csharp
public class CartService : ICartService 
{
    // virtual이면서 ICommand<T>에 대해 Task<T>를 반환해야 함;
    // 명령은 첫 번째 인수여야 합니다. 다른 인수는 직접 전달되는 CancellationToken을 제외하고 DI 컨테이너에서 확인됩니다.
    public virtual Task<Unit> UpdateCart(UpdateCartCommand command, CancellationToken cancellationToken) 
    {
        if (Computed.IsInvalidating()) {
            // 여기에 이 명령에 대한 무효화 로직을 작성하세요.
            //
            // Fusion에 의해 등록된 명령 처리기 세트는 "정상" 로직이
            // 성공적으로 완료하면 무효화 블록 내에서 이 처리기를 "재시도" 합니다.
            // 또한 다중 호스트 무효화를 사용하는 경우
            // 클러스터의 모든 노드에서 이 블록을 실행합니다.
            return default;
        }

        // 명령어 처리기 코드는 여기에 있습니다.
    }
```

명령어 처리기 등록:
```csharp
// 컴퓨팅 서비스 내에서 선언된 핸들러에는 아무 것도 필요하지 않습니다.
```


## 복제(Replica) 서비스를 통해 클라이언트에 명령 노출

명령에 대한 컨트롤러 메서드를 추가:
```csharp
public class CartController : ControllerBase, ICartService
{
    // 항상 [HttpPost] 과 [FromBody] 를 사용
    [HttpPost]
    public Task<Unit> UpdateCart([FromBody] UpdateCartCommand command, CancellationToken cancellationToken)
        // 여기에서 해당 서비스 메서드를 직접 호출할 수도 있지만
        // CommanderOptions.AllowDirectCommandHandlerCalls = false를 사용하는 경우
        // 이 "스타일" 명령 호출만 작동합니다.
        // 참고로 둘은 동일하게 작동합니다. 즉, 명령 처리기가 직접 호출될 때 호출은
        // 여전히 ICommander를 통해 라우팅됩니다.
        => _commander.Call(command, cancellationToken);
}
```

클라이언트 정의 인터페이스에 명령 처리기를 추가:
```csharp
public interface ICartClientDef 
{
    // 항상 [Post] 와 [Body] 를 사용
    [Post(nameof(UpdateCart))]
    Task<Unit> UpdateCart([Body] UpdateCartCommand command, CancellationToken cancellationToken);
```

클라이언트 측 명령 처리기 등록:
```csharp
// 복제 서비스 내에서 선언된 핸들러에는 아무것도 필요하지 않습니다.
``


## `IComputed` 동작

캡처:
```chsarp
var computed = await Computed.Capture(ct => service.ComputeMethod(args, ct), cancellationToken);
```

`IComputed`가 여전히 일관되는지 확인:
```csharp
if (computed.IsConsistent()) {
    // ...
}
```

무효화 대기:
```csharp
// 항상 여기에 CancellationToken을 전달하십시오. 그렇지 않으면
// 이벤트 처리기 등록 수가 증가하여 메모리 누수가 발생하게 됩니다.
// 물론 무효화될 것이지만, 그런 일이 일어나지 않는다면 어떻게 될까요?
await computed.WhenInvalidated(cancellationToken);
// 또는
computed.Invalidated += c => Console.WriteLine("Invalidated!");
```

계속됩니다.
