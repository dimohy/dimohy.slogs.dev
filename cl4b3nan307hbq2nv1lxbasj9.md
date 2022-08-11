## EF Core 6 배우기 - 3. DB 컨텍스트

지금까지 EF Core의 개발 환경 구성 및 엔터티에 대해 간략히 살펴봤습니다. 좀 더 자세한 내용은 [마이크로소프트 EF Core 문서](https://docs.microsoft.com/en-us/ef/core/)를 통해 살펴볼 수 있습니다.

오늘은 DB 컨텍스트를 구성하여 간단한 `할 일` 기능을 구현하려고 합니다.


### `할 일` 개체관계 모델링(ERD)

ORM에서도 엔터티간의 관계가 중요하므로 여전히 ERD를 작성하는 것은 중요합니다. 그래서 먼저 `할 일`에 대한 ERD를 작성하는 것으로 시작합시다. ERD는 요구사항을 수집하여 논리 모델을 먼저 완성한 후 이것을 이용해 실제 데이터베이스에 적용할 물리 모델을 완성 하는 것으로 전개하는데요, ERD를 이용해 만든 `할 일` 논리 모델은 다음과 같습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654954133546/TE1F5c8n6.png align="left")

이렇게 `사용자정보`와 `할일정보`의 비식별관계를 이용해 1:N의 관계를 표현할 수 있습니다. 이것을 식별관계로도 표현할 수 있는데요,

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654954384622/8SJslCwHr.png align="left")

엔터티와 엔터티간 관계가 밀접할 경우 이렇게 식별 관계로 표현할 수 있습니다. 위의 예시의 경우 `할일정보`는 복합키를 가지게 됩니다. 저는 비식별관계를 이용해 계속 전개하도록 할께요.

논리 모델이 끝나면 물리 모델을 전개합니다. 대략 다음 모습이 되는데요,

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655022321250/zogoMPaZH.png align="left")

이것을 그대로 엔터티 클래스로 전환해봅시다.

> 여기서 문자 길이는 엔터티 속성에 `MaxLength` 특성을 적용해 지정할 수 있지만 본 글에서는 지정하지 않고 진행 합니다.


### 엔터티

| UserInfo.cs
```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace TodoApp.Entities;

[Table(nameof(UserInfo))]
public record UserInfo
{
    [Key]
    public string UserId { get; set; } = default!;
    public string UserName { get; set; } = default!;

    public virtual ICollection<TodoInfo> Todos { get; } = default!;
}
```

| TodoInfo.cs
```csharp

using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace TodoApp.Entities;

[Table(nameof(TodoInfo))]
public record TodoInfo
{
    [Key]
    public int Seq { get; set; }

    public DateOnly? TodoDate { get; set; }
    public DateOnly? CompleteDate { get; set; }
    public bool IsComplete { get; set; }
    public string? Memo { get; set; }
    public bool IsDel { get; set; }

    [ForeignKey(nameof(User))]
    public string UserId { get; set; } = default!;
    
    public virtual UserInfo User { get; set; } = default!;
}

```

여기서 `UserInfo`의 `Todos`와 `TodoInfo`의 `User`는 탐색 속성이 되며 나중에 질의 시 `Include()`를 통해 조인할 수 있게 됩니다.


### DB 컨텍스트

EF Core에서는 `엔터티`와 `DB 컨텍스트`를 이용해 모델링을 합니다. DB 컨텍스트는 다음의 형태로 만들 수 있습니다.

| TodoContext.cs
```csharp
using Microsoft.EntityFrameworkCore;

using TodoApp.Entities;

namespace TodoApp.DbContexts;

public class TodoContext : DbContext
{
    public DbSet<UserInfo> Users => Set<UserInfo>();
    public DbSet<TodoInfo> Todos => Set<TodoInfo>();

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // "database.db" 파일로 SQLite 사용
        optionsBuilder.UseSqlite("Data Source=database.db");
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Todo의 키인 `Seq`는 자동 증가로 설정
        modelBuilder.Entity<TodoInfo>()
            .Property(x => x.Seq)
            .ValueGeneratedOnAdd();
    }
}
```

`OnConfiguring()`를 통해 SQLite 데이터베이스 파일명을 설정하고 `OnModelCreating()`을 통해 `TodoInfo`의 키인 `Seq` 속성의 값을 자동 증가하도록 설정하고 있습니다. 특성을 이용해 엔터티 및 속성의 특성을 설정 할 수 있지만 생성시 다양한 설정은 `OnModelCreating()에서 제공하는 `Fluent API` 로 상세히 설정할 수 있습니다.

`DBSet<TEntity>`는 LINQ를 이용해 엔터티의 인스턴스를 쿼리하고 저장할 수 있게 해줍니다. 프로젝트에 `<Nullable>` 설정이 되어 있다면 `DbSet<UserInfo> { get; set; }` 대신 `DbSet<UserInfo> Users => Set<UserInfo>()`로 널 관련 경고를 없앨 수 있습니다.


### 마이그레이션 및 데이터베이스 업데이트

`dotnet CLI 도구`가 잘 설치되어 있다면 다음의 명령을 통해 마이그레이션 할 수 있습니다.

```
dotnet ef migrations add first
```

`first`의 이름으로 마이그레이션을 수행 하며 `엔터티` 및 `DB 컨텍스트`를 수정하여 데이터베이스에 반영해야 할 때 적절한 이름으로 마이그레이션을 수행할 수 있습니다.

정상적으로 마이그레이션이 마이그레이션 내용을 데이터베이스에 업데이트 할 수 있습니다.

```
dotnet ef database update
```

데이터베이스에 업데이트가 정상 수행되면 다음 처럼 데이터베이스에 반영이 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655024283825/zzL7hjwzH.png align="left")


### 쿼리

이제 다양한 쿼리 테스트를 해봅시다. 쿼리는 `DB 컨텍스트`를 통해 이뤄지므로 먼저 `DB 컨텍스트` 인스턴스를 생성해야 합니다.

```csharp
using Microsoft.EntityFrameworkCore;

