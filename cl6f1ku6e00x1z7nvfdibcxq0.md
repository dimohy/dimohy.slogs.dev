## Fusion 개요

> 본 글은 [Stl.Fusion](https://github.com/servicetitan/Stl.Fusion)의 [Fusion Overview](https://github.com/servicetitan/Stl.Fusion/blob/master/docs/Overview.md) 문서를 번역한 것입니다.

> 원문 링크는 번역을 완료하면 번역한 링크로 교체할 예정입니다.


## "실시간 사용자 인터페이스"란 무엇입니까?

> 사용자가 조치를 취하지 않아도 업데이트되는 항상 최신 콘텐츠를 표시하는 UI 입니다.

먼저 관련 없어 보이는 문제를 먼저 살펴보겠습니다. **실시간 항목 무효화를 위한 캐싱** 특정 계산에 대해 이 문제를 해결하는 의사 코드는 다음과 같습니다.

```csharp
var key = (nameof(ComputeValueFor), arg1, arg2, arg3, ...);
for (;;) {
    var observableValue = await ComputeAsync(arg1, arg2, arg2, ...);
    await cache.SetAsync(key, observableValue.Value);
    await observableValue.ChangedAsync();
    await cache.Evict(key);
}
```

이제 `observableValue`가 HTML 또는 전체 UI 또는 해당 조각을 정의하는 다른 마크업이라고 생각하세요. 변경될 때 (또는 변경될 가능성이 있는 경우) 유사하게 신호를 보낼 수 있는 실시간 UI 업데이트 로직은 다음과 같을 수 있습니다.

```csharp
for (;;) {
    var observableMarkup = await ComputeMarkupAsync(arg1, arg2, arg2, ...);
    Render(observableMarkup);
    await observableValue.ChangedAsync();
}
```

여기에는 `비동기` 함수와 같이 실제 출력에 래퍼를 반환하여 출력(동일한 인수의 집합)이 변경될 때 신호를 보내는 추상화를 통해 잘 해결할 수 있는 광범위한 문제가 있습니다.

따라서 이것을 `비동기` 함수와 비교하면 차이점은 다음과 같습니다.

- 비동기 함수는 `Task<T>`를 반환하여 계산 완료를 비동기로 기다릴 수 있습니다. 즉, `Completed` 이벤트를 구독합니다. 이 이벤트가 발생하면 작업의 `Result` 및 `Exception` 속성에 엑세스할 수 있습니다.
- 우리의 새 함수는 `IComputed<T>`를 반환해야 합니다. 이는 유사하게 `Value` 및 `Error`에 엑세스 하도록 허용하지만 (이를 위해 기다릴 필요는 없음) 동일한 함수에 대한 호출이 `IComputed<T>`를 생성하는 순간을 비동기로 기다릴 수도 있습니다. 이를 `(Value, Error)` 쌍이 현재 있는 쌍과 다를 수 있습니다.

궁극적으로 이것이 바로 Fusion이 제공하는 것입니다. `IComputed<T>`에 대한 추상화와 이러한 [계산된 값]을 반환하는 함수를 작성하는 방법입니다.

놀랍게도 Fusion은 이러한 `IComputed<T>` "상자"를 "래핑" 및 "풀기"할 필요가 없습니다. 곧 [Compute 서비스](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part01.md)의 메서드 반환 유형이 동일하게 유지되더라도 배후에서 `IComputed<T>` 상자에 의해 암시적으로 "지원"된다는 것을 알게 될 것입니다. "명시적인 것이 암시적인 것보다 낫다"는 원칙에 위반하는 것 처럼 보이지만, 예외가 단지 그 규칙이 옳다는 것을 증명하는 경우라는 것을 여러분이 알아주길 바랍니다. 왜냐하면 이것은 가독성에 대한 것이기 때문이다.

- 거의 모든 [Compute 서비스](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part01.md) 메서드는 비동기식이어야 하며 모든 곳에서 `Task<Whatever>` 대신 `Task<IComputed<Whatever>>`를 보는 것은 고통스럽습니다.
- 그러나 `var value = (await GetSomethingAsync(...).ConfigureAwait(false).Value)`를 사용해서 이러한 출력의 래핑을 해제하는 것은 일반적으로 작성하는 모든 메서드에 대해 둘 이상의 호출 사이트가 있다고 가정할 때 훨씬 더 고통스럽습니다.

그리고 더 중요한 것은 Fusion은 [Compute 서비스](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part01.md)에서 생성된 [계산된 값] 간의 종속성을 자동으로 추적하므로 이들 중 하나가 무효화되면 이에 종속된 모든 값도 무효화 합니다. 이 기능은 엄청난 시간을 절약해 줍니다. 사실 **Fusion에서는 외부 (비 Fusion 기반) 데이터 소스에서 생성된 값만 수동으로 무효화해야 하기 때문**입니다. 나머지는 자동으로 무효화됩니다.

그러나 더 깊이 파고 들기 전에 이 모든 것이 최종 일관성과 어떻게 관련되어 있는지, 보다 정확하게는 이 접근 방식이 궁극적으로 일관성 있는 시스템의 가장 중요한 속성을 어떻게 극적으로 향상 시키는지 알아보겠습니다.


## 캐싱, 무효화 및 최종 일관성

일관성과 캐싱이 무엇인지 간략하게 요약하면 다음과 같습니다.

1. **"일관성"**은 관찰되는 값이 그에 대해 정의된 관계 규칙을 충족하는 상태입니다. 관계는 값에 대한 술어(또는 주장)의 집합으로 정의되지만 가장 일반적인 관계는 `x == fn(a, b)`입니다. 즉, `x`는 `(a, b)`에 적용된 `fn` 함수의 출력입니다. 다시 말해, **기능적 관계**입니다.

1. 일관성은 **부분적**일 수 있습니다. 예로 `(x, a, b)`이 그것에 대해 정의된 모든 관계에 대해 일관된 상태에 있다고 말할 수 있지만 일부 다른 값 또는 관계에서는 그렇지 않습니다. 간단히 말해서 "일관성"은 항상 그 범위의 일부를 의미하며 이 범위는 단일 값 또는 일관성 규칙만큼 좁을 수 있습니다.

1. 일관성은 **궁극적**일 수 있습니다. 이것은 시스템을 "손대지 않은" 상태로 두면 (즉, 새로운 변경 사항을 도입하지 않을 경우) 결국 (즉, 미래의 어떤 시점에서) 일관된 상태를 발견할 수 있다는 멋진 표현입니다.

1. *오작동하지 않는 모든 시스템은 최소한 결국 일관성이 있습니다**. 결국 일관성보다 나쁘다는 것은 "결코 회복하지 못할 실패에 취약하다는 것"과 거의 동일합니다.

1. **"캐싱"**은 "계산 결과를 어딘가에 저장하고 실제 계산을 다시 실행하지 않고 재사용합니다"라는 멋진 표현일 뿐입니다.
   - 일반적으로 "캐싱"은 일부 내장 무효화 정책(LRU, 타이머 기반 만료 등)이 있는 고성능 키-값 저장소의 사용을 의미하지만...
   - "캐싱"을 넓게 정의하면 CPU 레지스터에 데이터를 저장하는 것조차 캐싱의 한 예입니다. 더 나아가 "캐싱"을 대부분 이런 의미에서 사용할 것입니다. 즉, "다시 계산을 실행하지 않고 이전에 저장된 계산 결과를 재사용"하는 의미입니다.

이제 우리가 A와 B라는 두 개의 시스템을 가지고 있다고 가정해 봅시다. 그리고 두 시스템 모두 결국 일치합니다. 개발자 입장에도 똑같이 좋은가요? 아니요. 결국 일관된 시스템을 구별하는 가장 큰 요인은 시스템이 일관된(또는 일관되지 않은) 상태로 발견될 확률입니다. 이를 ~ 업데이트 이후의 임의 트랜잭션에 대한 예상 불일치 기간, 즉 임의 업데이트 후 이 시스템의 임의 사용자가 불일치를 캡처하는 일부 데이터를 얻을 수 있는 기간으로 정의할 수도 있습니다.
- 이 기간이 짧으면 이러한 시스템은 항상 일관된 시스템과 매우 유사합니다. 대부분의 읽기는 일관성이 있으므로 개발자는 읽기의 불일치를 낙관적으로 무시하고 업데이트를 적용할 때만 이를 확인할 수 있습니다.
- 반대로 큰 불일치 기간은 매우 고통스럽습니다. 데이터를 읽는 코드를 포함하여 모든 곳에서 이를 고려해야 합니다.

> 위의 "작은" 기간과 "큰 불일치" 기간은 상대적인 용어입니다. 즉, 불일치를 포착하는 트랜잭션의 백분율에만 관심이 있습니다. 따라서 앱 사용자가 사람인 경우 "작은"은 약 1밀리초 정도지만 로봇(거래 등)용 API를 구축하는 경우 이 "작은"은 마이크로초 미만일 수 있습니다.

간단히 말해서 우리는 작은 불일치 기간을 원합니다. 하지만 잠시만요... 대부분의 캐시가 제공하는 것을 살펴보면 이를 제어하는 두 가지 방법이 있습니다.
- 항목 만료 시간 설정
- 항목을 수동으로 제거

첫 번째 옵션은 코딩하기 쉽지만 큰 단점이 있습니다.
- 만료 시간이 짧으면 불일치 기간이 줄어들지만 동시에 캐시 적중률이 감소합니다.
- 반대로 만료 시간이 길면 캐시 적중률이 좋아질 수 있지만 불일치 기간이 훨씬 길어지면 큰 문제가 될 수 있습니다.

![](https://github.com/servicetitan/Stl.Fusion/raw/master/docs/img/InconsistencyPeriod.gif)

솔루션 계획은 다음과 같습니다.
- 만약 우리가 `x == f(...)` 스타일 일관성 규칙에만 관심이 있다고 가정한다면 우리는 특정 함수의 출력이 변경될 때 가능한 빨리 우리에게 알려줄 무언가가 필요합니다.
- 이 조각이 있으면 두 가지 문제를 모두 해결할 수 있습니다.
   - 캐시 불일치
   - 실시간 UI 업데이트


## 구현

변경 사항을 정확하게 감지하는 것은 함수 자체를 계산하는 것만큼 비용이 많이 듭니다. 그러나 작은 퍼센트의 거짓 양성도 괜찮다면 우리는 함수의 입력이 변경되면 출력이 항상 변경된다고 가정할 수 있습니다. 모든 함수가 [임의의 오라클](https://en.wikipedia.org/wiki/Random_oracle) 또는 완벽한 해시 함수라는 가정과 유사합니다.

이 규칙을 사용하는 방법을 곧 설명하겠습니다. 하지만 지금은 계산하는 모든 항목에 대해 변경 알림을 받는 데 필요한 API에 대해서도 생각해 보겠습니다.

이것이 "원래" 함수의 코드라고 가정해 보겠습니다.

```csharp
DateTime GetCurrentTimeWithOffset(TimeSpan offset) {
    var time = GetCurrentTime()
    return time + offset;
}  
```

보시다시피 이 함수는 인수와 외부 상태(GetCurrentTime())를 모두 사용하여 출력을 생성합니다. 따라서 먼저 출력이 무효화되면 알림을 받을 수 있는 무언가를 반환하도록 만들고 싶습니다.

```csharp
IComputed<DateTime> GetCurrentTimeWithOffset(TimeSpan offset) {
    var time = GetCurrentTime()
    return Computed.New(time + offset); // Notice we return a different type now!
}  
```

여기서 반환하는 [IComputed<T>](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part02.md)는 다음과 같이 정의됩니다.

```csharp
interface IComputed<T> {
    ConsistencyState ConsistencyState { get; } // Computing -> Consistent -> Invalidated
    T Value { get; }
    Action Invalidated; // Event, triggered just once on invalidation

    void Invalidate();
}
```

멋진. 이제 보다시피 이 메서드는 다른 인수에 대해 `IComputed<T>`의 다른(사실 새로운) 인스턴스를 반환하므로 함수 입력의 이 부분은 출력에 대해 항상 일정합니다. (주어진 인수 집합에 대한) 출력을 변경할 수 있는 유일한 것은 `GetCurrentTime()`의 출력입니다.

그러나 이 함수가 우리의 API도 지원한다고 가정하면 어떨까요? 즉, 다음과 같은 서명이 있습니다.

```csharp
IComputed<DateTime> GetCurrentTime();
```

이 경우 `GetCurrentTimeWithOffset()`의 작업 버전은 다음과 같을 수 있습니다.

```csharp
IComputed<DateTime> GetCurrentTimeWithOffset(TimeSpan offset) {
    var cTime = GetCurrentTime()
    var result = Computed.New(cTime.Value + offset);
    cTime.Invalidated += () => result.Invalidate();
    return result;
}  
```

> [knockout.js](https://knockoutjs.com/) 또는 [MobX](https://mobx.js.org/README.html)를 사용한 적이 있다면 이것은 다른 옵저버블을 사용하는 계산된 옵저버블을 생성하고자 할 때 작성하는 코드와 거의 동일합니다. 단, 종속성 무효화를 수동으로 구독할 필요가 없습니다(이것은 자동으로 수행됨)

보시다시피 이제 `GetCurrentTime()`의 출력이 무효화되면 `GetCurrentTimeWithOffset()`의 모든 출력이 자동으로 무효화되므로 `GetCurrentTimeWithOffset()`에 대한 무효화 문제를 완전히 해결했습니다!

그러나 `GetCurrentTime()`의 출력을 무효화하는 것은 무엇입니까? 실제로 두 가지 옵션이 있습니다.
1. 이 함수의 출력은 `GetCurrentTimeWithOffset()` 출력이 무효화된 것과 같은 방식으로 무효화됩니다. 즉, 모든 종속성의 무효화 이벤트에 유사하게 구독하고 이들 중 하나가 신호를 보내면 스스로를 무효화합니다.
1. 또는 "수동으로" 무효화하는 다른 코드가 있습니다. 이는 소비하는 값 중 하나가 `IComputed<T>`를 지원하지 않는 경우에 항상 발생합니다.

지금은 두 번째 범주의 기능은 무시합시다. 첫 번째 범주의 모든 일반 함수를 코드를 변경하지 않고 `IComputed<T>`를 반환하는 함수로 바꿀 수 있습니까? 그렇습니다.

```csharp
// The original code in TimeService class
virtual DateTime GetCurrentTimeWithOffset(TimeSpan offset) {
    var time = GetCurrentTime()
    return time + offset;
}

// ...

// The "decorated" version of this function generated in a 
// descendant of TimeService class:                     
override DateTime GetCurrentTimeWithOffset(TimeSpan offset) {
    // Below is a GROSS SIMPLIFICATION of what really happens, I
    // provide it here mainly to explain all the high-level actions
    var dependant = Computed.GetCurrent(); // Relies on AsyncLocal<T>
    try {
        // 1. Trying to pull cached value w/o locking;
        //    the real cacheKey is, of course, more complex.
        var cacheKey = (object) (this, nameof(GetCurrentTimeWithOffset), offset);
        if (ComputedRegistry.TryGet(cacheKey, out var result))
            return result;
        
        // 2. Retrying the same with async lock to make sure 
        //    we never recompute the same result twice
        using var _ = await LockAsync(cacheKey);
        if (ComputedRegistry.TryGet(cacheKey, out result))
            return result;

        // 3. Nothing is cached, so we have to compute the result
        result = Computed.New();
        using var _ = Computed.SetCurrent(result);
        try {
            result.SetValue(base.GetCurrentTimeWithOffset(offset));
        }
        catch (Exception e) {
            result.SetError(e);
        }
        ComputedRegistry.Set(cacheKey, result);
        return result.Value; // Re-throws an error if SetError was called 
    }
    finally {
        // Let's setup a dependent-dependency link; again,
        // the real logic is very different from this.
        if (dependant != null)
            result.Invalidated += () => dependant.Invalidate();
    }
}  
```

보시다시피 원본에 대한 유일한 변경 사항은 프록시 유형이 기본 메서드의 본문을 변경하지 않고 이 메서드를 재정의하고 원하는 동작을 구현할 수 있도록 하는 "가상" 키워드 뿐입니다.

> 명확성을 위해: 실제 Fusion 프록시 코드는 다음과 같은 요인들로 인해 훨씬 더 복잡합니다.
> - 캐싱 로직(특히 키 계산 및 비교 관련)은 더 복잡합닏.
> - 다른 무효화 구독 모델을 사용합니다. 위에 표시된 것은 GC 친화적이지 않습니다. `dependency.Invalidated` 처리기는 종속 인스턴스를 참조하는 클로저를 참조하므로 일부 하위 수준 종속성이 살아 있는 동안(GC 루트에서 도달 가능) 종속 항목의 전체 하위 트리도 힙에 남아 있습니다. 분명히 상당히 나쁩니다.
> - 완전한 솔루션이 작동하려면 몇 가지 다른 부분이 필요합니다. 수동 무효화, 메소드 인수에 대한 더 빠르고 사용자 정의 가능한 동등 비교(캐시에 있는 것과 비교해야 함), `CancellationToken` 처리, ...
> 그러나 전반적으로 개념적 수준에서 매우 유사한 코드입니다.

이제 두 번째 부분으로 돌아가 보겠습니다. `IComputed<T>`를 반환하는 다른 메서드를 호출하지 않아 자동으로 무효화되지 않는 메서드입니다. 어떻게 무효화합니까?

글쎄요, 이걸 수동으로 해야 할 것 같습니다. 긍정적인 측면에서 실제 앱의 논리에 대해 생각해 봅시다.
- 80%는 고급 논리입니다. 특히 클라이언트 측과 대부분의 서버 측에서 데이터를 얻기 위해  동일한 앱에서 다른 것을 호출합니다. 특히 대부분의 컨트롤러와 서비스 계측이 그런 논리입니다.
- 로직의 20%만 일부 타사 코드 똔느 API(예: SQL 데이터 공급자 쿼리)를 호출하여 데이터를 가져올 수 있습니다. 이것은 수동 무효화를 지원해야 하는 코드입니다.

곧 여러분은 이 20%의 사례를 다루는 것이 그리 어렵지 않다는 것을 알게 될 것입니다. 하지만 먼저, 몇 가지를 봅시다...


### 실제 Fusion 코드

```csharp
public class TimeService
{
    ...

    [ComputeMethod]
    public virtual async Task<DateTime> GetTimeWithOffsetAsync(TimeSpan offset)
    {
        var time = await GetTimeAsync(); // Yes, it supports async calls too
        return time + offset;
    }
}
```

보시다시피, 본 코드와 거의 동일하지만 한 가지 차이점이 있습니다.
- 이 메서드는 `[ComputeMethod]`로 장식되어 있습니다. 이 속성은 주로 일부 안전 검사를 활성화하는 데 필요하지만 이 메서드를 지원하는 `IComputed<T>` 인스턴스에 대한 옵션을 구성할 수도 있습니다.

다른 의미에서는 앞에서 설명한 것과 거의 비슷하게 작동합니다.

수동 무효화로 돌아갑시다. 아래 코드는 Fusion 샘플의 `ChatService.cs`에서 가져온 것입니다. `CancellationToken` 및 모든 `.ConfigureAwait(false)` 호출과 관련된 모든 것은 가독성을 위해 제거되었지만(다른 .NET 비동기 논리에서와 동일한 상용구 코드) 나머지는 건드리지 않았습니다.

```csharp
// Notice this is a regular method, not a compute service method
public async Task<ChatUser> CreateUserAsync(string name)
{
    // The real-life code should do this a bit differently
    using var dbContext = CreateDbContext();

    // That's the code you'd see normally here
    var userEntry = dbContext.Users.Add(new ChatUser() {
        Name = name
    });
    await dbContext.SaveChangesAsync();
    var user = userEntry.Entity;

    // And that's the extra logic performing invalidations
    Computed.Invalidate(() => GetUserAsync(user.Id));
    Computed.Invalidate(() => GetUserCountAsync());
    return user;
}
```

보시다시피 매우 간단합니다. `Computed.Invalidate(...)`를 사용하여 다른 컴퓨팅 서비스 메서드의 결과를 캡처하고 무효화합니다.

무엇을 무효화해야 하는지 정확하게 짚어내기 어려운 경우도 있을 거라고 예상할 수 있습니다. 예,  여기에 약간 까다로운 예가 있습니다.

```csharp
// The code from ChatService.cs from Stl.Samples.Blazor.Server.
[ComputeMethod]
public virtual async Task<ChatPage> GetChatTailAsync(int length)
{
    using var dbContext = GetDbContext();

    // The same code as usual
    var messages = dbContext.Messages.OrderByDescending(m => m.Id).Take(length).ToList();
    messages.Reverse();
    // Notice we fetch users in parallel by calling GetUserAsync(...) 
    // instead of using a single query with left outer join in SQL? 
    // Seems sub-optiomal, right?
    var users = await Task.WhenAll(messages
        .DistinctBy(m => m.UserId)
        .Select(m => GetUserAsync(m.UserId, cancellationToken)));
    var userById = users.ToDictionary(u => u.Id);

    await EveryChatTail(); // <- Notice this line
    return new ChatPage(messages, userById);
}

[ComputeMethod]
protected virtual async Task<Unit> EveryChatTail() => default;
```

> Q: 여기서 문제는 무엇입니까?

A: `GetChatTailAsync(...)`에는 `length` 인수가 있으므로 `AddMessageAsync` 메서드를 작성한다고 가정해 보겠습니다. 다른 클라이언트가 다른 길이 값으로 이 메서드를 호출한다고 가정하고 무효화할 모든 채팅 테일을 어떻게 찾습니까?

> Q: 병렬로 사용자를 가져오는 것이 여기에서 괜찮은 이유는 무엇입니까?

A: `GetUserAsync()`는 계산 서비스 메서드로, 결과가 캐시됩니다. 따라서 실제 채팅 앱에서 이러한 호출은 DB를 통해 해결될 것으로 예상되지 않습니다. 대부분의 사용자는 이미 캐시되어 있어야 합니다. 즉, 이러한 호출은 DB에 도달하지 않고 동기식으로 완료됩니다.

`GetUserAsync()`를 호출하는 두 번째 이유는 결과 채팅 페이지가 거기에 나열된 모든 사용자에 종속되도록 만들기 위한 것입니다. 이것이 채팅 샘플에서 사용자 이름의 변경 사항을 즉시 확인할 수 있는 이유입니다. 이름이 변경되면 해당 GetUserAsync() 호출 결과가 무효화되고, 차례로 이 사용자가 사용된 모든 채팅 페이지가 무효화됩니다.

> Q: 분명히 아무 것도 하지 않는 `EveryChatTail()`을 호출하는 이유는 무엇입니까?

A: 이 호출은 채팅 테일 페이지를 종속 페이지로 만들기 위한 것입니다. `EveryChatTail()`의 사용법을 찾으면 다른 것을 찾을 수 있습니다.

```csharp
public async Task<ChatMessage> AddMessageAsync(long userId, string text)
{
    using var dbContext = CreateDbContext();
    
    // Again, this absolutely usual code
    await GetUserAsync(userId, cancellationToken); // Let's make sure the user exists
    var messageEntry = dbContext.Messages.Add(new ChatMessage() {
        CreatedAt = DateTime.UtcNow,
        UserId = userId,
        Text = text,
    });
    await dbContext.SaveChangesAsync(cancellationToken);
    var message = messageEntry.Entity;

    // And that's the extra invalidation logic:
    Computed.Invalidate(EveryChatTail); // <-- Pay attention to this line
    return message;
}
```

보시다시피 여기에서 가짜 데이터 소스인 `EveryChatTail()`에 대한 의존성을 만들고 이 가짜 데이터 소스를 무효화하여 길이에 따라 독립적으로 모든 채팅 테일을 무효화합니다.

훨씬 더 복잡한 경우에 동일한 트릭을 사용하여 데이터를 무효화할 수 있습니다. 이러한 메소드에도 매개변수를 도입하고, 많은 매개변수를 호출하고, 더 "광범위한" 범위로 재귀적으로 호출하도록 할 수 있습니다.

자, 이제 수동 무효화에는 일반적으로 데이터를 수정하는 모든 메소드당 1 ~ 3줄의 추가 코드가 필요하고 데이터를 읽는 모든 메소드당 0줄(일반적으로)의 추가 코드 라인이 필요하다는 것을 알고 있습니다. 많지는 않지만 그래도 이 가격을 지불하고 실시간 UI를 갖고 싶으신가요?

물론 대답은 "그것은 상황에 따라 다름"이지만, 추가 비용은 분명히 엄청나게 높아 보이지 않습니다. 어쨌든 실시간 UI가 필요하거나 실시간 무효화 기능이 있는 강력한 캐싱 계층이 필요한 경우 여기에 표시된 접근 방식이 가장 좋은 옵션일 수 있습니다.

Fusion 용어로 전환하면 다음에 대해 배웠습니다.
- [Compute 서비스](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part01.md) - Compute 서비스를 가진 서비스. 이러한 메서드는 `[ComputeMethod]` 특성으로 장식되며
- 내부에서 [IComputed] 일명 [Computed Values]와 함께 제공됩니다.

하지만 몇 가지 더 흥미로운 개념이 있으며 다음으로 중요한 것은 [Replica 서비스](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part04.md)입니다.


### 분산 컴퓨팅 서비스

먼저, 다음과 같은 "종류"의 `IComputed<T>`를 만드는 데 방해가 되는 것은 없습니다.

```csharp
public class ReplicaComputed<T> : IComputed<T> {
    ConsistencyState ConsistencyState { get; } // Computing -> Consistent -> Invalidated
    T Value { get; }
    Action Invalidated;
    
    public ReplicaComputed<T>(IComputed<T> source) {
        Value = source.Value;
        ConsistencyState = source.ConsistencyState;
        source.Invalidated += () => Invalidate();
    } 

    public void Invalidate() { ... }
}
```

보시다시피 소스의 동작을 "복제" 하는 것 외에는 아무 것도 하지 않습니다. 별로 유용해 보이지 않죠? 그러나 원격 복제본은 어떻습니까? 클라이언트가 원격 복제본을 생성할 수 있도록 서버 측에서 계산된 인스턴스를 게시할 수 있는 것을 구현하고 이러한 원격 복제본이 모든 동일한 작업을 지원하게 된다면 어떻게 될까요? 예를 들어 무효화 이벤트를 구독하거나 업데이트를 요청할 수 있습니까?

간단히 말해서 이러한 유형은 Fusion에 실제로 존재하며 내가 설명한 대로 거의 작동합니다. 업데이트에 대해 말하자면 `IComputed<T>`에 유용한 메서드를 하나 더 추가해 보겠습니다.

```csharp
interface IComputed<T> {
    ConsistencyState ConsistencyState { get; }
    T Value { get; }
    Action Invalidated; 
    
    void Invalidate();
    Task<IComputed<T>> UpdateAsync(); // THIS ONE
}
```

`UpdateAsync`를 사용하면 동일한 계산에 해당하는 최신 `IComputed<T>`를 얻을 수 있습니다. 현재 `IComputed`가 여전히 일관성이 있으면 단순히 자신을 반환합니다. 그렇지 않으면 계산을 다시 트리거하고 최신 출력을 저장하는 새 `IComputed<T>`를 생성합니다. 또는 `UpdateAsync`를 호출할 때 이미 생성된 경우 캐시에서 가져올 것입니다.

원격 복제본이 `UpdateAsync`도 지원하는 경우 원격 클라이언트는 언제든지 현재 일치하지 않는 값을 자유롭게 업데이트할 수 있습니다!

이제 프로세스 경계조차 볼 수 없도록 복제 또는 실제 컴퓨팅 인스턴스 중 어느 것을 사용하는지 신경 쓰지 않도록 할 수 있을까요? 클라이언트측 코드를 서버측 코드와 동일하게 만들 수 있습니까?

여러분이 추측할 수 있듯이, 예 그렇습니다!

Fusion은 ASP.NET Core API 컨트롤러에 대한 멋진 기본 유형인 `FusionController`를 제공합니다. 이 컨트롤러는 현재 단일 추가 메서드를 제공하며 여기에 전체 소스 코드가 있습니다.

```csharp
protected virtual Task<T> PublishAsync<T>(Func<CancellationToken, Task<T>> producer)
{
    var cancellationToken = HttpContext.RequestAborted;
    var headers = HttpContext.Request.Headers;
    var mustPublish = headers.TryGetValue(FusionHeaders.RequestPublication, out var _);
    if (!mustPublish)
        return producer.Invoke(cancellationToken);
    return Publisher
        .PublishAsync(producer, cancellationToken)
        .ContinueWith(task => {
            var publication = task.Result;
            HttpContext.Publish(publication);
            return publication.State.Computed.Value;
        }, cancellationToken);
}
```

사용하는 방법은 다음과 같습니다.

```csharp
[Route("api/[controller]")]
[ApiController]
public class TimeController : FusionController, ITimeService
{
    TimeService Time { get; }
 
    ...

    [HttpGet("get")]
    public Task<DateTime> GetTimeAsync() 
        => PublishAsync(ct => Time.GetTimeAsync(ct)); // LOOK AT THIS LINE
}
```

`Time.GetTimeAsync()`에 의해 생성된 `IComputed<T>`를 게시하는 데 필요한 전부입니다!

위의 `PublishAsync` 코드를 보면 요청에 `RequestPublication` 헤더(실제 값은 "X-Fusion-Publish"임)가 있는지 확인하고 다음을 확인할 수 있습니다.
- 이 헤더가 있으면 `producer` 실행하고 출력을 게시하기 위해 몇 가지 추가 작업을 수행합니다.
- 그렇지 않으면 단순히 생성된 값을 반환합니다.

이 모든 것은 컨트롤러를 이와 같이 작성하면 "일반" 서버 측 API와 Fusion 게시 메커니즘을 거의 무료로 지원하는 API를 모두 얻을 수 있음을 의미합니다!

증거: Fusion 샘플을 열고 `http://localhost:5005/swagger/` 페이지로 이동하면 다음이 표시됩니다.

![](https://github.com/servicetitan/Stl.Fusion/raw/master/docs/img/SwaggerDoc.jpg)

물론 여기에서 이러한 방법 중 하나를 시작할 수 있습니다.

네트워킹 탭에서 수행되는 작업을 확인하면 채팅 샘플이 다음과 같습니다.

![](https://github.com/servicetitan/Stl.Fusion/raw/master/docs/img/ChatNetworking.gif)

다음 사항을 살펴보세요.
- 모든 `get*` API 메서드는 주어진 인수 집합에 대해 한 번만 호출됩니다. 한 번의 API 호출로 이 호출에 대해 배후에서 생성된 서버 측 계산 인스턴스의 클라이언트 측 복제본을 생성하기에 충분합니다.
- 이 복제본에 대한 업데이트는 먼저 표시된 WebSocket 연결을 통해 제공됩니다.

이러한 API 요청의 헤더는 다음과 같습니다.

![](https://github.com/servicetitan/Stl.Fusion/raw/master/docs/img/FusionHeaders.gif)

Fusion API를 사용하는 클라이언트 측 코드가 어떻게 생겼는지 궁금할 것입니다. 다음은 "서버 화면" 샘플을 구동하는 거의 모든 클라이언트 측 코드(보기 제외)입니다.

- 이 인터페이스는 말 그대로 배후에서 계산된 복제본을 제공하는 API 끝점을 "소비"하는 데 필요한 전부입니다! Fusion은 [RestEase](https://github.com/canton7/RestEase)를 사용하여 런타임에 이 인터페이스를 구현하는 실제 HTTP 클라이언트를 생성하고 복제본과 관련된 모든 것을 가로채기 위해 다시 한 번 "래핑"합니다.

```csharp
// typeof(ITimeService) on the next line tells the type Fusion exposes it as
[RestEaseReplicaService(typeof(ITimeService))] 
[BasePath("time")]
public interface ITimeClient : IRestEaseReplicaClient
{
    [Get("get")]
    Task<DateTime> GetTimeAsync(CancellationToken cancellationToken = default);
}
```

- 그리고 이것은 이 서비스를 사용하여 UI 구성 요소를 자동 업데이트합니다.

```csharp
@page "/serverTime"
@using System.Threading
@inherits LiveComponentBase<DateTime>
@inject ITimeService TimeService

@{
    var time = State.LastValue.Format();
    var error = State.Error;
}

<h1>Server Time</h1>

<p>Server Time: @time</p>

@if (error != null) {
    <div class="alert alert-warning" role="alert">
        Update error: @error.Message
    </div>
}

<button class="btn btn-primary" @onclick="() => State.Invalidate(true)">Refresh</button>

@code {
    protected override Task<DateTime> ComputeStateAsync(CancellationToken cancellationToken)
        => TimeService.GetTimeAsync(cancellationToken);
}
```

이 구성 요소는 `LiveComponentBase<T>`에서 상속하므로 `State` 속성([라이브 상태](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part03.md))과 변경 후 다시 계산하는 데 필요한 모든 논리가 있습니다. [여기에서 이에 대한 자세한 내용을 읽을 수 있습니다](https://github.com/servicetitan/Stl.Fusion/blob/master/README.md#enough-talk-show-me-the-code).

클라이언트에서 [Compute 서비스](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part01.md)를 복제할 수 있는 기능을 [Replica 서비스](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part04.md)라고 합니다. 이러한 서비스는 컴퓨팅 서비스와 다른가요? 예 또는 아니오:
- 예, 자동으로 구현되기 때문입니다.
- 아니요. 모방한 [Compute 서비스](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part01.md)와 거의 똑같이 동작하기 때문에 특히 다른 [Compute 서비스](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/Part01.md)에서 생성하는 값을 "소비"할 수 있으며 모든 무효화 체인이 제대로 작동합니다.

![](https://github.com/servicetitan/Stl.Fusion/raw/master/docs/img/Stl-Fusion-Chat-Sample.gif)

"구성" 샘플(오른쪽 하단 창에 표시됨)이 정확히 이를 증명합니다. 두 가지 다른 방법으로 자체 모델을 "구성"합니다.
- 첫 번째 패널의 UI 모델은 [서버 측에서 구성](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/src/Blazor/Server/Services/ComposerService.cs)됩니다. 클라이언트 측 복제본은 패널을 표시하는 구성 요소에 바인딩됩니다.
- 두 번째 패널은 클라이언트에서 사용된 모든 값의 서버 측 복제본을 결합하여 클라이언트에서 완전히 구성된 UI 모델을 사용합니다.
- **놀라운 부분**: 위의 두 파일은 거의 동일합니다!

그렇기 때문에 Fusion은 거의...


### 투명한 추상화

아마도 `IComputed<T>`를 직접 처리할 필요가 거의 없다는 것을 이미 발견했을 것입니다.
- API의 일부로 볼 수 없습니다.
- 원격 클라이언트에서 볼 수 없습니다.
- 그리고 Fusion 기반의 웹 API도 일반 API처럼 보입니다!
- 그럼에도 불구하고 거기에 있고 작동합니다!

이러한 의미에서 Fusion은 우리 모두가 좋아하는 훨씬 더 유명한 투명 추상화인 Garbage Collection과 매우 유사합니다.

![](https://github.com/servicetitan/Stl.Fusion/raw/master/docs/img/Invisiboard.jpg)

집중력 유지, 이것이 마지막 주제입니다:)


### Fusion 및 다른 기술


#### SignalR, Pusher 등

기술적으로 Fusion에는 [SignalR](https://dotnet.microsoft.com/en-us/apps/aspnet/signalr) 또는 이와 유사한 것이 필요하지 않습니다.
- 예, 클라이언트에게 메시지를 푸시할 수 있습니다.
- 그리고 Fusion이 현재 "복제본에 대한 원래 계산된 인스턴스가 무효화되었습니다"라는 한 가지 유형의 메시지만 "푸시"하고 있지만, 이는 후속 업데이트(또는 다른 API 호출)가 필요한 모든 데이터를 가져올 수 있기 때문에 나머지를 가지기에 충분합니다.

다음과 같이 생각할 수 있습니다.
- 보통 우리가 보내는 메시지는 두 가지 정보를 결합합니다: "상태가 바뀌었다" + "바로 그렇게 바뀌었다."
- Fusion은 상태 변경을 추적하도록 설계된 추상화입니다. 첫 번째 종류의 메시지("상태 변경됨")만 전송하지만 이 알림을 받은 후 실제 변경 사항을 받는 것은 저렴한 작업입니다. Fusion은 또한 변경 후 모든 것이 한 번만 계산되도록 보장하기 때문입니다.

다음은 메시징을 구현하는 데 사용할 수 있는 API 끝점의 예입니다.

```csharp
Task<int> GetLastMessageIndexAsync(string userId);
Task<Message[]> GetLastMessagesAsync(string userId, int count);
```

이러한 솔루션은 SignalR의 서버 측 브로드캐스트`(Clients.All.*` 호출)와 같은 것보다 훨씬 낫습니다. 이러한 메시지를 지속하면 클라이언트 연결이 끊기고 서버가 재시작 된 후에도 안정적으로 전달되기 때문에 Fusion은 복제본에 대해 자동으로 다시 연결되며, 이 경우 모든 클라이언트 측 복제본이 최신 상태를 쿼리합니다.

그 외에도, "주제"나 "구독"과 같은 개념에 대해 생각할 필요가 없습니다. 클라이언트에서 소비하는 모든 것이 자동으로 업데이트되며, 여러 클라이언트에서 공유되는 것이라면 이 항목을 "주제"로 생각할 수 있습니다.

물론 SignalR이 Fusion에서 절대적으로 쓸모가 없다는 의미는 아닙니다. 여전히 이점을 얻을 수 있는 시나리오가 있습니다. 예를 들어, 업데이트를 가능한 한 빨리 전달하고 싶다면 SignalR이 더 나은 선택이 될 수 있습니다. 현재 Fusion에 업데이트를 무효화 메시지와 함께 푸시하도록 지시할 방법이 없으므로 모든 업데이트에는 무효화 후 명시적인 왕복이 필요합니다. 나중에 이것이 실제로 매우 합리적인 이유를 배우게 될 것이지만 그럼에도 불구하고 이것이 거래 위반이 될 수 있는 경우가 있습니다. 우리는 미래에 이 특정 시나리오를 다루는 것을 고려하지만 이것이 완료되더라도 Fusion보다 X를 선호하는 다른 경우와 이유가 항상 있습니다. 은색 총알이 될 수는 없습니다. 아무리 노력해도 은색 총알이 될 수는 없습니다. :)

추신: 유사점과 차이점을 자세히 알아보려면 ["Fusion이 SignalR과 얼마나 유사합니까?"](https://medium.com/@alexyakunin/how-similar-is-stl-fusion-to-signalr-e751c14b70c3?source=friends_link&sk=241d5293494e352f3db338d93c352249)를 확인하세요.

#### Fusion + Blazor = ❤

여기에 표시된 클라이언트 측 코드의 모든 부분이 [Blazor](https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor)에서 실행되어야 하는 이유는 무엇입니까?

![](https://github.com/servicetitan/Stl.Fusion/raw/master/docs/img/Blazor.jpg)

Fusion은 불과 5개월 된 프로젝트(2020년 10월 1일)입니다. 인간적인 측면에서 볼 때, 그녀는 아직 기어다니기 시작조차 하지 말아야 한다:) 그리고 지금까지 그것은 1인 창조물입니다. - 그것은 완제품보다는 MVP에 가깝습니다. 짐작하시겠지만, 이 모든 + 샘플, 설명서, 그리고 심지어 첫 번째 실제 구현(예, 이미 구현이 있습니다!)에 대한 3개월은 매우 빡빡한 일정입니다.

완전한 기능을 갖춘 자바스크립트 클라이언트를 이 "팩"에 추가하면, MobX에 의존하여 구축한다고 가정할 때, 즉, 서버에 있는 화려한 "투명 추상화" 개념을 복제하려고 시도조차 하지 않을 경우, 타임라인은 최소 1개월 정도 확장됩니다.

이것이 바로 내가 자바스크립트 클라이언트를 구현하려는 생각을 버리고 Blazor에 베팅한 최초의 이유이다.

> 이 작업을 시도하는 것을 환영합니다. 기꺼이 도와드리겠습니다.

그리고 그것이 Blazor가 정말 놀랍다는 것을 알게 된 방법입니다! 나는 클라이언트 측 Blazor, 즉 WebAssembly 부분에 대해 이야기하고 있습니다. 서버 측 Blazor가 더 나쁘지는 않다고 확신하지만, 단지 달성하려고 했던 것과 맞지 않았기 때문에 아직 즐기지 않았습니다.

Blazor에 대해 가장 인상 깊었던 몇 가지를 나열하겠습니다.
- UI 구성 요소 모델은 .NET 세계에 맞게 조정된 React의 모방품입니다. React를 안다면 Blazor를 아주 빨리 배우게 될 것입니다. 기본적으로 여러분이 알고 있는 거의 모든 개념에 대해 1-1 매핑이 있습니다.
- React 스타일 구성 요소 모델은 개념적으로 오늘날 UI에 사용할 수 있는 최상의 옵션입니다.
- Blazor는 `netstandard2.1` 대상과 매우 호환됩니다. 예를 들어, [이와 같은 코드](https://github.com/servicetitan/Stl.Fusion/blob/master/src/Stl/Async/TaskSource.cs#L89)가 수정 없이 Blazor에서 실행될 것이라고는 전혀 예상하지 못했습니다. 명확하게 하기 위해 `TaskCompletionSource`의 전용 읽기 전용 필드를 작성하기 때문에 평소와 같이 구성해도 유효성 검사를 통과하지 않는 런타임 생성 람다 식을 컴파일합니다.
- 아직 프리뷰 상태인 상태에서 작업을 시작했습니다. 버그나 이상한 문제가 발견되지 않았습니다. 릴리스 전이나 후에 없습니다.
- 예, 속도는 .NET Core와 동등하지 않습니다. 현재 MSIL은 JITted가 아닌 해석되기 때문에 전혀 동등하지 않습니다. 하지만:
  - 지금까지는 이 수준의 성능으로도 충분했습니다. Fusion 클라이언트에서 실행되는 로직의 양을 고려하면 인상적입니다(즉, 서버에서 실행되는 모든 런타임 생성 프록시, 인수 캡처, 캐싱 등과 같은 로직과 정확히 동일합니다).
  - Server-Side Blazor를 사용하는 옵션도 매우 중요합니다. 아마도 이것이 내년의 주요 옵션이 될 것입니다. 그리고 Fusion을 사용하면 약간의 추가 작업으로 두 모드를 모두 타겟팅할 수 있다는 사실을 알고 계십니까? Blazor 샘플을 확인하십시오. 인덱스 페이지에서 이 두 모드 사이를 전환할 수 있습니다.
- 마지막으로 블레이저에 대한 AOT 컴파일은 약 1년 이내에 이루어질 것으로 예상되며, 이는 블레이저 웹어셈블리를 훨씬 더 매력적으로 만들 것입니다.

그 외에도 Blazor는 특히 .NET Core(또는 .NET)에서 서버 측 코드를 실행하는 회사에 매우 좋은 장기 투자라고 생각합니다.
- WASM은 젊고 꽤 빠르게 개선되고 있습니다. 최근에 스레드가 생겼습니다. JavaScript에서는 스레드를 얻을 수 없습니다. 20개 이상의 코어 CPU가 주류가 되는 세상에서 이 단 하나의 성능 차이도 20배의 성능 차이를 만들 수 있습니다.
- 요즘 JavaScript는 사용하려는 실제 언어라기보다는 VM에 가깝습니다. 사용하는 실제 언어는 TypeScript 또는 JavaScript의 모든 고통스러운 문제를 해결하도록 설계된 형제 중 하나입니다. 그리고 WebAssembly로도 컴파일할 수 있다고 가정하고 JavaScript로 컴파일하는 것은 점점 무의미해지고 있습니다. 그렇다면 TypeScript에 작성된 코드베이스의 장기적인 미래가 실제로 큰 문제에 직면한다면 왜 TypeScript와 같은 기술에 베팅하고 싶습니까? 즉. 그것이 작동하지 않는다는 것은 아니지만 미래에 최선의 선택이 되지 않을 것이라고 가정하고 다른 것으로 전환하는 것을 연기하는 요점은 무엇입니까?


#### 다음 단계

- [튜토리얼](https://github.com/servicetitan/Stl.Fusion.Samples/blob/master/docs/tutorial/README.md)을 확인하거나 [문서 홈](https://github.com/servicetitan/Stl.Fusion/blob/master/docs/README.md)으로 이동
- [Discord 서버](https://discord.com/invite/EKEwv6d) 또는 [Gitter](https://gitter.im/Stl-Fusion/community)에 가입하여 질문을 하고 프로젝트 업데이트를 추적하세요.
