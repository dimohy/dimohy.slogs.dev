## EF Core 6 배우기 - 2. 엔터티

EF Core는 엔터티 클래스로 ORM의 규칙을 적용해 모델을 작성합니다. 규칙은 특성을 통해 부여하거나 EF Core에서 약속한 구성으로 만들 수 있습니다. 또한 DB 컨텍스트에서 Fluent API 방식으로 규칙을 정의할 수도 있습니다.

## 엔터티

엔터티는 컨텍스트에 DBSet으로 포함되면 EF Core의 모델에 포함됩니다. 일반적으로 이러한 형식을 엔터티라 합니다. 엔터티는 데이터베이스에 적용될 때 테이블 단위가 됩니다.

| 엔터티 클래스 정의
```csharp
public class UserInfo
{
...
}
```

| 엔터티 클래스를 엔터티로 등록
```csharp
public class MyContext : DbContext
{
...
    public DbSet<UserInfo> Users { get; set; }
```

이외에 엔터티가 되는 경우는 다음과 같습니다.

- 엔터티에 탐색 속성으로 정의될 경우
```csharp
public class UserInfo
{
...
    public List<UserHistory> Histories { get; set; }
...
```
`UserHistory`는 `UserInfo`와 1대 N의 관계가 형성되어 탐색 대상이 되기 때문에 엔터티 입니다.

- 컨텍스트의 `OnModelCreating()`에 엔터티로 지정될 경우
```csharp
public class MyContext : DbContext
{
...
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<UserAuth>();
    }
```
`UserAuth`는 컨텍스트의 `OnModelCreating()`에서 엔터티로 지정되었기 때문에 엔터티 입니다.

## 엔터티 규칙
엔터티를 클래스로 표현하기 위해 특성 및 Fluent API를 이용해 규칙을 지정할 수 있습니다.

### 키
키는 엔터티를 식별할 수 있는 기준입니다. 규칙에 따라 다음은 엔터티의 키가 됩니다.

- 속성 이름이 `Id`일 경우
```csharp
public class UserInfo
{
    public string Id { get; set; } // Id는 UserInfo의 키
    ...
}
```

- 속성 이름이 `엔터티명 + "Id"`일 경우
```csharp
public class UserInfo
{
    public string UserId { get; set; } // UserId는 UserInfo의 키
    ...
}
```

- `[Key]` 특성을 이용해 속성을 키로 지정
```csharp
public string UserHistory
{
    [Key]
    public int Seq { get; set; } // Seq는 UserHistory의 키
    ....
}
```

- 컨텍스트에서 키로 지정
컨텍스트에서 Fluent API 방식으로 해당 엔터티의 키를 지정할 수 도 있습니다.
```csharp
public class MyContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Blog>()
            .HasKey(b => b.BlogId); // Blog 엔터티의 BlogId를 키로 지정
    }
```

### 외래키

EF Core에서 외래키는 Fluent API로 직접 지정할 수도 있지만 엔터티의 탐색 속성을 이용해 1:1 및 1:N의 관계를 나타낼 수 있어서 데이터 질의 시 유용하므로 이를 추천합니다.

| 1:1 관계
```csharp
public class UserInfo
{
    [Key]
    public string Id { get; set; }

    public UserInfo(string id)
    {
        Id = id;
    }

    public UserDetail Detail { get; set; } = default!;
}

public class UserDetail
{
    [Key]
    [ForeignKey(nameof(User))]
    public string Id { get; set; } = default!;

    public string Address { get; set; } = default!;

    public UserInfo User { get; set; } = default!;
}
```

여기서 `UserInfo`의 `Detail` 속성은 탐색 속성이 됩니다. `UserDetail`의 `User` 속성은 관계 속성이 됩니다. EF Core에서는 관계 속성과 탐색 속성을 통해 관계된 다양한 질의를 할 수 있게 됩니다.

| 1:N 관계
```csharp
public class UserInfo
{
    [Key]
    public string Id { get; set; }

    public UserInfo(string id)
    {
        Id = id;
    }

    public IList<Todo> Todos { get; set; } = default!;
}

public class Todo
{
    [Key]
    public int Seq { get; set; }

    public UserInfo User { get; set; } = default!;
}
```

`할 일`을 예로 들어 봅시다. 사용자 한명은 여러 개의 `할 일`을 가질 수 있으므로 `UserInfo`와 `Todo`는  1:N 관계 입니다. 이는 `UserInfo`의 `Todos` 탐색 속성을 통해 목록으로 정의할 수 있고 질의할 수 있습니다. 하나의 할 일은 사용자 한명에게 할당되므로  `Todo`에 `User` 관계 속성으로 정의하고 질의할 수 있습니다.

이런 엔터티 관계를 통해 다음의 SQLite 테이블 및 컬럼이 생성됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1653961022693/n3U23PdbT.png align="left")

### 정리
EF Core의 엔터티에 대해 간략히 소개하였습니다. 좀 더 자세한 내용은 마이크로소프트  문서의 [모델 만들기](https://docs.microsoft.com/ko-kr/ef/core/modeling/)를 통해 살펴보실 수 있습니다.

다음 시간에는 `할 일` 모델링을 EF Core를 이용해 엔터티와 DB 컨텍스트로 구성해서 콘솔 애플리케이션으로 그 동작을 같이 확인해보도록 합시다.
