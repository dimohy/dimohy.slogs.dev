---
title: "C# Next (C# 11 후보) 언어 기능"
datePublished: Wed Dec 29 2021 09:48:06 GMT+0000 (Coordinated Universal Time)
cuid: ckxrcuzk409qk1js1erxo48vh
slug: c-next-c-11
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1640841167354/8lR7DA5UD.png
tags: csharp

---

C# 11의 언어 기능이 될 C# Next가 공개되었습니다.
오늘은 C# Next 언어 기능에 대해 대략적으로 살펴볼 것입니다.

## C# Next 언어 기능 목록

### 보간 내 개행 (Newlines in interpolations)
보간된 문자열의 중괄호 안의 식이 개행이 되면 컴파일 오류가 발생합니다. 다음의 코드는 `C# 10`에서는 오류입니다.

```csharp
var v = $"Count is\t: { this.Is.A.Really()
                            .That.I.Should(
                                be + able)[
                                    to.Wrap()] }.";
```

하지만 C# Next에서는 정상적인 코드로 간주합니다.
`Visual Studio 2022 17.1 Preview 1` 이후 프로젝트 설정 `EnablePreviewFeatures`을 `true`로 주어 사용해볼 수 있습니다.

### 리스트 패턴 (List patterns)
C#의 강력한 패턴 매칭 기능에 리스트 패턴이 포함될 예정입니다.

```csharp
_ = list switch
{
    { .., >= 0 } => 1,
    { < 0 } => 2,
    { Count: <= 0 or > 1 } => 3,
};
```

### 인자 null 확인 (Parameter null-checking)
함수 인자가 null 일 경우 우리는 일일이 이를 확인해야 했었습니다.

```csharp
void Insert(string s) {
  if (s is null)
    throw new ArgumentNullException(nameof(s));

  ...
}
```

C# Next에서는 다음처럼 인자에 `!!`을 줘서 처리할 예정입니다.

```csharp
void Insert(string s!!) {
  ...
}
```

### 순수 문자열 리터럴 (Raw string literals)
C#에서는 여러 개행으로 구성된 문자열을 효과적으로 표현할 방법이 없었습니다.

C# Next에서는 `"""`을 이용해 이를 표현할 수 있을 예정입니다.

```csharp
var xml = """
          <element attr="content"/>
          """;
```

### nameof(parameter)

`nameof()`에 인자를 넣을 수 있도록 확장합니다.

```csharp
var consoleName = nameof(System, Console);
```

### `ref` 및 `partial`의 순서 완화 (Relax ordering of `ref` and `partial` modifiers)

### 제네릭 특성 (Generic attributes)

전에는 특성(attribute)에 제네릭을 사용할 수 없었습니다.

`Visual Studio 2022 17.0 Preview 4`이후  프로젝트 설정 `EnablePreviewFeatures`을 `true`로 주어 사용해볼 수 있습니다.

```csharp
public class Person
{
  [ValidateIf<RangeAttribute>(() => new RangeAttribute(params), Condition set up...)]
  public string Name { get; set; }
}
```

```csharp
[Command("foo")]
public Task Foo([OverrideTypeReader<AlternativeBoolReader>] bool switch)
{
    //.....
}
```

### default 분해 (Default in deconstruction)
default를 값 튜플로 분해할 수 있도록 합니다.

```
(int i, string j) = default;
```

### 반자동 속성 (Semi-auto-properties)

속성으로 저장할 필드를 `field` 키워드로 바로 사용 가능할 예정입니다.

```csharp
public string PropertyConstraint {
    get;
    set => field = value ?? throw new ArgumentNullException();
} = ""
```

### required 맴버 (Required members)

속성 또는 필드가 개체 초기화 단계에서 반드시 설정되어야 함을 나타내는 `required` 키워드가 추가될 예정입니다.

```csharp
public class Person
{
    // The default constructor requires that FirstName and LastName be set at construction time
    public required string FirstName { get; init; }
    public string MiddleName { get; init; } = "";
    public required string LastName { get; init; }
}
```

### 최상위 프로그램에서 Main 속성 부여 (Top Level statement attribute specifiers)
현재로선 TLS에서 Main 메소드에 속성을 부여할 수 없습니다.

C# Next에서는 이것이 가능해 질 예정입니다.

```csharp
[main: STAThread]
```

### 기본 생성자 (Primary Constructors)
`record`는 선언을 단순화 하는 기본 생성자를 지원합니다.

```csharp
public record Person(string FirstName, string LastName);
```

이를 `class`, `struct`에 확장할 예정입니다.

### Span + Stackalloc Params의 모든 배열 유형 (Params Span + Stackalloc any array type)
`params Span<T>`를 지원해서 힙 할당 없이 `params`를 사용할 수 있을 예정입니다.

## 상세 정보
- [Language Feature Status](https://github.com/dotnet/roslyn/blob/main/docs/Language%20Feature%20Status.md) C# Next
