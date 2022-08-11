## Stl.Fusion 개요, HelloBlazorServer 샘플 분석

[Stl.Fusion](https://github.com/servicetitan/Stl.Fusion)은 실시간 업데이트를 위한 구현 코드를 1% 미만의 추가 코드로 실시간 앱을 만들 수 있도록 도와주는 라이브러리이다.

이 라이브러리는 [Servicetian](https://www.servicetitan.com/)의 [Alex Yakunin](https://github.com/alexyakunin)님의 부단한 노력에 의해 만들어졌으며 `Stl.Fusion`에 대해 이해할 수 있도록 [왜 실시간 웹앱에 Blazor와 Fusion이 필요한가요? 이야기](https://alexyakunin.github.io/Stl.Fusion.Materials/Slides/Fusion_v2/Slides.html) 프리젠테이션 자료를 제공하고 있다.

그렇다면 Stl.Fusion에서 말하는 실시간 앱이란 무엇인가?

## 분산 반응형 메모이제이션 (Distributed REActive Memoization) : DREAM

[메모이제이션](https://ko.wikipedia.org/wiki/%EB%A9%94%EB%AA%A8%EC%9D%B4%EC%A0%9C%EC%9D%B4%EC%85%98)이란 컴퓨터가 동일한 계산을 반복할 때 재계산 하지 않고 이전 계산된 메모리에 저장된 값을 대신 사용해 실행 속도를 빠르게 하는 기술이다. 

이 메모이제이션의 기능에 더해 `분산`처리와 `반응형` 기능을 더한 기술이 Stl.Fusion에서 제공하는 분산 반응형 메모이제이션이고 Stl.Fusion에서는 이를 DREAM이라고 말한다.

Stl.Fusion에서는 DREAM을 효과적으로 구현하기 위해서 관찰 대상을 종속성 그래프로 보았을 때 필요로 하는 가지만 동기화하는 기술을 사용했다. 가령 서버에서 관리하는 상태가 1천만개라면 클라이언트가 관찰해야 하는 자신의 1천개의 상태만 서버와 상태 동기화가 되는 것이다. 만약 서버와의 연결이 끊겼다 하더라도 다시 접속했을 때 새로운 관찰 대상의 상태 정보가 동기화가 되며 이는 매우 효율적으로 동작한다고 한다. 소개 페이지에는 MMORPG 게임 엔진이 수행하는 그것과 유사하게 작업을 수행한다고 한다.

또한 반응형이므로 상태가 변경되었음을 감지했을 때 연결된 분산 네트워크를 통해 상태는 `즉시` 동기화 된다. 이떄 동기화는 매우 효율적인 방식으로 이루어진다고 한다.

흥미로운 점은 상태 정보 동기화가 반응형으로 분산된 장치에 메모이제이션 된다는 점이다. Stl.Fusion을 이용하면 실시간 서비스를 매우 빠르게 제공할 수 있다는 것이다.


## Stl.Fusion 샘플 소개

Stl.Fusion의 제공 기능을 빠르게 이해하고 활용하기 위해 [HelloBlazorServer](https://github.com/servicetitan/Stl.Fusion.Samples/tree/master/src/HelloBlazorServer) 샘플을 분석할 것이다. `HelloBlazorServer` 샘플을 실행하면 다음의 화면을 볼 수 있다. Blazor의 기본 프로젝트 템플릿에 의해 생성된 구성과 유사한 것 같다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658664531387/WhIyAU6nV.png align="left")

하지만 `Counter`, `Fetch data`는 실시간성을 확인하기 위해 다른 결과를 보여준다.


### Counter

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658664685142/CEhSxFOs5.png align="left")

`https://localhost:5000/counter` 주소로 창을 두 개 열고 한쪽의 `Increment` 버튼을 눌렀을 때 실시간으로 다른 쪽의 `Count`가 증가하는 것을 볼 수 있다.


### Fetch data

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658664779556/zUMU7l7kS.png align="left")

1초마다 날씨정보가 갱신되며 왼쪽 창과 오른쪽 창이 동일하게 변경되는 것을 확인할 수 있다.


### Simple Chat

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658664849184/wnjNlo6CC.png align="left")

간단한 채팅 서비스를 `Stl.Fusion`을 이용해 구현되어 있다. 왼쪽 창에서 메시지를 보내면 오른쪽 창에서 보낸 메시지를 확인할 수 있다.


## Stl.Fusion 샘플 분석

먼저 위의 샘플을 깃허브에서 클론한 후 잘 실행되는지를 확인하자.


### ComputeService 등록

`Startup.cs`에 다음의 코드로 Fusion 및 Fusion의 ComputeService들을 등록한다.

| Startup.cs
```csharp
var fusion = services.AddFusion();
fusion.AddBlazorUIServices();
fusion.AddFusionTime(); // IFusionTime is one of built-in compute services you can use
fusion.AddComputeService<CounterService>();
fusion.AddComputeService<WeatherForecastService>();
fusion.AddComputeService<ChatService>();
fusion.AddComputeService<ChatBotService>();
This is just to make sure ChatBotService.StartAsync is called on startup
services.AddHostedService(c => c.GetRequiredService<ChatBotService>());

// Default update delay is set to min.
services.AddTransient<IUpdateDelayer>(_ => UpdateDelayer.MinDelay);
```

> `ComputeService`는 실시간 서비스에서 필요로 하는 상태 정보를 계산하는 서비스 단위이다.  `ComputeService` 구현을 통해 실시간 상태를 관리할 수 있게 된다.


### ComputeService 구현

샘플에서 가장 단순한 구조인 `CounterService`를 분석하도록 하자. 전체 소스 코드는 다음과 같다.

```csharp
public class CounterService
{
    private readonly object _lock = new();
    private int _count;
    private DateTime _changeTime = DateTime.Now;

    [ComputeMethod]
    public virtual Task<(int, DateTime)> Get()
    {
        lock (_lock) {
            return Task.FromResult((_count, _changeTime));
        }
    }

    public Task Increment()
    {
        lock (_lock) {
            ++_count;
            _changeTime = DateTime.Now;
        }
        using (Computed.Invalidate())
            Get();
        return Task.CompletedTask;
    }
}
```

`CounterService`에서 실시간 상태 정보는 `카운트 값`이며 `Get()` 메소드를 통해 그 값을 얻을 수 있다. 그런데 이 메소드는 `ComputeMethod`라는 특성으로 장식되어 있으며 이 특성을 통해 이 메소드가 한번 호출된 뒤로는 다시 호출될 때 `Get()` 메소드의 구현 코드가 실행되는 것이 아니라 `Stl.Fusion`에 의해 메모리에 캐싱된 값을 반환하여 메모이제이션을 수행한다.

하지만 `Increment()` 메소드를 호출하게 되면 `카운트 값`의 실제 값인 `_count`를 1 증가하며 캐싱된 값을 무효화 하기 위해  `Computed.Invalidate()`를 호출한다.

코드에 `lock`을 쓰인 것을 확인할 수 있는데 `ComputeService`는 싱글톤 인스턴스임을 알 수 있다.


### 계산된 값(Computed Value) 표현

이제 ComputeService를 어떻게 사용하는지를 살펴보자. 다음은 `Counter.razor` 소스 코드이다.


| Counter.razor
```html
@page "/counter"
@using System.Threading
@using Stl.Fusion.Extensions
@inherits ComputedStateComponent<string>
@inject CounterService CounterService
@inject IFusionTime Time
@inject NavigationManager Nav

@{
    var state = State.ValueOrDefault;
    var error = State.Error;
}

<h1>Counter</h1>

<div class="alert alert-primary">
    Open this page in <a href="@Nav.Uri" target="_blank">another window</a> to see it updates in sync.
</div>
@if (error != null) {
    <div class="alert alert-warning" role="alert">Update error: @error.Message</div>
}

<p>Count: @state</p>

<button class="btn btn-primary" @onclick="Increment">Increment</button>

@code {
    protected override async Task<string> ComputeState(CancellationToken cancellationToken)
    {
        var (count, changeTime) = await CounterService.Get();
        var momentsAgo = await Time.GetMomentsAgo(changeTime);
        return $"{count}, changed {momentsAgo}";
    }

    private async Task Increment()
    {
        await CounterService.Increment();
    }
}
```

Blazor의 화면은 상태가 변경될 때 반응을 해야 하므로 `Stl.Fusion`에서 제공하는 `ComputedStateComponent<TState>`를 상속 받는다.

```
@inherits ComputedStateComponent<string>
```

상태가 갱신될 때 `ComputeState()` 메소드가 호출이 된다.

```csharp
protected override async Task<string> ComputeState(CancellationToken cancellationToken)
{
    var (count, changeTime) = await CounterService.Get();
    var momentsAgo = await Time.GetMomentsAgo(changeTime);
    return $"{count}, changed {momentsAgo}";
}
```

반환된 값은 카운트 값과 변경된 시각에서 얼마나 경과 되었는지의 정보이며 이 값은

```csharp
var state = State.ValueOrDefault;
```

`state` 필드에 저장이 되고 아래의 위치에 표현된다.

```html
<p>Count: @state</p>
```


## 정리

`Stl.Fusion`에 대해 간략히 소개하고 `Stl.Fusion`을 이용하면 쉽게 실시간 앱을 구현할 수 있음을 확인하였다.

다음 시간에는 `Blazor Server`가 아닌 `Blazor Webassembly` 샘플을 통해 Proxy 모델에서도 동일하게 쉽게 DREAM을 구현할 수 있는지 살펴 보도록 하자.


