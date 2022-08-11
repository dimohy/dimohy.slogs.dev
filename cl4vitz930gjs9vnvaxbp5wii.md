## EF Core 6 배우기 - 5. 변경 추적 기능

EF Core는 변경 내용을 추적하는 기능을 제공 합니다. 이 기능을 이용해서 DbContext의 `SaveChanges`가 호출되었을 때 변경된 내용만 데이터베이스에 반영되게 됩니다.


## 엔터티를 추적하는 방법

EF Core가 변경을 추적하는 유형은 다음과 같습니다.

- 데이터베이스 쿼리 반환
- `Add`, `Attach`, `Update` 또는 유사한 메서드를 통해 명시적으로 DbContext에 연결
- 추적된 기존 엔터티에 연결된 엔터티로 검색할 경우

다음은 추적이 종료되는 경우 입니다.

- DbContext 삭제
- 변경 내용 추적기가 제거된 경우
- 엔터티가 명시적으로 분리된 경우

DbContext는 일시적인 작업 단위로 설계되었으므로 엔터티 추적을 종료하는 가장 일반적인 방법은 DbContext를 삭제하는 것입니다. DbContext의 수명은 일반적으로 다음과 같습니다.

- DbContext 인스턴스 생성
- 엔터티 추적
- 엔터티 변경
- SaveChanges를 통해 엔터티 추적 변경 내용 반영
- DbContext 인스턴스 삭제


## 엔터티 상태

엔터티의 변경을 추적하기 위해 엔터티는 상태를 가집니다. [EntityState](https://docs.microsoft.com/ko-KR/dotnet/api/microsoft.entityframeworkcore.entitystate?view=efcore-6.0)를 통해 엔터티의 상태가 관리됩니다.

- `Detached`는 추적되지 않습니다.
- `Added`는 삽입되어야 할 추적 대상이 있음을 의미합니다.
- `Unchanged`는 쿼리 시점 이후에 변경되지 않았음을 의미합니다.
- `Modified`는 쿼리 후 변경되었음을 의미합니다. 따라서 `SaveChanged`시 변경된 내용이 업데이트 됩니다.
- `Deleted`는 데이터베이스에는 존재하지만 추적에 의해 대상이 삭제되었음을 의미하며 `SaveChanged` 호출 시 삭제됩니다.


## 동작에 따른 상태 변화

변경 내용 추적은 쿼리 시점과 반영 시점이 동일한 DbContext 인스턴스를 사용할 때 가장 이상적으로 동작합니다. 변경 내용 추적은 DbContext가 삭제될 때 같이 소멸되기 때문입니다. 쿼리 및 쿼리 대상을 수정하여 업데이트 하는 것 뿐만 아니라 쿼리 후 삽입, 업데이트, 삭제를 순차적으로 했을 경우에도 해당합니다. 엔터티는 내부적으로 `EntityState`를 이용해 순차적으로 추적 내용을 업데이트 하며 최종 상태를 이용해 `SaveChanges`시점에서 데이터베이스에 변경 내용을 업데이트 한 후 상태를 `Unchanged`로 변경합니다.

이를 기존 프로젝트를 이용해 확인해보도록 합시다.

```csharp
using TodoApp.DbContexts;

using var c = new TodoContext();

var result = c.SaveChanges();
Console.WriteLine(result);
```

|  결과
```
0
```

`DBContext` 인스턴스를 생성하고 어떠한 작업 없이 `SaveChanges()`를 호출하면 변경 추적 내용이 없으므로 결과로 `0`을 반환합니다.

새로운 사용자를 추가해 봅시다.

```csharp
var newUser = new UserInfo
{
    UserId = "test",
    UserName = "test"
};
```

newUser는 아직은 DbContext에 연결된 인스턴스가 아니므로 상태를 확인했을 때 `Detached`임을 알 수 있습니다.

```csharp
using TodoApp.DbContexts;
using TodoApp.Entities;

using var c = new TodoContext();

var newUser = new UserInfo
{
    UserId = "test",
    UserName = "test"
};

Console.WriteLine(c.Entry(newUser).State);

var result = c.SaveChanges();
Console.WriteLine(result);
```

| 결과
```
Detached
0
```

`newUser`를 `c.Users`에 `Add`했을 때 상태가 `Added`로 변경됩니다.

```csharp
c.Users.Add(newUser);

Console.WriteLine(c.Entry(newUser).State);
```

| 결과
```
Added
```

이제 `SaveChanges`를 호출해서 상태가 어떻게 변하는지를 확인해 봅시다.

```csharp
using TodoApp.DbContexts;
using TodoApp.Entities;

using var c = new TodoContext();

var newUser = new UserInfo
{
    UserId = "test",
    UserName = "test"
};

c.Users.Add(newUser);

Console.WriteLine(c.Entry(newUser).State);

var result = c.SaveChanges();
Console.WriteLine(result);

Console.WriteLine(c.Entry(newUser).State);
```

| 결과
```
Added
1
Unchanged
```

`SaveChanges` 호출에 의해 데이터베이스에 업데이트 되고 반영된 갯수가 1임을 확인할 수 있었고 상태가 `Unchanged`로 변경되었음을 확인할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656253961797/CSNfGZHVg.png align="left")

