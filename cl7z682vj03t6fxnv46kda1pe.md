## Fusion 튜토리얼 - 1부: 컴퓨팅 서비스

이 부분을 다룬 동영상:

%[https://youtu.be/G-MIdfDP3gI]

Fusion은 실시간 서비스를 구축할 수 있는 3가지 주요 추상화를 제공합니다.

1. 계산된 값(Computed Value) – 유형 `T`의 계산 결과를 설명하는 개체로, 이 결과가 무효화될 때(대부분 실제와 일치하지 않음) 이를 사용자에게 알릴 수도 있습니다. 이러한 값은 항상 `IComputed<T>`를 구현합니다. 가장 유용한 구현이 이미 있기 때문에 이 인터페이스를 구현할 필요가 없습니다.
1. 컴퓨팅 서비스(Compute Service) – 해당 메서드 출력의 종속성을 자동으로 캡처하고 `IComputed<T>` 인스턴스로 투명하게 "지원"하는 서비스로 이러한 출력이 무효화될 때(실제와 일치하지 않을 때) 누구나 알 수 있게 합니다. 컴퓨팅 서비스는 사용자가 작성해야 합니다.
1. 상태(State) – 단일 `IComputed<T>`를 "추적"하는 추상화, 즉 가장 최신 버전을 지속적으로 참조합니다. 다시 말하지만 일반적으로 고유한 `IState<T>`를 구현할 필요가 없습니다. Fusion은 가장 유용한 3가지 특징을 제공합니다.

컴퓨팅 서비스를 주로 다루므로 이 부분부터 시작하겠습니다.

하지만 먼저 컴퓨팅 서비스를 호스팅하는 `IServiceProvider`를 생성할 수 있는 도우미 메서드를 생성해 보겠습니다.

```csharp
public static IServiceProvider CreateServices()
{
    var services = new ServiceCollection();
    var fusion = services.AddFusion();
    fusion.AddComputeService<CounterService>();
    fusion.AddComputeService<CounterSumService>(); // We'll be using it later
    fusion.AddComputeService<HelloService>();      // We'll be using it later
    return services.BuildServiceProvider();
}
```

이제 첫 번째 컴퓨팅 서비스를 선언할 준비가 되었습니다.

```csharp
public class CounterService
{
    private readonly ConcurrentDictionary<string, int> _counters = new ConcurrentDictionary<string, int>();

    [ComputeMethod]
    public virtual async Task<int> Get(string key)
    {
        WriteLine($"{nameof(Get)}({key})");
        return _counters.TryGetValue(key, out var value) ? value : 0;
    }

    public void Increment(string key)
    {
        WriteLine($"{nameof(Increment)}({key})");
        _counters.AddOrUpdate(key, k => 1, (k, v) => v + 1);
        using (Computed.Invalidate())
            _ = Get(key);
    }
}
```

지금은 `Get`이 비동기 메서드로 선언되었다는 사실을 무시하십시오. 비록 그것이 진정한 비동기가 아니더라도 - 나중에 그것이 합리적인 이유를 설명할 것입니다.

`CounterService`를 사용합시다.

```csharp
var counters = CreateServices().GetRequiredService<CounterService>();
WriteLine(await counters.Get("a"));
WriteLine(await counters.Get("b"));
```

출력은 다음과 같아야 합니다.

```
Get(a)
0
Get(b)
0
```

평범해 보이죠? 그러나 이것은 어떻습니까?

```csharp
var counters = CreateServices().GetRequiredService<CounterService>();
WriteLine(await counters.Get("a"));
WriteLine(await counters.Get("a"));
```

이제 출력이 이상해 보입니다.

```
Get(a)
0
0
```

그렇다면 여기서 "Get(a)"가 두 번 인쇄되지 않은 이유는 무엇입니까? 정답:

- 모든 계산 방법이 출력을 자동으로 캐시한다고 생각할 수 있습니다.
- 캐시 키는 `(MethodInfo, this, argument1, argument2, ...)`
- 캐시된 값은 메서드 출력입니다.
- 항목은 동일한 인수 세트를 사용하여 동일한 서비스의 동일한 메소드에 대해 `Computed.Invalidate(() => ...)`를 호출하면 만료됩니다.

어떻게 작동하는지 봅시다.

```csharp
var counters = CreateServices().GetRequiredService<CounterService>();
WriteLine(await counters.Get("a"));
counters.Increment("a");
WriteLine(await counters.Get("a"));
```

출력:
```
Get(a)
0
Increment(a)
Get(a)
1
```

위의 `CounterService.Increment` 소스 코드를 확인하십시오. `Computed.Invalidate`를 호출하여 항목을 제거합니다. 이것은 이전에 첫 번째 호출에 대해서만 인쇄되었지만 이 예제에서 "Get(a)"가 두 번 인쇄되는 이유를 설명합니다.


## 종속성

이제 다른 컴퓨팅 서비스를 추가해 보겠습니다.

```csharp
public class CounterSumService
{
    public CounterService Counters { get; }

    public CounterSumService(CounterService counters) => Counters = counters;

    [ComputeMethod]
    public virtual async Task<int> Sum(string key1, string key2)
    {
        WriteLine($"{nameof(Sum)}({key1}, {key2})");
        return await Counters.Get(key1) + await Counters.Get(key2);
    }
}
```

그리고 사용해봅시다.
```csharp
var services = CreateServices();
var counterSum = services.GetRequiredService<CounterSumService>();
WriteLine(await counterSum.Sum("a", "b"));
WriteLine(await counterSum.Sum("a", "b"));
```

출력:
```
Sum(a, b)
Get(a)
Get(b)
0
0
```

이러한 서비스가 현재 어떻게 작동하는지 알고 있다고 가정하면 이것이 정확히 예상한 것입니다.

또다른 예시:
```chsarp
var services = CreateServices();
var counterSum = services.GetRequiredService<CounterSumService>();
WriteLine("Nothing is cached (yet):");
WriteLine(await counterSum.Sum("a", "b"));
WriteLine("Only Get(a) and Get(b) outputs are cached:");
WriteLine(await counterSum.Sum("b", "a"));
WriteLine("Everything is cached:");
WriteLine(await counterSum.Sum("a", "b"));
```

출력:
```
Nothing is cached (yet):
Sum(a, b)
Get(a)
Get(b)
0
Only Get(a) and Get(b) results are cached:
Sum(b, a)
0
Everything is cached:
0
```

다시 말하지만 예상치 못한 것은 없습니다. 결과는 여전히 캐시되지만 키는 인수 순서에 민감하므로 `("a", "b")` 및 `("b", "a")` 항목이 다릅니다.

그러나 이것은 어떨까요?

```csharp
var services = CreateServices();
var counters = services.GetRequiredService<CounterService>();
var counterSum = services.GetRequiredService<CounterSumService>();
WriteLine(await counterSum.Sum("a", "b"));
counters.Increment("a");
WriteLine(await counterSum.Sum("a", "b"));
```

출력:
```
Sum(a, b)
Get(a)
Get(b)
0
Increment(a)
Sum(a, b)
Get(a)
1
```

이것은 매우 이례적인 일이죠? 어쨌든 `Sum("a", "b")`은 증가로 인해 무효화되었기 때문에 `Get("a")` 결과를 먼저 새로 고쳐야 한다는 것을 알아냈습니다. 하지만 어떻게?

실제로 모든 컴퓨팅 메서드는 캐시된 출력을 얻거나 실행할 계산을 "지원"하는 새 `IComputed<T>` 인스턴스를 빌드하고 계산이 실행되는 동안 이 인스턴스는 `Computed.GetCurrent()` 메서드를 통해 계속 사용할 수 있습니다. 따라서 계산 중에 호출된 다른 계산 메서드는 현재 계산된 인스턴스의 종속성으로 고유한 숨겨진 출력(`IComputed<T>`도 포함)을 참여시킬 기회를 얻습니다.

실제 프로세스는 아직 예상하지 못한 시나리오를 설명하기 때문에 조금 더 복잡합니다.

- 재귀 및 여러 수준의 컴퓨팅 메서드 호출이 완전히 지원됩니다.
- 일부 결과는 계산 중에 바로 무효화될 수 있습니다.
- 각각의 고유한 결과에 대해 하나 이상의 계산이 주어진 순간에 실행되어서는 안 됩니다.

이 섹션을 닫기 위해 마지막 속성을 자세히 살펴보겠습니다.


## 동시 평가

Fusion이 동시성을 처리하는 방법을 테스트하는 간단한 서비스를 만들어 보겠습니다.

```csharp
public class HelloService
{
    [ComputeMethod]
    public virtual async Task<string> Hello(string name)
    {
        WriteLine($"+ {nameof(Hello)}({name})");
        await Task.Delay(1000);
        WriteLine($"- {nameof(Hello)}({name})");
        return $"Hello, {name}!";
    }
}
```

보시다시피 `Hello` 메서드는 형식이 지정된 "Hello, X!"를 반환하지만 1초 지연됩니다. 동시에 실행해 보겠습니다.

```csharp
var hello = CreateServices().GetRequiredService<HelloService>();
var t1 = Task.Run(() => hello.Hello("Alice"));
var t2 = Task.Run(() => hello.Hello("Bob"));
var t3 = Task.Run(() => hello.Hello("Bob"));
var t4 = Task.Run(() => hello.Hello("Alice"));
await Task.WhenAll(t1, t2, t3, t4);
WriteLine(t1.Result);
WriteLine(t2.Result);
WriteLine(t3.Result);
WriteLine(t4.Result);
```

출력:
```
+ Hello(Bob)
+ Hello(Alice)
- Hello(Bob)
- Hello(Alice)
Hello, Alice!
Hello, Bob!
Hello, Bob!
Hello, Alice!
```

보시다시피 4개의 값이 모두 계산되었지만 `Hello` 평가는 2개(고유한 인수에 대해서만) 뿐 아니라 이 두 평가가 동시에 실행되고 있었습니다.

이것은 예상된 동작입니다. 처음에는 아무 것도 캐시되지 않았지만 예를 들어 다음과 같이 둘 이상의 계산을 실행할 이유가 없습니다. "Bob" 인수는 동시에 모두 동일한 결과를 생성해야 하기 때문입니다. 이것이 바로 Fusion이 보장하는 것입니다.

그리고 반대로 `Hello("Alice")` 계산이 `Hello("Bob")`와 동시에 실행되도록 하는 것이 완전히 합리적입니다. 왜냐하면 서로 다른 출력을 생성할 수 있고 동시에 실행되는 경우 `HelloService`가 이를 지원하도록 설계되었기 때문입니다.

전반적으로 Fusion의 거의 모든 것이 동시 호출을 지원합니다.

- 컴퓨팅 서비스는 동시성을 지원하는 싱글톤이어야 합니다.
- 모든` IComputed<T>` 구현은 완전히 동시적입니다.
- 모든 `IState<T>`도 마찬가지
- 예외는 대부분 서비스 등록 단계에서 사용되어야 하는 `XxxOptions` 및 메서드와 같은 형식과 동시에 사용되지 않아야 하는 형식(예: 모든 Blazor 구성 요소 - `StatefulComponentBase<TState>` 및 해당 하위 항목)입니다.