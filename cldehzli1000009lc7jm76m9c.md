---
title: "상태 반환 패턴을 이용한 예시"
datePublished: Fri Jan 27 2023 12:25:57 GMT+0000 (Coordinated Universal Time)
cuid: cldehzli1000009lc7jm76m9c
slug: 7iob7yociouwmo2zmcdtjkjthltsnyqg7j207jqp7zwcioyyioylna
tags: csharp

---

아래의 글에 영감을 받아

[https://www.thereformedprogrammer.net/a-pattern-library-for-methods-that-return-a-status-including-localization/](https://www.thereformedprogrammer.net/a-pattern-library-for-methods-that-return-a-status-including-localization/)

다음의 코드를 생성해 보았습니다. 양수만 더할 수 있는 `AddPositive()`가 있을 때 올바른 연산과 잘못된 연산을 `Result<T>`을 이용해 반환할 수 있습니다.

```csharp
// 함수의 성공/실패를 반환값을 통해 식별할 수 있는 Result 구현

Console.Write("a = ? ");
var a = int.Parse(Console.ReadLine()!);
Console.Write("b = ? ");
var b = int.Parse(Console.ReadLine()!);

var result = AddPositive(a, b);
if (result.IsError is true)
{ 
    Console.WriteLine(result.ErrorMessage);
    return;
}

Console.WriteLine(result.Value);

static Result<int> AddPositive(int a, int b)
{
    if (a < 0 || b < 0)
        return Result<int>.Error("양수만 더할 수 있습니다.");

    return Result<int>.Success(a + b);
}



readonly ref struct Result<T>
{
    public required T Value { get; init; }
    public string ErrorMessage { get; init; }
    public bool IsError => ErrorMessage is not null;

    public static Result<T> Success(T value) => new()
    {
        Value = value
    };

    public static Result<T> Error(string errorMessage) => new()
    {
        Value = default!,
        ErrorMessage = errorMessage
    };
}
```

| 정상인 경우

```csharp
a = ? 5
b = ? 4
9
```

| 오류인 경우

```csharp
a = ? 5
b = ? -2
양수만 더할 수 있습니다.
```

상태를 가진 반환형은 필요에 따라 다양하게 디자인 할 수 있으며 특히 `LINQ`와의 궁합도 잘 맞으리라 생각합니다.