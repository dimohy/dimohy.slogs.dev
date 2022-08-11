## EF Core 6 배우기 - 6. 로깅

EF Core는 다양한 로깅 방법을 제공합니다.

- 간단한 로깅
- Microsoft.Extensions.Logging
- 이벤트
- 인터셉터
- 진단 수신기


## 간단한 로깅

`DbContext`의 `OnConfiguring()`에 [LogTo()](https://docs.microsoft.com/ko-KR/dotnet/api/microsoft.entityframeworkcore.dbcontextoptionsbuilder.logto)를 이용하면 간단하게 EF Core의 로그를 로깅할 수 있습니다.

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    => optionsBuilder.LogTo(Console.WriteLine);
```

그러면 앞 전의 다음의 코드가 어떤 쿼리를 생성하는지 살펴볼 수 있습니다.

```csharp
var testUser =  c.Users
                .Include(user => user.Todos)
                    .ThenInclude(todo => todo.Tags)
                .First(x => x.UserId == "test");

foreach (var testTodos in testUser.Todos)
{
    Console.WriteLine(testTodos.Memo);
    foreach (var tag in testTodos.Tags)
        Console.WriteLine($"Tags : {tag.TagId} - {tag.Descption}");
}
```

| 로그
```
...
      SELECT "t"."UserId", "t"."UserName", "t3"."Seq", "t3"."CompleteDate", "t3"."IsComplete", "t3"."IsDel", "t3"."Memo", "t3"."TodoDate", "t3"."UserId", "t3"."TagsTagId", "t3"."TodosSeq", "t3"."TagId", "t3"."Descption"
      FROM (
          SELECT "u"."UserId", "u"."UserName"
          FROM "UserInfo" AS "u"
          WHERE "u"."UserId" = 'test'
          LIMIT 1
      ) AS "t"
      LEFT JOIN (
          SELECT "t0"."Seq", "t0"."CompleteDate", "t0"."IsComplete", "t0"."IsDel", "t0"."Memo", "t0"."TodoDate", "t0"."UserId", "t1"."TagsTagId", "t1"."TodosSeq", "t1"."TagId", "t1"."Descption"
          FROM "TodoInfo" AS "t0"
          LEFT JOIN (
              SELECT "t2"."TagsTagId", "t2"."TodosSeq", "t4"."TagId", "t4"."Descption"
              FROM "TodoInfoTodoTagInfo" AS "t2"
              INNER JOIN "TodoTagInfo" AS "t4" ON "t2"."TagsTagId" = "t4"."TagId"
          ) AS "t1" ON "t0"."Seq" = "t1"."TodosSeq"
      ) AS "t3" ON "t"."UserId" = "t3"."UserId"
      ORDER BY "t"."UserId", "t3"."Seq", "t3"."TagsTagId", "t3"."TodosSeq"
...
```


## Microsoft.Extensions.Logging

EF Core는 ASP.NET Core 애플리케이션에서 사용하는 [Microsoft.Extensions.Logging](https://docs.microsoft.com/ko-KR/dotnet/core/extensions/logging?tabs=command-line)와 통합됩니다. 좀 더 자세한 내용은 [EF Core에서 Microsoft.Extensions.Logging 사용](https://docs.microsoft.com/ko-kr/ef/core/logging-events-diagnostics/extensions-logging?tabs=v3)을 살펴보세요.


## 이벤트

EF Core에서는 특정 작업에 대한 이벤트를 수신 할 수 있습니다. `ChangeTracker`의 `StateChanged` 및 `Tracked` 이벤트를 구독하여 상태 변경 및 추적 이벤트를 구독할 수 있습니다.

```csharp
public interface IHasTimestamps
{
    DateTime? Added { get; set; }
    DateTime? Deleted { get; set; }
    DateTime? Modified { get; set; }
}

private static void UpdateTimestamps(object sender, EntityEntryEventArgs e)
{
    if (e.Entry.Entity is IHasTimestamps entityWithTimestamps)
    {
        switch (e.Entry.State)
        {
            case EntityState.Deleted:
                entityWithTimestamps.Deleted = DateTime.UtcNow;
                Console.WriteLine($"Stamped for delete: {e.Entry.Entity}");
                break;
            case EntityState.Modified:
                entityWithTimestamps.Modified = DateTime.UtcNow;
                Console.WriteLine($"Stamped for update: {e.Entry.Entity}");
                break;
            case EntityState.Added:
                entityWithTimestamps.Added = DateTime.UtcNow;
                Console.WriteLine($"Stamped for insert: {e.Entry.Entity}");
                break;
        }
    }
}
```

| TodoContext.cs
```csharp
public TodoContext()
{
    ChangeTracker.StateChanged += UpdateTimestamps;
    ChangeTracker.Tracked += UpdateTimestamps;
}
```

위의 코드는 이벤트 구독을 통해 인터페이스를 구현하는 모든 엔터티에 대한 타임스탬프를 설정합니다.


## 인터셉터

인터셉터를 이용해 동작 가로채기를 할 수 있습니다. 인터셉터는 작업을 수정하거나 제거할 수 있다는 점에서 로깅 및 진단과 다릅니다.

DbContext의 `OnConfiguring()`에 `AddInterceptors()`를 통해 인터셉터를 추가할 수 있습니다.

```csharp
public class TodoContext : DbContext
{
    ...
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // "database.db" 파일로 SQLite 사용
        optionsBuilder.UseSqlite("Data Source=database.db")
            .AddInterceptors(new TaggedQueryCommandInterceptor());
    }
    ...
}
```

인터셉터는 상태 비저장일 경우 전역으로 생성해 사용할 수도 있습니다.

```csharp
...
    private static readonly TaggedQueryCommandInterceptor _interceptor
        = new TaggedQueryCommandInterceptor();

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        => optionsBuilder.AddInterceptors(_interceptor);
...
```

인터셉터로는 `IInterceptor` 인터페이스를 구현한 인스턴스를 사용할 수 있으며 `DbCommandInterceptor`, `DbConnectionInterceptor`, `DbTransctionInterceptor`를 상속받아 구현할 수도 있습니다.

- DbCommandInterceptor(IDbCommandInterceptor) : 생성, 실행, 명령 실패, DbDataReader 명령 Dispose
- DbConnectionInterceptor(IDbConnectionInterceptor) : 연결 및 닫기, 연결 실패
- DbTransactionInterceptor(IDbTransactionInterceptor) : 생성, 사용, 커밋, 롤벡, 생성 및 세이브포인트 사용, 트렌젝션 실패


## 진단 수신기

진단 수신기를 사용하면 .NET 프로세스에서 발생하는 모든 EF Core 이벤트를 수신할 수 있습니다.

진단 수신기는 프로세스 기준으로 이벤트를 수신하므로 단일 DbContext 인스턴스에 대한 이벤트를 수신하는 데는 적합하지 않습니다.


### 진단 이벤트 관찰

먼저 `IObserver<DiagnosticListener>` 인터페이스를 구현하는 옵버저 클래스를 구현합니다.

```csharp
public class DiagnosticObserver : IObserver<DiagnosticListener>
{
    public void OnCompleted()
        => throw new NotImplementedException();

