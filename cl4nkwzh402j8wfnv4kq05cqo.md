---
title: "EF Core 6 배우기 - 4. 마이그레이션"
datePublished: Tue Jun 21 2022 03:00:28 GMT+0000 (Coordinated Universal Time)
cuid: cl4nkwzh402j8wfnv4kq05cqo
slug: ef-core-6-4
tags: dotnet, efcore

---

데이터 모델은 기능이 구현됨에 따라 계속해서 변화됩니다. 새로운 엔터티가 추가되거나 속성이 추가, 변경 및 제거 되고 엔터티가 변경됨에 따라 애플리케이션과 동기화 되어야 할 데이터베이스 스키마 역시 변경되어야 합니다. EF Core에서는 마이그레이션 기능을 제공해서 데이터 모델의 변경을 추적해서 관리할 수 있고 변경된 내용을 데이터베이스 스키마와 동기화 하는 기능을 제공 합니다.

## 시작하기

앞 전에 만들었던 `할 일` 모델을 확장해 봅시다. 할 일을 변경했을 때 변경 이력을 기록하고, 할 일에 대한 태그를 기록할 수 있도록 해봅시다.

| TodoChangedHistory.cs, 할 일 변경 이력
```csharp
    [Table(nameof(TodoChangedHistory))]
    public class TodoChangedHistory
    {
        [Key]
        [ForeignKey(nameof(Todo))]
        public int TodoSeq { get; set; }
        [Key]
        public int Seq { get; set; }

        public TodoChangedKind ChangedKind { get; set; }
        public string Before { get; set; } = default!;
        public string After { get; set; } = default!;

        public TodoInfo Todo { get; set; } = default!;
    }

    public enum TodoChangedKind
    {
        할일날짜변경 = 1,
        완료날짜변경 = 2,
        완료유무변경 = 3,
        메모변경 = 4,
        삭제유무변경 = 5,
    }
```

| TodoTagInfo.cs, 할일 태그
```csharp
    [Table(nameof(TodoTagInfo))]
    public class TodoTagInfo
    {
        [Key]
        public string TodoId { get; set; } = default!;

        public string Descption { get; set; } = "";

        public virtual ICollection<TodoInfo> Todos { get; set; } = default!;
    }
```

| TodoInfo.cs, 탐색 속성 추가
```
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

    public virtual ICollection<TodoChangedHistory> Histories { get; set; } = default!;
    public virtual ICollection<TodoTagInfo> Tags { get; set; } = default!;
}
```

| TodoContext.cs, 수정
```csharp
public class TodoContext : DbContext
{
    public DbSet<UserInfo> Users => Set<UserInfo>();
    public DbSet<TodoInfo> Todos => Set<TodoInfo>();
    public DbSet<TodoChangedHistory> TodoChangedHistories => Set<TodoChangedHistory>();
    //public DbSet<TagInfo> Tags => Set<TagInfo>();
    public DbSet<TodoTagInfo> TodoTags => Set<TodoTagInfo>();

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // "database.db" 파일로 SQLite 사용
        optionsBuilder.UseSqlite("Data Source=database.db");
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Todo의 `Seq`키는 자동 증가
        modelBuilder.Entity<TodoInfo>()
            .Property(x => x.Seq)
            .ValueGeneratedOnAdd();

        // TodoChangedHistory는 복합키(TodoSeq, Seq)
        modelBuilder.Entity<TodoChangedHistory>()
            .HasKey(x => new { x.TodoSeq, x.Seq });
    }
}
```

이것을 ERD로 표현하면 다음과 같습니다. (키 속성만 표현하였습니다)

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655778272342/nZD-cvKrH.png align="left")

흥미로운 점은 `TodoInfo`와 `TodoTagInfo`가 탐색 속성으로 인해 `다 대 다 관계`로 표현되었다는 점입니다.

UserInfo.cs
> public virtual ICollection<TodoTagInfo> Tags { get; set; } = default!;

TodoTagInfo.cs
> public virtual ICollection<TodoInfo> Todos { get; set; } = default!;

EF Core는 데인터 모델을 마이그레이션 할 때 다음과 같이 `다 대 다 관계`를 위한 테이블을 자동 생성해줍니다.

| 마이그레이션
```
$ dotnet ef migrations add 220621
```

|  Migrations/20220621022016_220621.cs, 마이그레이션 명령에 의해 생성된 파일
```csharp
...
            migrationBuilder.CreateTable(
                name: "TodoInfoTodoTagInfo",
                columns: table => new
                {
                    TagsTagId = table.Column<string>(type: "TEXT", nullable: false),
                    TodosSeq = table.Column<int>(type: "INTEGER", nullable: false)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_TodoInfoTodoTagInfo", x => new { x.TagsTagId, x.TodosSeq });
                    table.ForeignKey(
                        name: "FK_TodoInfoTodoTagInfo_TodoInfo_TodosSeq",
                        column: x => x.TodosSeq,
                        principalTable: "TodoInfo",
                        principalColumn: "Seq",
                        onDelete: ReferentialAction.Cascade);
                    table.ForeignKey(
                        name: "FK_TodoInfoTodoTagInfo_TodoTagInfo_TagsTagId",
                        column: x => x.TagsTagId,
                        principalTable: "TodoTagInfo",
                        principalColumn: "TagId",
                        onDelete: ReferentialAction.Cascade);
                });

            migrationBuilder.CreateIndex(
                name: "IX_TodoInfoTodoTagInfo_TodosSeq",
                table: "TodoInfoTodoTagInfo",
                column: "TodosSeq");
...
```

