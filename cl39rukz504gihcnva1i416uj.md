## EF Core 6 배우기 - 1. 시작하기

[Entity Framework Core](https://docs.microsoft.com/ko-kr/ef/core/) (줄여서 EF Core)는 마이크로소프트에서 개발한 .NET(Core)용 ORM 프레임워크로 [Entity Framework](https://en.wikipedia.org/wiki/Entity_Framework)의 경험으로 새롭게 재개발 되었습니다.
[ORM(Object-relational mapping)](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping)을 이용하면 SQL 쿼리를 사용하지 않고 DBMS에 종속적인 코드를 최소화 할 수 있습니다.

EF Core 6에 이르러서 데이터베이스를 마이그레이션하기 위한 별도의 실행형 번들을 지원하고 `ConfigureConventions()` 메소드를 오버라이드 하는 것으로 다양한 관례를 일괄 적용할 수 있는 방법을 제공하며, 컴파일된 모델 지원과 [Dapper](https://github.com/DapperLib/Dapper)에 근접한 성능 및 효율적인 SQL 쿼리 생성 등 많은 개선사항이 포함되었습니다. 이제 EF Core를 이용해 EF Core를 지원하는 DBMS를 이용해 ORM을 충분히 현업에서 사용할 수 있습니다.

이번 장은 콘솔 프로젝트에서 EF Core 개발 환경을 구성하는 방법을 설명합니다.

## 개발 환경
- Visual Studio 2022
- .NET 6

## EF Core 패키지 추가
[NuGet](https://ko.wikipedia.org/wiki/NuGet)을 이용하면 쉽게 EF Core 패키지를 추가할 수 있습니다.

먼저 적절한 프로젝트 디렉토리에서 콘솔 프로젝트를 생성한 후

```shell
dotnet new console
```

`SQLite`를 사용하는 구성으로 EF Core 패키지를 추가합니다.

```shell
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
```

`Microsoft.EntityFrameworkCore.Sqlite`만 추가하면 `SQLite`를 이용한 EF Core의 패키지가 모두 추가됩니다.

## 마이그레이션 도구 설치

EF Core로 만든 ORM 모델은 결국 데이터베이스 스키마로 변환 되어야 합니다. EF Core에서 제공하는 `dotnet CLI 도구` 또는 `패키지 관리자 콘솔 도구`를 사용할 수 있는데 여기서는 `dotnet CLI 도구`를 사용하는 방법을 소개합니다.

`dotnet tool install` 명령을 통해 `dotnet-ef` 도구를 전역 설치합니다.

```shell
dotnet tool install --global dotnet-ef
```

이후 `dotnet tool update`로 최신 버젼으로 업데이트 할 수 있습니다.

```shell
dotnet tool update --global dotnet-ef
```

이제 이 도구를 사용할 프로젝트 디렉토리에서 `Microsoft.EntityFrameworkCore.Design` 패키지를 추가합니다.

```shell
dotnet add package Microsoft.EntityFrameworkCore.Design
```

## 모델 생성

EF Core의 동작을 빠르게 확인하기 위해 간단한 엔터티를 추가해 보도록 합시다.

| Entities/LogHistory.cs
```csharp
using System.ComponentModel.DataAnnotations;

namespace EFCoreFirstApp.Entities;
public class LogHistory
{
    [Key]
    public int Seq { get; set; }
    public string Detail { get; set; }
    public DateTime CreateTime { get; set; } = DateTime.Now;

    public LogHistory(string detail)
    {
        Detail = detail;
    }
}
```

다음으로 DB 컨텍스트를 생성해서 모델을 완성합니다.

| DbContexts/FirstAppContext.cs
```csharp
using EFCoreFirstApp.Entities;

using Microsoft.EntityFrameworkCore;

namespace EFCoreFirstApp.DbContexts;

public class FirstAppContext : DbContext
{
    public DbSet<LogHistory> LogHistories => Set<LogHistory>();

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // "database.db" 파일로 SQLite 사용
        optionsBuilder.UseSqlite("Data Source=database.db");
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // LogHistory의 키인 `Seq`는 자동 증가로 설정
        modelBuilder.Entity<LogHistory>()
            .Property(x => x.Seq)
            .ValueGeneratedOnAdd();
    }
}
```

## 마이그레이션

EF Core는 도구를 통해 마이그레이션 할 수 있으며 엔터티 및 DB 컨텍스트의 변화를 감지해 변화에 대한 마이그레이션 코드를 자동 생성합니다. 이 정보로 데이터베이스에 업데이트해서 최신의 모델을 데이터베이스 스키마로 적용할 수 있습니다.

| 마이그레이션 추가
```shell
dotnet ef migrations add first
```

`dotnet ef migrations add` 명령과 `first`라는 마이그레이션명으로 마이그레이션을 진행했으며 이 명령을 통해 다음의 코드가 자동 생성됩니다.

| Migrations/20220517050233_first.cs
```csharp
using System;
using Microsoft.EntityFrameworkCore.Migrations;

#nullable disable

namespace EFCoreFirstApp.Migrations
{
    public partial class first : Migration
    {
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.CreateTable(
                name: "LogHistories",
                columns: table => new
                {
                    Seq = table.Column<int>(type: "INTEGER", nullable: false)
                        .Annotation("Sqlite:Autoincrement", true),
                    Detail = table.Column<string>(type: "TEXT", nullable: false),
                    CreateTime = table.Column<DateTime>(type: "TEXT", nullable: false)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_LogHistories", x => x.Seq);
                });
        }

        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropTable(
                name: "LogHistories");
        }
    }
}
```

이 정보를 이용해 데이터베이스 스키마에 적용하려면 다음의 명령을 사용합니다.

```shell
dotnet ef database update
```


명령이 정상 실행되면 프로젝트 디렉토리에 `database.db`가 생성된 것을 확인할 수 있으며 `DB Browser`등의 SQLite 클라이언트를 이용해 데이터베이스 스키마가 잘 생성되었음을 확인할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652764049012/_dpaHCf84.png align="left")

> 생성된 `database.db`는 실행경로로 복사해야 동일한 설정 환경에서 테스트가 가능합니다.
>  ![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652767597040/IdlPXI33h.png align="left")

## 코드 작성

생성한 모델을 이용해 정보를 저장하고 조회하는 코드를 작성해봅시다.

```csharp
using EFCoreFirstApp.DbContexts;
using EFCoreFirstApp.Entities;

using var c = new FirstAppContext();

// 갯수 조회
var count = c.LogHistories.Count();

// 목록 3개 추가
var lastSeq = 0;
for (var i = 0; i < 3; i++)
{
    var newinfo = new LogHistory($"{i + 1}번째 데이터");
    c.LogHistories.Add(newinfo);

    c.SaveChanges();    
    // 저장 후 식별번호 생성됨
    lastSeq = newinfo.Seq;
}

// 목록 조회
foreach (var info in c.LogHistories)
{
    Console.WriteLine($"{info.Seq} : {info.Detail}, {info.CreateTime}");
}

// 추가한 목록 중 마지막 항목 수정
var targetInfo = c.LogHistories.First(x => x.Seq == lastSeq);
targetInfo.Detail += "(수정함)";
c.SaveChanges();

Console.WriteLine();

// 다시 목록 조회
foreach (var info in c.LogHistories)
{
    Console.WriteLine($"{info.Seq} : {info.Detail}, {info.CreateTime}");
}

Console.WriteLine();

// 마지막 항목 삭제
c.LogHistories.Remove(targetInfo);
c.SaveChanges();

// 다시 목록 조회
foreach (var info in c.LogHistories)
{
    Console.WriteLine($"{info.Seq} : {info.Detail}, {info.CreateTime}");
}
```

| 결과
```
1 : 1번째 데이터, 2022-05-17 오후 3:15:35
2 : 2번째 데이터, 2022-05-17 오후 3:15:35
3 : 3번째 데이터, 2022-05-17 오후 3:15:35

1 : 1번째 데이터, 2022-05-17 오후 3:15:35
2 : 2번째 데이터, 2022-05-17 오후 3:15:35
3 : 3번째 데이터(수정함), 2022-05-17 오후 3:15:35

1 : 1번째 데이터, 2022-05-17 오후 3:15:35
2 : 2번째 데이터, 2022-05-17 오후 3:15:35
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652768181539/co5Et38KW.png align="left")

LINQ 메소드와 항목을 추가하거나 변경하고 삭제하는 동작이 데이터베이스에 반영됨을 확인할 수 있습니다.

## 소스코드
https://github.com/dimohy/efcore-learning/tree/main/1.%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/EFCoreFirstApp