    public void OnError(Exception error)
        => throw new NotImplementedException();

    public void OnNext(DiagnosticListener value)
    {
        if (value.Name == DbLoggerCategory.Name) // "Microsoft.EntityFrameworkCore"
        {
            value.Subscribe(new KeyValueObserver());
        }
    }
}
```

`OnNext()`에 의해 `Microsoft.EntityFrameworkCore`를 구독하게 됩니다.

둘째 `KeyValueObserver`를 구현합니다.

```csharp
public class KeyValueObserver : IObserver<KeyValuePair<string, object>>
{
    public void OnCompleted()
        => throw new NotImplementedException();

    public void OnError(Exception error)
        => throw new NotImplementedException();

    public void OnNext(KeyValuePair<string, object> value)
    {
        if (value.Key == CoreEventId.ContextInitialized.Name)
        {
            var payload = (ContextInitializedEventData)value.Value;
            Console.WriteLine($"EF is initializing {payload.Context.GetType().Name} ");
        }

        if (value.Key == RelationalEventId.ConnectionOpening.Name)
        {
            var payload = (ConnectionEventData)value.Value;
            Console.WriteLine($"EF is opening a connection to {payload.Connection.ConnectionString} ");
        }
    }
}
```

마지막으로 `DiagnosticListener`를 통해 진단 수신기를 등록합니다.

```csharp
DiagnosticListener.AllListeners.Subscribe(new DiagnosticObserver());
```

| 관찰 예시
```
EF is initializing BlogsContext
EF is opening a connection to Data Source=blogs.db;Mode=ReadOnly
EF is opening a connection to DataSource=blogs.db
EF is opening a connection to Data Source=blogs.db;Mode=ReadOnly
EF is opening a connection to DataSource=blogs.db
EF is opening a connection to DataSource=blogs.db
EF is opening a connection to DataSource=blogs.db
EF is initializing BlogsContext
EF is opening a connection to DataSource=blogs.db
EF is opening a connection to DataSource=blogs.db
```


## 정리

오늘은 EF Core에서 제공하는 다양한 로깅 방식에 대해 알아봤습니다. 좀 더 자세한 내용은 [로깅, 이벤트 및 진단](https://docs.microsoft.com/ko-kr/ef/core/logging-events-diagnostics/) 문서를 살펴보시기 바랍니다.

다음 시간에는 EF Core를 이용해서 간단한 `To Do` 앱을 같이 구현해 보도록 하겠습니다.
