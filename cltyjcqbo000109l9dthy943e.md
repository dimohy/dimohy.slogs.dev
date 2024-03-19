---
title: "[EF Core] 목록형 반환 시 페이지, 정렬, 검색 적용"
datePublished: Tue Mar 19 2024 15:34:50 GMT+0000 (Coordinated Universal Time)
cuid: cltyjcqbo000109l9dthy943e
slug: ef-core-page-order-search
tags: dotnet, efcore

---

목록을 조회할 때 기본적으로 페이지, 정렬, 검색에 대한 처리를 해야 합니다. 이를 쉽게 하는 방법에 대해 소개 합니다.

페이지, 정렬, 검색을 쿼리 인자로 받게 하기 위해 `SearchOptions`을 만든 후,

| SearchOptions.cs

```csharp
public class SearchOptions
{
public class SearchOptions
{
    [FromQuery(Name = "pn")]
    [DefaultValue(0)]
    public int PageNumber { get; set; } = 0;
    [FromQuery(Name = "ps")]
    [DefaultValue(30)]
    public int PageSize { get; set; } = 30;
    [DefaultValue(true)]
    [FromQuery(Name = "oo")]
    public bool IsOrderByDesc { get; set; } = true;
    [FromQuery(Name = "op")]
    public string? OrderByPropertyName { get; set; }
    [FromQuery(Name = "sp")]
    public string? SearchByPropertyName { get; set; }
    [FromQuery(Name = "sk")]
    public string? SearchKeywords { get; set; }
}
```

이를 `GetTestList()` 인자로 사용합니다.

```csharp
[HttpGet(Name = "/list")]
public IEnumerable<TestInfo> GetTestList([FromQuery] SearchOptions so)
{
...
```