이제 저장되어 있는 `test` 사용자를 찾아 이름을 `테스트`로 변경해봅시다.

```csharp
var testUser = c.Users.Find("test")!;
Console.WriteLine(testUser);

Console.WriteLine(c.Entry(testUser).State);

testUser.UserName = "테스트";

Console.WriteLine(c.Entry(testUser).State);

var result = c.SaveChanges();
Console.WriteLine(result);

Console.WriteLine(c.Entry(testUser).State);
```

| 결과
```
UserInfo { UserId = test, UserName = test, Todos =  }
Unchanged
Modified
1
Unchanged
UserInfo { UserId = test, UserName = 테스트, Todos =  }
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656254522914/sI-PJK7eL.png align="left")

이제 추가된 사용자를 삭제해봅시다.

```csharp
var testUser = c.Users.Find("test")!;

Console.WriteLine(c.Entry(testUser).State);

c.Users.Remove(testUser);

Console.WriteLine(c.Entry(testUser).State);

var result = c.SaveChanges();
Console.WriteLine(result);

Console.WriteLine(c.Entry(testUser).State);

testUser = c.Users.Find("test")!;
Console.WriteLine(testUser);
```

| 결과
```
Unchanged
Deleted
1
Detached
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656254720179/jRz6AjU5U.png align="left")

각각의 동작에 따라 상태가 `Detached`, 'Unchanged', 'Added, 'Modified', 'Deleted' 됨을 확인할 수 있습니다.


## 탐색 속성의 상태 변경 추적

변경에 대한 추적은 일반 엔터티 및 일반 속성에만 해당되지 않습니다. `test`라는 사용자를 추가 한 후 다음의 코드를 실행해봅시다.

```csharp
var testUser = c.Users.Find("test")!;
Console.WriteLine(testUser);
Console.WriteLine(testUser.Todos);
```

| 결과
```
UserInfo { UserId = test, UserName = 테스트, Todos =  }

```

탐색 속성인 `Todos`는 `null`입니다. 질의를 수정해봅시다. 

```csharp
var testUser = c.Users.Include(x => x.Todos).First(x => x.UserId == "test");
Console.WriteLine(testUser);
Console.WriteLine(testUser.Todos);
```

| 결과
```
UserInfo { UserId = test, UserName = 테스트, Todos = System.Collections.Generic.HashSet`1[TodoApp.Entities.TodoInfo] }
System.Collections.Generic.HashSet`1[TodoApp.Entities.TodoInfo]
```

이제 결과가 다르게 나옵니다. `Include`를 이용해 `Join` 질의를 했으며 이렇게 탐색 속성과 함께 질의를 했을 때 탐색 속성도 상태 변경 추적의 대상이 됩니다.


#### 할 일과 태그를 동시에 업데이트 하기

다음의 코드를 통해 `test`사용자의 `할 일`과 `할 일`에 태그를 기록해서 데이터베이스에 업데이트 해봅시다.

```csharp
var testUser = c.Users.Include(x => x.Todos).First(x => x.UserId == "test");
var newTodo = new TodoInfo
{
    //UserId = "test",
    TodoDate = DateOnly.FromDateTime(DateTime.Now),
    IsComplete = false,
    IsDel = false,
    Memo = "탐색 속성 변경 추적 확인",
    Tags = new TodoTagInfo[] { new() { TagId = "dev", Descption = "개발" }, new() { TagId = "test", Descption = "테스트" } }
};

testUser.Todos.Add(newTodo);

var result = c.SaveChanges();
Console.WriteLine(result);
```

| 결과
```
5
```

| TodoInfo 테이블
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656256267722/qcgqhVPYg.png align="left")

| TodoTagInfo 테이블
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656256300627/N0X9DaZQG.png align="left")

| TodoInfoTodoTagInfo 테이블 (다대다 탐색 속성에 의한 자동 생성 테이블)
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656256337501/etOEMEkfG.png align="left")

코드는 단지 하나의 `TodoInfo`를 삽입했을 뿐인데 총 5개의 업데이트가 이루어졌는데요, 이것은 탐색 속성 또한 상태 추적의 대상이 되기 때문입니다.