EF Core에서는 내부적으로 이 테이블을 사용하며 할 일 정보와 태그 정보를 `다대다 관계`로 유지합니다.

| 데이터베이스 확인

```
$ dotnet ef database update
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655778777846/vi_75VT8N.png align="left")

## 마이그레이션 관리
EF Core에서 다음과 같은 다양한 마이그레이션 기능을 이용할 수 있습니다.

### 마이그레이션 추가
데이터 모델이 변경된 경우 변경 사항에 대한 마이그레이션을 추가할 수 있습니다.

```
$ dotnet ef migrations add AddBlogCreatedTimestamp
```

### 마이그레이션 제거
데이터베이스 동기화를 하기 전에 마이그레이션을 제거해야 할 때도 있습니다.

```
$ dotnet ef migrations remove
```

이미 데이터베이스에 스키마와 동기화가 된 경우에는 마이그레이션을 제거하면 안됩니다. 수정사항을 복구한 후 마이그레이션을 추가해야 합니다.

### 마이그레이션 나열
다음의 명령으로 마이그레이션 목록을 확인할 수 있습니다.

```
$ dotnet ef migrations list
```

### 모든 마이그레이션 재설정
마이그레이션 목록이 상당히 누적되어 더이상 마이그레이션 이력이 의미가 없을 경우 모든 마이그레이션을 삭제할 필요가 있습니다. 이런 경우 가장 간단한 방법은 마이그레이션 파일을 모두 삭제하고 데이터베이스를 삭제한 후 첫번째 마이그레이션을 추가 해서 데이터베이스에 업데이트 하는 것입니다.

하지만 데이터를 삭제하지 않고 다음의 절차로 마이그레이션을 재설정 할 수 있습니다.
- 마이그레이션 폴더 삭제
- 새 마이그레이션 추가, 이에 대한 SQL 스크립트를 생성 함
- 데이터베이스에서 마이그레이션 기록 테이블(__EFMigrationsHistory) 삭제
- 테이블이 이미 존재하므로 첫번째 마이그레이션이 적용되었음을 마이그레이션 기록 테이블에 추가

### SQL 스크립트 생성
SQL 스크립트를 통해 마이그레이션의 최종 형태를 배포할 수 있습니다.

```
$ dotnet ef migrations script
```

다음은 주어진 마이그레이션에서 최신 마이그레이션으로의 차이만 SQL 스크립트로 기록합니다.

```
$ dotnet ef migrations script AddNewTables
```

어디부터 어디까지의 마이그레이션 내용을 SQL 스크립트로 기록할지는 다음처럼 할 수 있습니다.

```
$ dotnet ef migrations script AddNewTables AddAuditTable
```

### 멱등성 SQL 스크립트 생성

EF Core는 마이그레이션 기록 테이블을 이용해 마이그레이션 해야 할 내용을 누적해서 적용할 수 있습니다.

```
$ dotnet ef migrations script --idempotent
```

###  데이터베이스 스키마 동기화
데이터베이스에 마이그레이션을 적용하려면 다음의 명령을 이용합니다.

```
$ dotnet ef database update
```

다음은 지정한 마이그레이션으로 업데이트 합니다.

```
$ dotnet ef database update AddNewTables
```

주의해야 할 것은 마이그레이션을 적용했을 때 테이블의 구조 변경에 의해 데이터가 유실 될 수 있다는 점입니다.

### 번들 생성
EF Core 6에서 도입된 것으로 단지 SQL 스크립트를 생성하는게 아니라 실행할 수 있는 번들을 생성합니다. 이는 다양한 예외 상황을 해결합니다.

- SQL 스크립트를 실행하려면 도구가 필요
- 트랜잭션 처리 시 오류가 발생할 경우에 대한 동작은 일관성이 필요함
- 배포 프로세스의 일부로 쉽게 실행 가능
- .NET SDK 또는 EF 도구를 설치하지 않아도 실행되며 프로젝트 소스코드가 필요하지 않음

```
$ dotnet ef migrations bundle
```

만약 자체 포함의 리눅스용 번들을 생성하려면 다음과 같이 할 수 있습니다.

```
$ dotnet ef migrations bundle --self-contained -r linux-x64
```

### 런타임에 마이그레이션 적용
다음과 유사한 코드를 통해 `db.Database.Migrate()`를 호출함으로써 마이그레이션을 할 수 있습니다. 이 방식은 테스트에는 유용하지만 프로덕션에는 추천하지 않습니다.

```
public static void Main(string[] args)
{
    var host = CreateHostBuilder(args).Build();

    using (var scope = host.Services.CreateScope())
    {
        var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        db.Database.Migrate();
    }

    host.Run();
}
```

## 정리
오늘은 EF Core의 마이그레이션에 대해 살펴봤습니다. 데이터 모델을 데이터베이스에 적용하기 위해 마이그레이션 및 데이터베이스 동기화 작업이 필요하므로 번거롭게 느껴질 수 있지만 개발이 진행됨에 있어서 데이터 모델 관련 코드와 데이터베이스 스키마가 지속적으로 관리되므로 점진적으로 기능을 확장하는데 되려 유리하다고 생각합니다.

다음시간에는 EF Core의 변경 추적 기능에 대해 살펴보도록 하겠습니다.

https://github.com/dimohy/efcore-learning/tree/main/4.%20%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98/TodoApp