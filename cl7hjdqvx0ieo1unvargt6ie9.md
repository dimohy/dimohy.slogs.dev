---
title: "C# 11 기능 정리"
datePublished: Wed Aug 31 2022 11:30:01 GMT+0000 (Coordinated Universal Time)
cuid: cl7hjdqvx0ieo1unvargt6ie9
slug: csharp11-future
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1661945723080/6nCdlBk46.png
tags: csharp

---

.NET 7이 미리보기 릴리스가 종료되면서 C# 11 기능이 확정되었습니다. 이에 따라 C# 11 기능을 대해 정리합니다.


## C# 11 기능 목록

새롭게 추가된 C# 11의 기능은 다음과 같습니다.
- [File-local Types](https://github.com/dotnet/csharplang/issues/6011) (파일 로컬 타입)
- [ref fields](https://github.com/dotnet/csharplang/blob/main/proposals/low-level-struct-improvements.md) (ref 필드)
- [Required members](https://github.com/dotnet/csharplang/issues/3630) (required 멤버)
- [DIM for Static Members](https://github.com/dotnet/csharplang/issues/4436) (정적 추상 멤버)
- [Numberic IntPtr](https://github.com/dotnet/csharplang/issues/4682) (숫자 IntPtr)
- [Unsigned right shift operator](https://github.com/dotnet/csharplang/issues/184) (부호 없는 오른쪽 시프트 연산자)
- [Utf8 String Literals](https://github.com/dotnet/csharplang/issues/184) (UTF8 문자열 리터럴)
- [Pattern matching on `ReadOnlySpan<char>`](https://github.com/dotnet/csharplang/issues/1881) (`ReadOnlySpan<char>`의 패턴 매칭)
- [Checked Operators](https://github.com/dotnet/csharplang/issues/4665) (checked 연산자)
- [auto-default structs](https://github.com/dotnet/csharplang/issues/5737) (자동 기본 구조체)
- [Newlines in interpolations](https://github.com/dotnet/csharplang/issues/4935) (보간의 개행)
- [List patterns](https://github.com/dotnet/csharplang/issues/3435) (목록 패턴)
- [Raw string literals](https://github.com/dotnet/csharplang/issues/4304) (원시 문자열 리터럴)
- [Cache delegates for static method group](https://github.com/dotnet/roslyn/issues/5835) (정적 메소드 그룹에 대한 캐시 대리자)
- [nameof(parameter)](https://github.com/dotnet/csharplang/issues/373)
- [Relaxing Shift Operator](https://github.com/dotnet/csharplang/issues/4666) (시프트 연산자의 제약 완화)
- [Generic attributes](https://github.com/dotnet/csharplang/issues/124) (제네릭 특성)


## C# 11 기능

### 파일 로컬 타입 ([File-local Types](https://github.com/dotnet/csharplang/issues/6011))

파일 내에서만 유효한 타입을 정의할 수 있습니다.

```csharp
// FileLocalType.cs
file class FileLocalType
{
}

// Program.cs
var s = new FileLocalType();
// 오류	CS0246	'FileLocalType' 형식 또는 네임스페이스 이름을 찾을 수 없습니다. using 지시문 또는 어셈블리 참조가 있는지 확인하세요.
```


### ref 필드 ([ref fields](https://github.com/dotnet/csharplang/blob/main/proposals/low-level-struct-improvements.md))

`Span<T>`의 `ref 필드` `_reference` 처럼

```csharp
 public readonly ref struct Span<T>
    {
        /// <summary>A byref or a native ptr.</summary>
        internal readonly ref T _reference;
        /// <summary>The number of elements this Span contains.</summary>
        private readonly int _length;
        ...
```

이제 일반 개발자도 `ref 필드`를 만들 수 있게 되었습니다.
> ref 필드는 스택 공간에만 존재할 수 있으므로 `ref struct`에서만 사용할 수 있습니다.

```csharp
ref struct Vector2D
{
	public ref int X;
	public ref int Y;
}
```


### required 멤버 ([Required members](https://github.com/dotnet/csharplang/issues/3630))

이제 `required` 키워드로 필수 멤버를 만들 수 있습니다.

```csharp
public class User
{
    public required string Name { get; init; }
    public required int Age { get; init; }
}
```

```csharp
var user1 = new User();
// 오류	CS9035	필수 구성원 'User.Age'은(는) 개체 이니셜라이저 또는 특성 생성자에서 설정해야 합니다.
// 오류	CS9035	필수 구성원 'User.Name'은(는) 개체 이니셜라이저 또는 특성 생성자에서 설정해야 합니다.

var user2 = new User
{
	Name = "dimohy",
	Age = 45
};
```


### 정적 추상 멤버 ([DIM for Static Members](https://github.com/dotnet/csharplang/issues/4436))

이제 정적 속성 및 메소드 역시 인터페이스로 구현을 강제할 수 있습니다.

```csharp
public interface IValue<T>
{
    static abstract T Empty { get; }
}

public class NumValue : IValue<NumValue>
{
    public static NumValue Empty { get; } = new(0);

    private int _value;

    public NumValue(int value) => _value = value;
}
```

이를 통해 이제 제네릭 형식 제약 조건 `where`를 이용해 정적 속성 및 메소드를 사용하는 기능을 구현할 수 있습니다.

```csharp
var emptyValue = GetEmptyValue(new NumValue(100));

T GetEmptyValue<T>(T value) where T : IValue<T>
{
    return T.Empty;
}
```


### 숫자 IntPtr ([Numberic IntPtr](https://github.com/dotnet/csharplang/issues/4682))

`IntPtr/UIntPtr`과 플랫폼 의존적인 정수 타입 [nint / nuint](https://www.sysnet.pe.kr/2/0/12366)이 
플랫폼 의존적인 동일한 정수형 데이터 값임에도 불구하고 다르게 취급되어 정수형에서 제공하는 연산을 제공하지 않았습니다.

이제 두 타입은 동일한 타입으로 처리됩니다. 즉, nint와 nuint는 IntPtr과 UIntPtr의 alias이고 동일한 타입이 됩니다.


### 부호 없는 오른쪽 시프트 연산자 ([Unsigned right shift operator](https://github.com/dotnet/csharplang/issues/184))

Java에도 존재하는 `uinsiged right shift` 연산자를 이제 C#에서도 제공합니다.

```chsarp
int n = -2020987651; // 0b10000111100010100010110011111101

Console.WriteLine($"[n] \t\t {Convert.ToString(n, 2).PadLeft(32, '0')}");
Console.WriteLine($"[n] >>> 4 ==> \t {Convert.ToString(n >>> 4, 2).PadLeft(32, '0')}");
Console.WriteLine($"[n] >> 4 ==> \t {Convert.ToString(n >> 4, 2).PadLeft(32, '0')}");
```

```
[n]              10000111100010100010110011111101
[n] >>> 4 ==>    00001000011110001010001011001111
[n] >> 4 ==>     11111000011110001010001011001111
```


### UTF8 문자열 리터럴 ([Utf8 String Literals](https://github.com/dotnet/csharplang/issues/184))

이제 `u8` 접미사를 통해 UTF8 문자열 리터럴을 만들 수 있습니다.

```csharp
ReadOnlySpan<byte> utf8Text = "디모이"u8; // 또는
var utf8Text = "디모이"u8;
```

UTF8 문자열 리터럴은 `ReadOnlySpan<byte>` 타입이므로 일반 문자열 리터럴 처럼 상수로 취급되지는 않습니다. 또한 `ReadOnlySpan<byte>` 타입이 스택에만 존재할 수 있으므로 비동기 메서드에서 사용할 수 없는 제약이 존재합니다.


### `ReadOnlySpan<char>`의 패턴 매칭 ([Pattern matching on `ReadOnlySpan<char>`](https://github.com/dotnet/csharplang/issues/1881))

이제 패턴 매칭에서 `ReadOnlySpan<char>`를 문자열과 매칭시킬 수 있습니다.

```csharp
ReadOnlySpan<char> text = "디모이";

switch (text)
{
    case "디모이":
        break;
    default:
        break;
}
```


### checked 연산자 ([Checked Operators](https://github.com/dotnet/csharplang/issues/4665))

이제 overflow 제어에 필요한 `checked/unchecked` 동작을 직접 처리할 수 있게 됩니다.

```csharp
// 출처 - 성태의 닷넷 이야기
public static Int3 operator checked ++(Int3 lhs)
{
    if (lhs.value + 1 > 8388607)
    {
        throw new OverflowException((lhs.value + 1).ToString());
    }

    return new Int3(lhs.value + 1);
}

public static Int3 operator ++(Int3 lhs) => new Int3(lhs.value + 1);
```


### 자동 기본 구조체 ([auto-default structs](https://github.com/dotnet/csharplang/issues/5737))

이제 구조체에서 필드를 초기화 하지 않아도 자동으로 `default`으로 초기화 됩니다.

```csharp
// C# 11 이전에는 초기화 관련 오류 발생
public struct S
{
    public int x, y;
    public S()
    {
    }
}
```


### 보간의 개행 ([Newlines in interpolations](https://github.com/dotnet/csharplang/issues/4935))

이제 보간에서 개행이 가능합니다.

```csharp
var v = $"Count is\t: { this.Is.A.Really()
                            .That.I.Should(
                                be + able)[
                                    to.Wrap()] }.";
```


### 목록 패턴 ([List patterns](https://github.com/dotnet/csharplang/issues/3435))

이제 목록 패턴이 지원됩니다.

```csharp
int[] arr = { 1, 2, 3, 4, 5, 6 };

var result = arr switch
{
    //[1, .., 6] => 1,
    [1, .., 5, _] => 2,
    _ => 0
};
Console.WriteLine(result);
```


### 원시 문자열 리터럴 ([Raw string literals](https://github.com/dotnet/csharplang/issues/4304))

이제 원시 문자열 리터럴을 사용할 수 있습니다.

```
var rawText = """
    Line 1, value 1
    Line 2, value 2
    Line 3, value 3
    """;

Console.WriteLine(rawText);
```

마지막 닫는 `"""` 위치에 맞게 문자열로 잘 대입됩니다.

```
Line 1, value 1
Line 2, value 2
Line 3, value 3
```

만약 문자열 안에 `"""`가 포함되어 있을 경우, 열고 닫는 큰따옴표를 `""""`로 할 수 있습니다.

```csharp
var rawText = """"
    Line 1, value 1
    Line 2, value 2
    Line 3, value 3
    """Text"""
    """";

Console.WriteLine(rawText);
```

```
Line 1, value 1
Line 2, value 2
Line 3, value 3
"""Text"""
```

또한 문자열 보간도 사용할 수 있습니다.

```csharp
var num = 1;
var rawText = $""""
    Line 1, value {num}
    Line 2, value {num + 1}
    Line 3, value {num + 2}
    """Text"""
    """";

Console.WriteLine(rawText);
```

```
Line 1, value 1
Line 2, value 2
Line 3, value 3
"""Text"""
```

그런데 JSON 문자열 처럼 문자열에 `{` 또는 `}`이 있을 경우 `$` 대신 `$$`을, ``{value}`` 대신 ``{{value}}``으로 사용할 수 있습니다.

```csharp
var num = 1;
var rawText = $$""""
    {
    Line 1, value {{num}}
    Line 2, value {{num + 1}}
    Line 3, value {{num + 2}}
    }
    """Text"""
    """";

Console.WriteLine(rawText);
```

```
{
Line 1, value 1
Line 2, value 2
Line 3, value 3
}
"""Text"""
```


### 정적 메소드 그룹에 대한 캐시 대리자 ([Cache delegates for static method group](https://github.com/dotnet/roslyn/issues/5835))

클로저가 없는 정적 메소드 그룹에 대해서 C# 11 이전에는 호출할 때마다 대리자의 새 인스턴스를 만들었습니다.

```csharp
void Demo(Func<string> action) { }
string GetString() => "";

```csharp
// C# code
Demo(GetString); // -> Demo(new Func<string>(GetString));
```

이제 클로저가 없으면 컴파일러는 대리자를 캐시하고 각 호출에서 재사용 합니다.


### [nameof(parameter)](https://github.com/dotnet/csharplang/issues/373)

이제 매개변수의 이름 역시 `nameof(parameter)` 형태로 사용할 수 있게 되었습니다.

```csharp
Assert(1 == 1);
Assert("A" == "B");


void Assert(bool condition, [CallerArgumentExpression(nameof(condition))] string? message = default)
{
    Console.WriteLine(nameof(condition));
    Console.WriteLine(message);
}
```

```
condition
1 == 1
condition
"A" == "B"
```


### 시프트 연산자의 제약 완화 ([Relaxing Shift Operator](https://github.com/dotnet/csharplang/issues/4666))

이제 시프트 연산자의 타입 완화로 다른 타입을 연산자 매개변수로 받을 수 있게 되었습니다.

```csharp
// 출처 - 성태의 닷넷 이야기 
C c = new C();
C d = c << 5.8f;

class C
{
    public static C operator << (C c, float o)
    {
        Console.WriteLine("shift operator called with " + o);

        return c;
    }
}
```


### 제네릭 특성 ([Generic attributes](https://github.com/dotnet/csharplang/issues/124))

이제 특성 타입에 제네릭을 사용할 수 있게 되었습니다.

```csharp
class ValueAttribute<TValue> : Attribute
    where TValue : struct
{
    public TValue V { get; set; }
}

[Value<int>(V = 10)]
class TestClass
{

}
```


## 참고 자료

좀 더 상세한 내용은 아래의 문서를 확인하세요.

- [성태의 닷넷 이야기](https://www.sysnet.pe.kr/)
	- [C# 11 - 메서드 매개 변수에 대한 nameof 지원](https://www.sysnet.pe.kr/2/0/13122?pageno=0)
	- [C# 11 - 파일 범위 내에서 유효한 타입 정의 (File-local types)](https://www.sysnet.pe.kr/2/0/13117?pageno=0)
	- [C# 11 - Span 타입에 대한 패턴 매칭 (Pattern matching on `ReadOnlySpan<char>`)](https://www.sysnet.pe.kr/2/0/13113?pageno=0)
	- [C# 11 - 목록 패턴(List patterns)](https://www.sysnet.pe.kr/2/0/13112?pageno=0)
	- [C# 11 - IntPtr/UIntPtr과 nint/nuint의 통합](https://www.sysnet.pe.kr/2/0/13111?pageno=0)
	- [C# 11 - 새로운 연산자 ">>>" (Unsigned Right Shift)](https://www.sysnet.pe.kr/2/0/13110?pageno=0)
	- [C# 11 - shift 연산자 재정의에 대한 제약 완화 (Relaxing Shift Operator)](https://www.sysnet.pe.kr/2/0/13100?pageno=0)
	- [C# 11 - 사용자 정의 checked 연산자](https://www.sysnet.pe.kr/2/0/13099?pageno=0)
	- [C# 11 - UTF-8 문자열 리터럴](https://www.sysnet.pe.kr/2/0/13096?pageno=0)
	- [C# 11 - 문자열 보간 개선 2가지](https://www.sysnet.pe.kr/2/0/13086?pageno=0)
	- [C# 11 - 원시 문자열 리터럴(raw string literals)](https://www.sysnet.pe.kr/2/0/13085?pageno=0)
	- [C# 11 - ref struct에 ref 필드를 허용](https://www.sysnet.pe.kr/2/0/13015?pageno=0)
	- [C# 11 - 제네릭 타입의 특성 적용](https://www.sysnet.pe.kr/2/0/12839?pageno=0)
	- [C# 11 - 인터페이스 내에 정적 추상 메서드 정의 가능 (DIM for Static Members)](https://www.sysnet.pe.kr/2/0/12814?pageno=0)
- [Performance: Lambda Expressions, Method Groups, and delegate caching | Gérald Barré | MEZIANTOU'S BLOG](https://www.meziantou.net/performance-lambda-expressions-method-groups-and-delegate-caching.htm)
