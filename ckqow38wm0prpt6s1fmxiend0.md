## [C# 10] record struct : C# struct은 default 값과 비교 할 수 없나요?

다음의 코드로 확인을 해봅시다.

```csharp
// default 값을 할당할 수는 있지만,
StructValue a = default;

// 비교할 수는 없습니다.
// CS0019 오류 발생
if (a == default)
{
}

struct StructValue
{
    int Value1 { get; init; }
    int Value2 { get; init; }
    object ObjValue { get; init; }
}
```

a와 default를 비교연산자로 비교하려고 했으나 `CS0019`오류가 발생하는데요, 이유는 기본 struct에는 비교연산자가 없기 때문입니다. struct에서 default로 비교하기 위해서는 비교연산자를 다음 처럼 구현해야 합니다.

```csharp
struct StructValue
{
    int Value1 { get; init; }
    int Value2 { get; init; }
    object ObjValue { get; init; }

    public static bool operator ==(StructValue lhs, StructValue rhs) => lhs.Value1 == rhs.Value1 && lhs.Value2 == rhs.Value2 && lhs.ObjValue == rhs.ObjValue;
    public static bool operator !=(StructValue lhs, StructValue rhs) => !(lhs.Value1 == rhs.Value1);

    public override bool Equals(object obj)
    {
        if (obj == null)
            return false;

        if (obj is not StructValue value)
            return false;

        return this == value;
    }

    public override int GetHashCode() => Value1.GetHashCode() ^ Value2.GetHashCode() ^ (ObjValue?.GetHashCode() ?? 0);
}
```

이제 다음처럼 비교 연산할 수 있게 됩니다.

```csharp
System.Console.WriteLine(a == default);
System.Console.WriteLine(a.Equals(default(StructValue)));
System.Console.WriteLine(a.GetHashCode());
```

결과
```console
True
True
0
```

C# 10에서는 struct에서 비교연산자를 구현해야 하는 번거로움을 `record struct`을 이용해 해결할 수 있습니다. record의 struct 버젼이라고 이해하면 될 것 같습니다.

```csharp
RecordStructValue b = default;
System.Console.WriteLine(b == default);
System.Console.WriteLine(b.Equals(default(RecordStructValue)));
System.Console.WriteLine(b.GetHashCode());

record struct RecordStructValue(int Value1, int Value2, object ObjValue);
```

결과
```console
True
True
0
```
