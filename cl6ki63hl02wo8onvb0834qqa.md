---
title: "Fusion 튜토리얼 - 9부: CommandR"
datePublished: Mon Aug 08 2022 08:39:40 GMT+0000 (Coordinated Universal Time)
cuid: cl6ki63hl02wo8onvb0834qqa
slug: fusion-9-commandr
tags: dotnet, stlfusion

---

[Stl.CommandR](https://www.nuget.org/packages/Stl.CommandR/)은 CQRS 스타일의 명령 처리기를 구현하는 데 도움이 되는 MediatR과 유사한 라이브러리입니다. 다른 추상화 세트와 함께 추가 코드 없이 이전 섹션에서 설명한 파이프라인을 얻을 수 있습니다.

> 튜토리얼의 이 부분은 CommandR 자체를 다룰 것입니다. 다음은 강력한 CQRS 파이프라인을 구현하기 위해 다른 Fusion 서비스와 함께 사용하는 방법을 보여줍니다.

CommandR은 MediatR과 동일한 문제를 해결하지만 몇 가지 새로운 기능을 제공합니다.

- 통합 처리기 파이프라인. 모든 CommandR 처리기는 필터(~ 미들웨어와 유사한 처리기) 또는 최종 처리기로 작동할 수 있습니다. MediatR은 CommandR의 필터링 처리기와 유사한 파이프라인 동작을 지원하지만 모든 명령에 대해 동일합니다. 그리고 이 기능은 실제로 매우 유용합니다. `IPreparedCommand`용 내장 필터는 유효성 검사를 통합하는 데 도움이 됩니다.

- `CommandContext` - 비 핸들러 코드가 현재 실행 중인 명령과 관련된 상태를 저장하고 액세스 할 수 있도록 도와주는 `HttpContext`와 유사한 유형입니다. 명령 컨텍스트가 중첩될 수 있지만(명령이 다른 명령을 호출할 수 있음) 전체 계층을 항상 사용할 수 있습니다.

- 규칙 기반 명령 처리기 검색 및 호출. 핸들러를 추가할 때마다 `ICommandHandler<TCommand, TResult>`를 구현할 필요가 없습니다. 어떠한 비동기 메서드라도 `[CommandHandler]`와 함께 쓰일 때  명령이 첫번째 매개 변수이고 마지막 매개 변수가 `CancellationToken` 이면 동작합니다. 다른 모든 매개 변수는 IoC 컨테이너를 통해 해결됩니다.

- AOP 스타일 명령 처리기. 이러한 핸들러는 두 개의 인수`(command, cancellationToken)`이 있는 virtual 비동기 메서드입니다. AOP 부분이 작동하도록 하려면 이러한 핸들러를 선언하는 유형을 실제 구현 유형 대신 런타임 생성 프록시를 등록하는 `IServiceCollection`에 대한 확장 메소드인 `AddCommandService(...)`로 등록해야 합니다. 프록시는 이러한 메서드에 대한 모든 호출이 여전히 `Commander.Call(command)`을 통해 라우팅되어 이 명령에 대한 전체 파이프라인(즉, 연결된 다른 모든 핸들러)을 호출하도록 합니다. 즉, 이러한 핸들러는 직접 또는 `Commander`를 통해 호출할 수 있지만 결과는 항상 동일합니다.

많은 분들이 MediatR에 익숙하기 때문에 다음은 해당 용어를 CommandR 용어로 매핑한 것입니다.

https://github.com/servicetitan/Stl.Fusion.Samples/raw/master/docs/tutorial/img/MediatR-vs-CommandR.jpg

적어도 앞에서 언급한 고유한 기능 중 일부를 사용하지 않는 한 CommandR에서 제공하는 API가 다소 단순하다는 것을 알 수 있습니다.


## Hello, CommandR!

첫 번째 명령과 MediatR 스타일 처리기를 선언:
```csharp
public class PrintCommand : ICommand<Unit>
{
    public string Message { get; set; } = "";
}

// Interface-based command handler
public class PrintCommandHandler : ICommandHandler<PrintCommand>, IDisposable
{
    public PrintCommandHandler() => WriteLine("Creating PrintCommandHandler.");
    public void Dispose() => WriteLine("Disposing PrintCommandHandler");

    public async Task OnCommand(PrintCommand command, CommandContext context, CancellationToken cancellationToken)
    {
        WriteLine(command.Message);
        WriteLine("Sir, yes, sir!");
    }
}
```

CommandR과 MediatR을 사용하는 것은 매우 유사:
```csharp
// IoC 컨테이너 빌드
var serviceBuilder = new ServiceCollection()
    .AddScoped<PrintCommandHandler>(); // 이것을 AddSingleton으로 변경해 보십시오.
var commanderBuilder = serviceBuilder.AddCommander()
    .AddHandlers<PrintCommandHandler>();
var services = serviceBuilder.BuildServiceProvider();

var commander = services.Commander(); // .GetRequiredService<ICommander>()와 동일
await commander.Call(new PrintCommand() { Message = "Are you operational?" });
await commander.Call(new PrintCommand() { Message = "Are you operational?" });
```

출력:
```
Creating PrintCommandHandler.
Are you operational?
Sir, yes, sir!
Disposing PrintCommandHandler
Creating PrintCommandHandler.
Are you operational?
Sir, yes, sir!
Disposing PrintCommandHandler
```

고지:
- CommandR은 명령 처리기 서비스를 자동 등록하지 않습니다. 이 서비스에서 사용 가능한 명령 처리기에 명령을 매핑하는 방법을 알아내는 데만 관심이 있습니다. 따라서 서비스를 별도로 등록해야 합니다.
- `Call`은 모든 명령 호출에 대한 서비스를 확인하기 위해 자체 `IServiceScope`를 만듭니다.

위의 예에서 `AddScoped`를 `AddSingleton`으로 변경해 보십시오.


## 규칙 기반 핸들러, CommandContext, 재귀

`CommandContext`가 어떻게 작동하는지 보기 위해 좀 더 복잡한 핸들러를 작성해 보겠습니다.
```csharp
public class RecSumCommand : ICommand<long>
{
    public long[] Numbers { get; set; } = Array.Empty<long>();
}

public class RecSumCommandHandler
{
    public RecSumCommandHandler() => WriteLine("Creating RecSumCommandHandler.");
    public void Dispose() => WriteLine("Disposing RecSumCommandHandler");

    [CommandHandler] // ICommandHandler<RecSumCommand, long> 지원이 필요하지 않습니다.
    private async Task<long> RecSum(
        RecSumCommand command,
        IServiceProvider services, // CommandContext.Services를 통해 해결됨
        ICommander commander, // CommandContext.Services를 통해 해결됨
        CancellationToken cancellationToken)
    {
        var context = CommandContext.GetCurrent();
        Debug.Assert(services == context.Services); // context.Services는 범위가 지정된 IServiceProvider
        Debug.Assert(commander == services.Commander()); // ICommander은 싱글톤
        Debug.Assert(services != commander.Services); // 범위가 지정된 IServiceProvider != 최상위 IServiceProvider
        WriteLine($"Numbers: {command.Numbers.ToDelimitedString()}");

        // 각 호출은 새로운 CommandContext를 생성
        var contextStack = new List<CommandContext>();
        var currentContext = context;
        while (currentContext != null)
        {
            contextStack.Add(currentContext);
            currentContext = currentContext.OuterContext;
        }
        WriteLine($"CommandContext stack size: {contextStack.Count}");
        Debug.Assert(contextStack[^1] == context.OutermostContext);

        // 마지막으로 CommandContext.Items는 HttpContext.Items와 비슷하며
        // 서비스 범위와 유사하게 재귀 호출 체인의 모든 컨텍스트에 대해 동일합니다.
        var depth = 1 + (int)(context.Items["Depth"] ?? 0);
        context.Items["Depth"] = depth;
        WriteLine($"Depth via context.Items: {depth}");

        // 마지막으로 실제 핸들러 로직입니다. :)
        if (command.Numbers.Length == 0)
            return 0;
        var head = command.Numbers[0];
        var tail = command.Numbers[1..];
        var tailSum = await context.Commander.Call(
            new RecSumCommand() { Numbers = tail }, false, // true로 변경해 봅니다.
            cancellationToken);
        return head + tailSum;
    }
}
```

```csharp
// Building IoC container
var serviceBuilder = new ServiceCollection()
    .AddScoped<RecSumCommandHandler>();
var commanderBuilder = serviceBuilder.AddCommander()
    .AddHandlers<RecSumCommandHandler>();
var services = serviceBuilder.BuildServiceProvider();

var commander = services.Commander(); // .GetRequiredService<ICommander>()와 동일
WriteLine(await commander.Call(new RecSumCommand() { Numbers = new[] { 1L, 2, 3 } }));
```

출력:
```
Creating RecSumCommandHandler.
Numbers: 1, 2, 3
CommandContext stack size: 1
Depth via context.Items: 1
Numbers: 2, 3
CommandContext stack size: 2
Depth via context.Items: 2
Numbers: 3
CommandContext stack size: 3
Depth via context.Items: 3
Numbers: 
CommandContext stack size: 4
Depth via context.Items: 4
6
```

몇 가지 흥미로운 점:
1. 규칙 기반 명령 처리기를 사용할 수 있습니다. `ICommandHandler<TCommand>`를 구현하는 대신 `[CommandHandler]`로 메서드를 장식하기만 하면 됩니다.
1. 이러한 핸들러는 다음 인수로 더 유연함:
   - 첫 번째 인수는 항상 명령이어야 합니다.
   - 마지막 것은 항상 `CancellationToken`이어야 합니다.
   - `CommandContext` 매개 변수는 `CommandContext.GetCurrent()`를 통해 확인됩니다.
   - 다른 모든 것은 `CommandContext.Services`, 즉 범위가 지정된 서비스 공급자를 통해 해결됩니다.

그러나 이 예제의 가장 복잡한 부분은 `CommandContext`를 다룹니다. 컨텍스트는 "유형이 지정"됩니다. 모두 `CommandContext`에서 상속되더라도 실제 유형은 `CommandContext<TResult>`입니다.

명령 컨텍스트를 통해 다음을 수행:
- 현재 실행 중인 명령 찾기
- 결과를 설정하거나 읽습니다. 일반적으로 결과를 수동으로 설정할 필요가 없습니다. 명령 처리기를 호출하는 코드는 "가장 깊은" 처리기가 존재하면 결과가 설정되도록 보장하지만 파이프라인의 일부 처리기에서 결과를 읽기 원할 수도 있습니다.
- `IServiceScope` 관리
- `항목`에 액세스합니다. 현재 명령과 관련된 모든 정보를 저장하는 데 도움이 되는 스레드로부터 안전한 사전과 같은 구조인 `OptionSet`입니다.

마지막으로 `CommandContext`는 클래스이지만 [API를 정의하는 내부 인터페이스인 `ICommandContext`](https://github.com/servicetitan/Stl.Fusion/blob/master/src/Stl.CommandR/Internal/ICommandContext.cs)도 있습니다. 확인하십시오. 세부 정보를 찾고 있다면 CommandContext 자체를 확인하십시오.

따라서 명령을 호출하면 새 `CommandContext`가 생성됩니다. 그러나 서비스 범위는 어떻습니까? `CommandContext<TResult>` 생성자 코드는 몇 문장보다 이것을 더 잘 설명합니다.
```csharp
// 여기에서 PreviousContext는 CommandContext.Current이며, 이는 곧 `this`로 대체될 것입니다.
if (PreviousContext?.Commander != commander) {
    OuterContext = null;
    OutermostContext = this;
    ServiceScope = Commander.Services.CreateScope();
    Items = new OptionSet();
}
else {
    OuterContext = PreviousContext;
    OutermostContext = PreviousContext!.OutermostContext;
    ServiceScope = OutermostContext.ServiceScope;
    Items = OutermostContext.Items;
}
```

보시다시피 이러한 호출에서 `ICommander` 인스턴스를 "전환"하면 새 컨텍스트가 최상위 컨텍스트인 것처럼 동작합니다. 즉, 새 서비스 범위인 새 항목을 만들고 자신을 `OutermostContext`로 노출합니다.

이제 위의 이 부분에서 `false`를 `true`로 변경하는 것이 좋습니다.
```csharp
var tailSum = await context.Commander.Call(
    new RecSumCommand() { Numbers = tail }, false, // true로 변경해 봅시다.
    cancellationToken);
```


## 명령을 실행하는 방법

[ICommander는 명령을 실행하는 단일 방법을 제공](https://github.com/servicetitan/Stl.Fusion/blob/master/src/Stl.CommandR/ICommander.cs)하지만 실제로는 가장 "낮은 수준" 옵션이므로 거의 필요하지 않습니다.

실제 옵션은 [CommanderExt 유형](https://github.com/servicetitan/Stl.Fusion/blob/master/src/Stl.CommandR/CommandExt.cs)으로 구현됩니다(Ext는 이러한 클래스에 대해 Stl의 모든 곳에서 사용되는 Extension의 바로 가기입니다).
- `Call`는 일반적으로 사용해야 하는 것입니다. 명령을 "호출"하고 결과를 반환합니다.
- `Run`은 `Call`처럼 작동하지만 대신 `CommandContext`를 반환합니다. 그렇기 때문에 명령 처리기 중 하나가 예외를 throw하지 않는 경우에도 예외가 발생하지 않습니다. 어떤 경우에도 성공적으로 완료됩니다. 예를 들어 `CommandContext.UntypedResult`를 사용하여 실제 명령 완료 결과 또는 예외를 가져올 수 있습니다.
- `Start`는 명령을 시작할 때 발생 시키고 잊어버리는 방법입니다. `Run`과 유사하게 `CommandContext`를 반환하지만 이 컨텍스트를 즉시 반환합니다. 즉, 이 컨텍스트와 연결된 명령이 여전히 실행중일 수 있습니다. `CommandContext`를 통해 명령이 언제 결과를 생성했는지 알 수 있지만(예: `CommandContext.UntypedResultTask`를 통해) 결과 자체가 이 명령의 파이프라인이 실행을 완료했음을 의미하지 않으므로 해당 핸들러의 코드가 계속 실행 중일 수 있습니다.

이 모든 메서드는 최대 3개의 인수를 사용:
- ICommand - 분명히
- `bool isolate = false` - 명령을 격리된 방식으로 실행해야 하는지 여부를 나타내는 선택적 매개변수입니다. true이면 [ExecutionContext.SuppressFlow 블록](https://docs.microsoft.com/en-us/dotnet/api/system.threading.executioncontext.suppressflow?view=net-5.0) 내에서 명령이 실행되므로 가장 바깥쪽에 있는 것도 확실합니다.
- `CancellationToken cancellationToken = default` - 거의 모든 비동기 메서드의 일반적인 인수입니다.


## 명령 서비스 및 필터링 처리기

명령 핸들러를 등록하는 가장 흥미로운 방법은 소위 명령 서비스 내에 명령 핸들러를 선언하는 것입니다.
```csharp
public class RecSumCommandService
{
    [CommandHandler] // ICommandHandler<RecSumCommand, long> 지원이 필요하지 않음
    public virtual async Task<long> RecSum( // "public virtual"임에 주목!
        RecSumCommand command,
        // 여기에 추가 매개변수 있을 수 없습니다.
        CancellationToken cancellationToken = default)
    {
        if (command.Numbers.Length == 0)
            return 0;
        var head = command.Numbers[0];
        var tail = command.Numbers[1..];
        var tailSum = await RecSum( // Note it's a direct call, but the whole pipeline still gets invoked!
            new RecSumCommand() { Numbers = tail },
            cancellationToken);
        return head + tailSum;
    }

    // 이 핸들러는 ANY 명령(ICommand)과 연결됩니다. Priority = 10은 기본 우선순위가 
    // 0인 핸들러보다 먼저 실행됨을 의미합니다. IsFilter는 InvokeRemainingHandlers를 통해
    // 다른 핸들러를 트리거한다고 알려줍니다.
    [CommandHandler(Priority = 10, IsFilter = true)]
    protected virtual async Task DepthTracker(ICommand command, CancellationToken cancellationToken)
    {
        var context = CommandContext.GetCurrent();
        var depth = 1 + (int)(context.Items["Depth"] ?? 0);
        context.Items["Depth"] = depth;
        WriteLine($"Depth via context.Items: {depth}");

        await context.InvokeRemainingHandlers(cancellationToken).ConfigureAwait(false);
    }

    // Another filter for RecSumCommand
    [CommandHandler(Priority = 9, IsFilter = true)]
    protected virtual Task ArgumentWriter(RecSumCommand command, CancellationToken cancellationToken)
    {
        WriteLine($"Numbers: {command.Numbers.ToDelimitedString()}");
        var context = CommandContext.GetCurrent();
        return context.InvokeRemainingHandlers(cancellationToken);
    }
}
```

이러한 서비스는 `CommanderBuilder`의 `AddCommandService` 메서드를 통해 등록해야 합니다.
```csharp
// Building IoC container
var serviceBuilder = new ServiceCollection();
var commanderBuilder = serviceBuilder.AddCommander()
    .AddCommandService<RecSumCommandService>(); // 이러한 서비스는 싱글톤으로 자동 등록됨
var services = serviceBuilder.BuildServiceProvider();

var commander = services.Commander();
var recSumService = services.GetRequiredService<RecSumCommandService>();
WriteLine(recSumService.GetType());
WriteLine(await commander.Call(new RecSumCommand() { Numbers = new[] { 1L, 2 } }));
WriteLine(await recSumService.RecSum(new RecSumCommand() { Numbers = new[] { 3L, 4 } }));
```

출력:
```
Castle.Proxies.RecSumCommandServiceProxy
Depth via context.Items: 1
Numbers: 1, 2
Depth via context.Items: 2
Numbers: 2
Depth via context.Items: 3
Numbers: 
3
Depth via context.Items: 1
Numbers: 3, 4
Depth via context.Items: 2
Numbers: 4
Depth via context.Items: 3
Numbers: 
7
```

보시다시피 이러한 서비스에 대해 생성된 프록시 유형은 `ICommander.Call`을 통해 명령 처리기의 모든 직접 호출을 라우팅합니다. 따라서 일반 핸들러와 달리 이러한 핸들러를 직접 호출할 수 있습니다. 어쨌든 전체 CommandR 파이프라인이 호출됩니다.