잘 업데이트 되었는지 다음의 코드를 통해 확인해봅시다.

```csharp
var testUser = c.Users.Include(x => x.Todos).First(x => x.UserId == "test");
foreach (var testTodos in testUser.Todos)
{
    Console.WriteLine(testTodos);
}
```

| 결과
```
TodoInfo { Seq = 2, TodoDate = 2022-06-27, CompleteDate = , IsComplete = False, Memo = 탐색 속성 변경 추적 확인, IsDel = False, UserId = test, User = UserInfo { UserId = test, UserName = 테스트, Todos = System.Collections.Generic.HashSet`1[TodoApp.Entities.TodoInfo] }, Histories = , Tags =  }
```

기대하는 `Tags`에 아무런 값이 없는데요, `Include`할 대상을 알아서 넣어주지는 않습니다. 다음처럼 변경해봅시다.

```csharp
var testUser = c.Users.Include(x => x.Todos).ThenInclude(y => y.Tags).First(x => x.UserId == "test");
foreach (var testTodos in testUser.Todos)
{
    Console.WriteLine(testTodos.Memo);
    foreach (var tag in testTodos.Tags)
        Console.WriteLine($"Tags : {tag.TagId} - {tag.Descption}");
}
```

> #### 데이터 지연 로드
> 질의 시 필요한 데이터를 모두 질의 하는 방법도 있지만 지연로드를 이용해 필요한 시점에서 (속성이 읽혀지는 시점에서) 추가 질의를 통해 데이터를 가져올 수도 있습니다. 이를 활성화 하려면 아래의 지침을 따릅니다.
> - `Microsoft.EntityFrameworkCore.Proxies` 패키지 추가
> - `DbContext`의 `OnConfiguring()`에 `.UserLazyLoadingProxies()` 추가
> ```csharp
> protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
>     => optionsBuilder
>         .UseLazyLoadingProxies()
>         .UseSqlServer(myConnectionString);
> ```
> - 또는 AddDbContext를 사용하는 경우:
> ```csharp
> .AddDbContext<BloggingContext>(
>     b => b.UseLazyLoadingProxies()
>           .UseSqlServer(myConnectionString));
> ```
> 이를 통해 `virtual` 탐색 속성은 지연된 로드를 사용하게 됩니다. 또는 `ILazyLoader` 서비스 참조를 통해 지연로드를 수동으로 사용할 수 도 있습니다.


## 연결되지 않은 엔터티 추적

DbContext 인스턴스를 생성해서 수행하는 추적 기능은 상당히 유용하지만 대부분의 HTTP 요청 웹 서비스의 경우 서버와 클라이언트의 데이터의 주고 받음에 의해 DbContext와의 연결이 끊어진 상태가 됩니다. 이때 끊어진 추적을 시작하려면 다음과 같이 수행합니다.

```csharp
context.Attach(
    new Blog { Id = 1, Name = ".NET Blog", });
```

> 위의 예시는 단순성을 위해 `new`를 사용하였지만 실제로는 클라이언트에서 수신된 데이터를 이용하게 됩니다.


## 변경 내용 추적기 디버깅

변경 내용에 대한 추적 내용은 [ChangeTracker](https://docs.microsoft.com/ko-kr/dotnet/api/microsoft.entityframeworkcore.changetracking.changetracker?view=efcore-6.0)을 통해 얻을 수 있습니다. [Tracked](https://docs.microsoft.com/ko-kr/dotnet/api/microsoft.entityframeworkcore.changetracking.changetracker.tracked?view=efcore-6.0) 이벤트를 통해 추적된 내용이 변경되었을 때 이벤트 수신을 통해 내용을 확인할 수 있습니다. 또한 디버그 모드에 유용한 [ChangeTracker.DebugView](https://docs.microsoft.com/ko-KR/dotnet/api/microsoft.entityframeworkcore.changetracking.changetracker.debugview?view=efcore-6.0)를 통해 추적된 이력을 확인할 수 있습니다.


## 정리

오늘은 EF Core의 변경 내용 추적에 대해 살펴보았습니다. EF Core 문서 [변경 내용 추적](https://docs.microsoft.com/ko-kr/ef/core/change-tracking/)을 통해 좀 더 상세한 내용을 살펴보실 수 있습니다. 다음 시간에는 EF Core가 생성하는 쿼리를 로그를 통해 확인하는 방법을 살펴보도록 하겠습니다.

https://github.com/dimohy/efcore-learning/tree/main/5.%20%EB%B3%80%EA%B2%BD%20%EC%B6%94%EC%A0%81%20%EA%B8%B0%EB%8A%A5/TodoApp