using TodoApp.DbContexts;

using var c = new TodoContext();
```


#### 사용자 추가

그런 후 `dimohy`와 `test` 계정을 생성해봅시다.

```csharp
c.Users.Add(new()
{
    UserId = "dimohy",
    UserName = "디모이"
});

c.Users.Add(new()
{
    UserId = "test",
    UserName = "테스트"
});

c.SaveChanges();
```

데이터베이스에 다음과 같이 저장되었습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655024450473/bUSa4v_Vf.png align="left")

실제 쿼리를 해봅시다. `Users`의 모든 사용자를 조회할 것이므로 코드는 다음과 같습니다.

```csharp
foreach (var user in c.Users)
{
    Console.WriteLine(user);
}
```

| 결과
```
UserInfo { UserId = dimohy, UserName = 디모이, Todos =  }
UserInfo { UserId = test, UserName = 테스트, Todos =  }
```


#### 할 일 추가

이제 할 일을 추가해 봅시다.

```csharp
c.Todos.Add(new()
{
    UserId = "dimohy",

    TodoDate = new DateOnly(2022, 6, 12),
    Memo = "`EF Core 6 배우기 - 3. DB 컨텍스트` 완료",
});

c.Todos.Add(new()
{
    UserId = "test",

    TodoDate = new DateOnly(2022, 6, 12),
    Memo = "삽입 테스트",
});

c.SaveChanges();
```

데이터베이스에 다음과 같이 저장되었습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655024700785/IvUKp7yRs.png align="left")


#### 사용자로 할 일 조회

EF Core는 탐색 속성을 통해 엔터티간 관계 조인을 통해 쿼리를 할 수 있습니다. 사용자로 할 일을 조회해봅시다.

```csharp
var dimohy = c.Users.Include(x => x.Todos).First(x => x.UserId == "dimohy");
Console.WriteLine(dimohy);

foreach (var todo in dimohy.Todos)
{
    Console.WriteLine(todo);
}
```

| 결과
```
UserInfo { UserId = dimohy, UserName = 디모이, Todos = System.Collections.Generic.HashSet`1[TodoApp.Entities.TodoInfo] }
TodoInfo { Seq = 1, TodoDate = 2022-06-12, CompleteDate = , IsComplete = False, Memo = `EF Core 6 배우기 - 3. DB 컨텍스 트` 완료, IsDel = False, UserId = dimohy, User = UserInfo { UserId = dimohy, UserName = 디모이, Todos = System.Collections.Generic.HashSet`1[TodoApp.Entities.TodoInfo] } }
```

#### 할 일로 사용자 조회

반대로 `할 일`에서 사용자를 확인해봅시다.

```csharp
ar todos = c.Todos.Where(x => x.UserId == "dimohy").Include(x => x.User);
foreach (var todo in todos)
{
    Console.WriteLine(todo);
}
```

| 결과
```
TodoInfo { Seq = 1, TodoDate = 2022-06-12, CompleteDate = , IsComplete = False, Memo = `EF Core 6 배우기 - 3. DB 컨텍스 트` 완료, IsDel = False, UserId = dimohy, User = UserInfo { UserId = dimohy, UserName = 디모이, Todos = System.Collections.Generic.HashSet`1[TodoApp.Entities.TodoInfo] } }
```


#### 할 일 속성 변경 및 데이터베이스 반영

```csharp
var testTodo = c.Todos.First(x => x.UserId == "test");
testTodo.IsComplete = true;
testTodo.CompleteDate = DateOnly.FromDateTime(DateTime.Now);
c.SaveChanges();
```

데이터베이스에 다음과 같이 반영되었습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655025347450/sTO2WjoSi.png align="left")


### 정리

EF Core에서 DB 컨텍스트 및 엔터티 클래스를 이용해 데이터베이스 생성 및 쿼리를 살펴보았습니다. 다음 시간에는 마이그레이션에 대해 좀 더 알아보도록 합시다.


### 소스코드

https://github.com/dimohy/efcore-learning/tree/main/3.%20DB%20%EC%BB%A8%ED%85%8D%EC%8A%A4%ED%8A%B8/TodoApp