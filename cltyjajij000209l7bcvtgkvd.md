---
title: "[EF Core] 엔터티 속성을 JSON 열에 매핑하는 방법"
datePublished: Tue Mar 19 2024 15:33:08 GMT+0000 (Coordinated Universal Time)
cuid: cltyjajij000209l7bcvtgkvd
slug: ef-core-json-attribute-mapping
tags: dotnet, efcore

---

EF Core는 제한적인 JSON 속성을 지원합니다.

EF Core 8에 추가된 [기본 형식 컬렉션](https://learn.microsoft.com/ko-kr/ef/core/what-is-new/ef-core-8.0/whatsnew#primitive-collections)으로 JSON 열에 매핑할 수 있습니다.

```csharp
public class PrimitiveCollections
{
    public IEnumerable<int> Ints { get; set; }
    public ICollection<string> Strings { get; set; }
    public IList<DateOnly> Dates { get; set; }
    public uint[] UnsignedInts { get; set; }
    public List<bool> Booleans { get; set; }
    public List<Uri> Urls { get; set; }
}
```

`Dictionary<TKey, TValue>`는 아직 지원하지 않습니다.

다른 방식으로 [소유 엔터티 유형(Owned Entity Types)](https://learn.microsoft.com/en-us/ef/core/modeling/owned-entities)을 이용하는 방법입니다. 이 방식을 이용하면 [JSON 열에 매핑](https://learn.microsoft.com/ko-kr/ef/core/what-is-new/ef-core-7.0/whatsnew#mapping-to-json-columns?WT.mc_id=DT-MVP-5004759) 가능합니다.

| JSON 열에 매핑할 구조 (예시)

```csharp
public class ContactDetails
{
    public Address Address { get; set; } = null!;
    public string? Phone { get; set; }
}

public class Address
{
    public Address(string street, string city, string postcode, string country)
    {
        Street = street;
        City = city;
        Postcode = postcode;
        Country = country;
    }

    public string Street { get; set; }
    public string City { get; set; }
    public string Postcode { get; set; }
    public string Country { get; set; }
}
```

그리고 이것을 Author 엔터티에 적용합니다.

```csharp
public class Author
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ContactDetails Contact { get; set; }
}
```

`Contract` 속성이 소유 엔터티 유형이라는 것을 알리기 위해 다음처럼 등록을 합니다.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Author>().OwnsOne(
        author => author.Contact, ownedNavigationBuilder =>
        {
            ownedNavigationBuilder.ToJson(); // <-
            ownedNavigationBuilder.OwnsOne(contactDetails => contactDetails.Address);
        });
}
```

`ownedNavigationBuilder.ToJson()`로 인해 `Contact` 속성이 JSON 열이 됩니다. Author.Contract -&gt; Address을 `OwnsOne()`으로 지정해야 하는 이유는 해당 속성을 쿼리할 수 있게 하기 위함입니다.

컬렉션도 `OwnsMany()`을 이용해 사용 가능하지만 안타깝게도 사전 형식(`Dictionary<TKey, TValue>`)은 지원하지 않습니다.

### 번외:사전을 포함한 자유로운 사용자 유형을 JSON으로 저장하고자 한다면?

`ValueConverter`를 사용해서 문자열 JSON &lt;-&gt; 사용자 인스턴스로 변환하여 사용하는 방법으로 JSON 속성으로 쿼리 하지는 못하지만 JSON으로 저장하고자 하는 목적에 그나마 부합하는 방법이 아닐까 합니다.

다음처럼 인터페이스 및 ValueConverter를 구성한 후

```csharp
public interface IHaveStringValue<TModel> : IHaveValueWithLength<TModel, string>
{
}

public interface IHaveValue<TModel, TProvider>
{
    public abstract static TModel Create(TProvider Value);

    TProvider Value { get; }
}

public interface IHaveValueWithLength<TModel, TProvider> : IHaveValue<TModel, TProvider>
{
    public abstract static int MaxLength { get; }
}

public class ValueWithLengthConverter<TModel, TProvider> : ValueConverter<TModel, TProvider>
    where TModel : IHaveValueWithLength<TModel, TProvider>
{
    public ValueWithLengthConverter() : base(
        v => v.Value,
        v => Convert(v),
        new ConverterMappingHints(size: TModel.MaxLength)
    )
    {
    }

    private static TModel Convert(TProvider value) => TModel.Create(value);
}

public class StringValueConverter<TModel> : ValueWithLengthConverter<TModel, string>
    where TModel : IHaveValueWithLength<TModel, string>
{
}
```

`JsonValue<TModel>`을 정의 합니다.

```csharp
public record JsonValue<TType>(TType Value) : IHaveStringValue<JsonValue<TType>>
{
    public static int MaxLength => 4192;

    string IHaveValue<JsonValue<TType>, string>.Value => JsonSerializer.Serialize(Value);

    public static JsonValue<TType> Create(string Value) => new(JsonSerializer.Deserialize<TType>(Value)!);
    public static JsonValue<TType> New() => Create("{}");
}
```

DbContext에 등록을 하고,

```csharp
            builder
                .Properties<JsonValue<폼속성>>()
                .HaveConversion<StringValueConverter<JsonValue<폼속성>>>();
```

다음처럼 사용할 수 있습니다.

```csharp
public class 폼 : BaseEntity
{
    // 키
    [Key]
    public Uid Id { get; init; } = Uid.New();

    // 속성
    public Title? 제목 { get; set; }
    [Required]
    public Description? 설명 { get; set; }
    [Required]
    public Description? 목표 { get; set; }
    public JsonValue<폼속성> 속성 { get; init; } = JsonValue<폼속성>.New();
}

public class 폼속성
{
    public List<KeyTextValue> Meta { get; init; } = [];
    public Dictionary<string, object> Meta2 { get; init; } = [];
}
```

| 테스트

```csharp
var form = new 폼
{
    제목 = new("제목1"),
    설명 = new("설명1"),
    목표 = new("목표1"),
    속성 = new JsonValue<폼속성>(new()
    {
        Meta2 = new Dictionary<string, object> { { "key1", "value1" }, { "key2", 2 } }
    })
};
```

![image](https://discourse-dotnetdev-upload.sgp1.vultrobjects.com/original/3X/d/f/df553478da81e30700a475fff33a57ef17055739.png align="left")

![image](https://discourse-dotnetdev-upload.sgp1.vultrobjects.com/original/3X/b/2/b2c431c26238276cae8273b37869156e48091f45.png align="left")