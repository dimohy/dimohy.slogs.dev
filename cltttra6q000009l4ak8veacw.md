---
title: "[EF Core] ValueConverter를 이용해서 엔터티 속성의 도메인 관리"
datePublished: Sat Mar 16 2024 08:27:14 GMT+0000 (Coordinated Universal Time)
cuid: cltttra6q000009l4ak8veacw
slug: ef-core-valueconverter
tags: dotnet, efcore

---

EF Core를 사용하면서 문자열 길이 등의 특성을 일일이 지정하는 것은 번거롭습니다.

```csharp
...
[MaxLength(32)]
public string? 제목 { get; set; }
```

> 엔터티가 한 개일 때는 상관이 없으나 `제목` 유형이 여러 엔터티에 사용될 경우 유형을 지정하기 번거롭습니다.

속성 유형을 도메인으로 관리하면 참 편할텐데요, [ValueConverter](https://learn.microsoft.com/en-us/ef/core/modeling/value-conversions?tabs=data-annotations#the-valueconverter-class&WT.mc_id=DT-MVP-5004759)를 이용할 수 있습니다.

그런데 이것을 [인터페이스 정적 추상](https://learn.microsoft.com/ko-kr/dotnet/csharp/language-reference/proposals/csharp-11.0/static-abstracts-in-interfaces?WT.mc_id=DT-MVP-5004759)를 사용해서 다음처럼 관리를 하면 편리합니다.

다음 코드 예시는 유형의 길이(MaxLength) 등을 정의해서 활용하는 방법입니다.

```csharp
public interface IHaveValue<TModel, TProvider>
{
    public abstract static TModel Create(TProvider Value);

    TProvider Value { get; }
}

public interface IHaveValueWithLength<TModel, TProvider> : IHaveValue<TModel, TProvider>
{
    public abstract static int MaxLength { get; }
}

public record Title(string Value) : IHaveValueWithLength<Title, string>
{
    public static int MaxLength => 64;

    public static Title Create(string Value) => new(Value);
}
```

| ValueWithLengthConverter.cs

```csharp
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
```

다음처럼 적용하고 사용할 수 있습니다.

| DbContext

```csharp
...
    protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
    {
        configurationBuilder
            .Properties<Title>()
            .HaveConversion<ValueWithLengthConverter<Title, string>>();
    }
...
```

| 엔터티

```csharp
...
    public Title? 제목 { get; set; }
...
```

이제 `제목`의 경우 `Title` 유형을 사용하는 것으로 일관되게 유형 길이가 적용이 됩니다.

---

uuid를 적용할때도 활용할 수 있습니다. 저는 [nanoid-net](https://github.com/codeyu/nanoid-net)를 쓸 것인데요, 다음처럼 사용할 수 있습니다.

```csharp
public record Uid(string Value) : IHaveValueWithLength<Uid, string>
{
    public static int MaxLength => 21;

    public static Uid Create(string Value) => new(Value);
    public static Uid New() => new(Nanoid.Generate(size: MaxLength));
}

public record Suid(string Value) : IHaveValueWithLength<Suid, string>
{
    public static int MaxLength => 10;

    public static Suid Create(string Value) => new(Value);

    public static Uid New() => new(Nanoid.Generate(size: MaxLength));
}
```

| DbContext

```csharp
...
    protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
    {
        configurationBuilder
            .Properties<Uid>()
            .HaveConversion<ValueWithLengthConverter<Uid, string>>();

        configurationBuilder
            .Properties<Suid>()
            .HaveConversion<ValueWithLengthConverter<Suid, string>>();
    }
...
```

| 엔터티

```csharp
    [Key]
    public Uid Id { get; init; } = Uid.New();
```

![image](https://discourse-dotnetdev-upload.sgp1.vultrobjects.com/original/3X/c/8/c8b20f6233d492052f3376344c3e9bb7667c1419.png align="left")

잘 적용되네요!