![image](https://discourse-dotnetdev-upload.sgp1.vultrobjects.com/original/3X/9/7/97ad89afe3f73cb3efe633ec78b3341015296d7c.png align="left")

이제 `SearchOptions`를 처리하는 확장 메서드를 다음처럼 만듭니다.

```csharp
public static class QueryableExtensions
{
    public static IQueryable<TModel> ApplySearchOptions<TModel>(this IQueryable<TModel> @this, SearchOptions options)
        where TModel : BaseEntity
    {
        // 페이지 처리
        var result = @this
            .Skip(options.PageStart * options.PageSize)
            .Take(options.PageSize);

        // 정렬 기준
        if (options.OrderByPropertyName is null || string.IsNullOrWhiteSpace(options.OrderByPropertyName) is true)
        {
            // 생성 날짜 기준으로 정렬
            if (options.IsOrderByDesc is true)
                result = result.OrderByDescending(x => x.CreateAt);
            else
                result = result.OrderBy(x => x.CreateAt);
        }
        else
        {
            // 'OrderByPropertyName' 기준으로 정렬
            if (options.IsOrderByDesc is true)
                result = result.OrderByDescending(ToLambda<TModel>(options.OrderByPropertyName));
            else
                result = result.OrderBy(ToLambda<TModel>(options.OrderByPropertyName));
        }

        // 검색 처리
        if (string.IsNullOrWhiteSpace(options.SearchByPropertyName) is false && string.IsNullOrWhiteSpace(options.SearchKeywords) is false)
        {
            result = result.Where(ToLambda<TModel>(options.SearchByPropertyName, options.SearchKeywords));
        }

        return result;
    }

    private static Expression<Func<T, object>> ToLambda<T>(string propertyName)
    {
        var parameter = Expression.Parameter(typeof(T));
        var property = Expression.Property(parameter, propertyName);
        var convert = Expression.Convert(property, typeof(object));
        return Expression.Lambda<Func<T, object>>(convert, parameter);
    }

    private static Expression<Func<T, bool>> ToLambda<T>(string propertyName, string keywords)
    {
        var parameter = Expression.Parameter(typeof(T));
        var property = Expression.Property(parameter, propertyName);
        var convert = Expression.Convert(property, typeof(string));
        var contains = Expression.Call(convert, typeof(string).GetMethod("Contains", [typeof(string)])!, Expression.Constant(keywords));
        return Expression.Lambda<Func<T, bool>>(contains, parameter);
    }
}
```

테스트를 위한 임시 구조를 만들고,

```csharp
public class TestInfo(string name, string description) : BaseEntity
{
    public string Name { get; set; } = name;
    public string Description { get; set; } = description;
}

// BaseEntity를 상속받은 이유는 `CreateAt` 기준으로 정렬이 기본 동작이기 때문입니다.
public class BaseEntity
{
    [Required]
    public Uid? CreateId { get; set; }
    [Required]
    public DateTime CreateAt { get; set; }
    public Uid? UpdateId { get; set; }
    public DateTime? UpdateAt { get; set; }
    public Uid? DeleteId { get; set; }
    public DateTime? DeleteAt { get; set; }

    public bool IsDeleted { get; set; }
}
```

이제 이것을 쿼리에 적용 합니다.

```csharp
        // 테스트를 위한 가짜 데이터
        List<TestInfo> list = [
            new("Test1", "Test1 Description"),
            new("Test2", "Test2 Description"),
            new("Test3", "Test3 Description"),
            new("Test4", "Test4 Description"),
            new("Test5", "Test5 Description"),
            new("Test6", "Test6 Description"),
            new("Test7", "Test7 Description"),
            new("Test8", "Test8 Description"),
            new("Test9", "Test9 Description"),
            new("Test10", "Test10 Description"),
            new("Test11", "Test11 Description"),
            new("Test12", "Test12 Description"),
            new("Test13", "Test13 Description"),
            new("Test14", "Test14 Description"),
            new("Test15", "Test15 Description"),
            new("Test16", "Test16 Description"),
            new("Test17", "Test17 Description"),
            new("Test18", "Test18 Description"),
            new("Test19", "Test19 Description"),
            new("Test20", "Test20 Description"),
            new("Test21", "Test21 Description"),
            new("Test22", "Test22 Description"),
            new("Test23", "Test23 Description"),
            new("Test24", "Test24 Description"),
            new("Test25", "Test25 Description"),
            new("Test26", "Test26 Description"),
            new("Test27", "Test27 Description"),
            new("Test28", "Test28 Description"),
            new("Test29", "Test29 Description"),
            new("Test30", "Test30 Description")
        ];

        var query = list.AsQueryable();
        return query
            .ApplySearchOptions(so)
          //~~~~~~~~~~~~~~~~~~~~~~~
            .ToList();
```

| 결과

![image](https://discourse-dotnetdev-upload.sgp1.vultrobjects.com/original/3X/9/5/953cd1f6a6e839a0889a394cfc5f88598a34b8ad.png align="left")

```json
[
  {
    "name": "Test13",
    "description": "Test13 Description",
    "createId": null,
    "createAt": "0001-01-01T00:00:00",
    "updateId": null,
    "updateAt": null,
    "deleteId": null,
    "deleteAt": null,
    "isDeleted": false
  },
  {
    "name": "Test12",
    "description": "Test12 Description",
    "createId": null,
    "createAt": "0001-01-01T00:00:00",
    "updateId": null,
    "updateAt": null,
    "deleteId": null,
    "deleteAt": null,
    "isDeleted": false
  },
  {
    "name": "Test11",
    "description": "Test11 Description",
    "createId": null,
    "createAt": "0001-01-01T00:00:00",
    "updateId": null,
    "updateAt": null,
    "deleteId": null,
    "deleteAt": null,
    "isDeleted": false
  },
  {
    "name": "Test10",
    "description": "Test10 Description",
    "createId": null,
    "createAt": "0001-01-01T00:00:00",
    "updateId": null,
    "updateAt": null,
    "deleteId": null,
    "deleteAt": null,
    "isDeleted": false
  },
  {
    "name": "Test1",
    "description": "Test1 Description",
    "createId": null,
    "createAt": "0001-01-01T00:00:00",
    "updateId": null,
    "updateAt": null,
    "deleteId": null,
    "deleteAt": null,
    "isDeleted": false
  }
]
```

코드 중에서 흥미로운 코드는,

```csharp
    private static Expression<Func<T, object>> ToLambda<T>(string propertyName)
    {
        var parameter = Expression.Parameter(typeof(T));
        var property = Expression.Property(parameter, propertyName);
        var convert = Expression.Convert(property, typeof(object));
        return Expression.Lambda<Func<T, object>>(convert, parameter);
    }

    private static Expression<Func<T, bool>> ToLambda<T>(string propertyName, string keywords)
    {
        var parameter = Expression.Parameter(typeof(T));
        var property = Expression.Property(parameter, propertyName);
        var convert = Expression.Convert(property, typeof(string));
        var contains = Expression.Call(convert, typeof(string).GetMethod("Contains", [typeof(string)])!, Expression.Constant(keywords));
        return Expression.Lambda<Func<T, bool>>(contains, parameter);
    }
```

인데 이 코드를 통해서 `문자열`로 속성을 지정하고 검색할 수 있게 됩니다.