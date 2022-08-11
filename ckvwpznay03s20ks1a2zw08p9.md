## C# 10의 새로운 기능 정리

## 개요

.NET 6과 함께 C# 10이 릴리즈 되었습니다. C#이 가장 많이 사용되는 프로그래밍 언어는 아니지만 꾸준히 발전하고 있으며 이번 .NET 6과 함께 C# 10은 더 빠르고 효율적인 언어가 되었습니다. 본 시간을 통해 여러분과 함께 C# 10에서 새롭게 추가된 기능을 살펴 보겠습니다.

전달이 용이하지 않은 용어는 의미가 훼손되지 않도록 영어도 같이 표현 했습니다.


## C# 10의 새로운 기능


### 전역 및 암시적 using

`global using`이 추가되면서 반복해서 파일에 포함되는 `using`을 `global using`을 사용해서 공통으로 적용할 수 있도록 해서 사용 횟수를 줄일 수 있습니다.
또한 프로젝트 유형에 따라 암시적으로 포함되는 `global using`을 설정하거나 해제할 수 있는 기능이 추가되었습니다.


#### global using 지시문

이제 `global using`을 사용하면 전체 프로젝트에 `using`이 적용됩니다.

```csharp
global using System;
```

`global using`은 `using`의 모든 기능을 동일하게 사용할 수 있습니다.

```csharp
global using static System.Console;
global using Env = System.Environment;
```

`global using`은 전체 프로젝트에 `using`이 적용되기 때문에 특정 파일, 예를 들어 `globalusings.cs`에 일괄 적용하여 사용 하는 것이 좋습니다.


#### 암시적 using

암시적 using의 기능은 프로젝트 유형에 따라 포함되는 `global using` 항목을 포함하거나 포함하지 않도록 하는 설정할 수 있습니다.

| *.csproj
```xaml
<PropertyGroup>
    <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

암시적 using은 .NET 6 템플릿에서 기본 활성화 됩니다. 활성화 되면 템플릿 프로젝트 유형에 따라 암시적으로 namespace가 포함됩니다. 프로젝트 유형에 따른 포함 항목은 [새로운 C# 전용 프로젝트의 암시적 `global using` 지시문](https://docs.microsoft.com/ko-kr/dotnet/core/compatibility/sdk/6.0/implicit-namespaces-rc1)의 SDK에 따른 기본 네임스페이스 목록을 살펴보세요.


#### 기능을 이용해서 통합

프로젝트 유형에 따라 적용되는 암시적 using은 대부분 잘 동작할 것입니다. 하지만 `global using`은 이름 충돌을 일으킬 수 있으므로 프로젝트 파일을 통해 이를 제어할 수 있습니다.

| *.csproj
```xaml
<ItemGroup>
  <Using Remove="System.Threading.Tasks" />
</ItemGroup>
```

반대로 `global using` 지시문 처럼 프로젝트 파일에 네임스페이스를 포함할 수도 있습니다.

| *.csproj
```xaml
<ItemGroup>
  <Using Include="System.IO.Pipes" />
</ItemGroup>
```


### 파일 범위 namespace

C# 10에 파일 범위 namespace가 추가되었습니다. C#은 namespace를 중괄호로 표현했는데 이는 다중 namespace를 표현하기 위함 이였습니다. 하지만 GitHub의 C# 코드 사례를 보면 99% 이상 다중 namespace를 사용하지 않는다고 합니다. 이에 따라 중괄호를 제거할 수 있는 파일 범위 namespace가 추가되었습니다.

```csharp
namespace MyCompany.MyNamespace;

class MyClass
{ ... } 
```

namespace의 중괄호가 없어져서 코드 중첩이 줄어들었습니다. 파일 범위 namespace는 유형이 선언되기 전에 위치해야 합니다.


### 람다 표현식과 메소드의 유추 개선

람다 표현식과 메소드 대입의 유추 기능이 개선되었습니다. 이에 따라 [ASP.NET Minimal API](https://devblogs.microsoft.com/dotnet/announcing-asp-net-core-in-net-6/)에서 좀 더 간결한 표현이 가능해졌습니다.


#### 람다식 유형의 유추

이전까지는 다음과 같이 람다식을 사용해야 했었습니다.

```csharp
Func<string, int> parse = (string s) => int.Parse(s);
```

이제 유추 기능이 개선되어 다음과 같이 쓸 수 있게 되었습니다.

```chsarp
var parse = (string s) => int.Parse(s);
```

람다식은 컴파일러에 의해 적절한 Func<...>나 Action<...>로 적용되게 됩니다.

하지만 매개 변수형이 없을 경우 반환 유형을 유추할 수 없어서 오류가 발생합니다.

```csharp
var parse = s => int.Parse(s); // 컴파일 오류: s 유형을 유추할 수 없음
```

또한 상위 타입으로 람다식을 할당할 수 있습니다.

```csharp
object parse = (string s) => int.Parse(s);   // Func<string, int>
Delegate parse = (string s) => int.Parse(s); // Func<string, int>
```

람다식을 표현식으로 받기 위해서 Expression을 명시적으로 표시할 수 있습니다.

```csharp
LambdaExpression parseExpr = (string s) => int.Parse(s); // Expression<Func<string, int>>
Expression parseExpr = (string s) => int.Parse(s);       // Expression<Func<string, int>>
```


#### 메서드 유추

메서드를 대리자로 대입하려 할 때

```csharp
Func<int> read = Console.Read;
Action<string> write = Console.Write;
```

이제 다음처럼 유추하여 대입할 수 있습니다.

```csharp
var read = Console.Read;
var write = Console.Write; // 컴파일 오류: Write 메소드 매개변수 형을 유추할 수 없음
```

하지만 `Console.Write()`의 경우 매개변수 형을 유추할 수 없으므로 컴파일 오류가 발생합니다.


#### 람다 반환 유형

람다의 반환형을 유추할 수 없으면 컴파일 오류가 발생합니다.

```csharp
var choose = (bool b) => b ? 1 : "two"; // 컴파일 오류: 반환 형을 유추할 수 없음
```

C# 10에서는 반환 유형을 지정 할 수 있습니다.

```csharp
var choose = object (bool b) => b ? 1 : "two"; // Func<bool, object>
```


#### 람다 특성(attribute)

람다식에 이제 특성을 넣을 수 있습니다.

```csharp
Func<string, int> parse = [Example(1)] (s) => int.Parse(s);
var choose = [Example(2)][Example(3)] object (bool b) => b ? 1 : "two";
```


### struct 개선

C# 10에서는 struct에 매개변수 없는 생성자와 필드 이니셜라이저, record struct 및 with 표현식이 추가되었습니다.


#### 매개변수 없는 struct 생성자 및 필드 이니셜라이저

이제 매개 변수 없는 struct 생성자를 포함할 수 있습니다. 포함하지 않으면 기존 처럼 암시적으로 생성자가 적용되며 모든 필드를 기본 값으로 설정합니다. 매개 변수 없는 struct 생성자는 반드시 public 이어야 합니다.

```csharp
public struct Address
{
    public Address()
    {
        City = "<unknown>";
    }
    public string City { get; init; }
}
```

아래와 같이 속성 이니셜라이저를 통해 초기화 할 수도 있습니다.

```csharp
public struct Address
{
    public string City { get; init; } = "<unknown>";
}
```


#### record struct

이제 `record struct`를 사용할 수 있습니다. `record`와 동일한 기능을 사용할 수 있으며 기존의 `record`는 `record class`의 축약 표현이 됩니다.

```csharp
public record struct Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
}
```

또한 다음처럼 기본 생성자를 통해 좀 더 작은 코드로 표현할 수 있습니다.

```csharp
public record struct Person(string FirstName, string LastName);
```

`record class` 기본 생성자의 속성은 읽기 전용인 반면에 `record struct` 기본 생성자의 속성은 읽기/쓰기라는 점이 차이점이 있습니다. 


#### `record class`의 sealed ToString()
`record class`의 `ToString()` 메소드가 파생 record에 의해 동작이 변경되지 않도록 하는 `sealed ToString()`이 추가되었습니다.

```csharp
record Info(string Name)
{
    public override sealed string ToString()
    {
        return Name;
    }
}
record UserInfo(string Name, int Age) : Info(Name);
```


#### struct 및 익명 유형에 대한 with 표현식

이제 `with`는 `record struct` 뿐만 아니라 struct 및 익명 형식에 대해 지원합니다.

```csharp
var person2 = person with { LastName = "Kristensen" };
```


### 보간된 문자열 개선 사항

C# 10의 보간된 문자열은 성능 및 표현력에서 개선되었습니다.


#### 보간 문자열 핸들러

보간 문자열을 처리하기 위한 기존 방식은 많은 메모리 할당을 야기했습니다. C# 10에서는 핸들러를 통해 가장 효과적인 처리 방식을 택할 수 있게 되었습니다.

```csharp
var sb = new StringBuilder();
sb.Append($"Hello {args[0]}, how are you?");
```

C# 10이전의 `Append()` 메소드 매개변수로 전달되는 보간 문자열은 보간 처리 후 전달될 수 밖에 없었습니다. 하지만 C# 10에 추가된 보간 문자열 핸들러에 의해 `Append()` 메소드에서 효과적으로 처리가 가능해졌습니다.

보간 문자열은 InterpolatedStringHandler 특성이 부여된 구조체로 컴파일 시점에서 전환될 수 있습니다. 이에 따라 StringBuilder의 `Append(ref StringBuilder.AppendInterpolatedStringHandler handler)`에 의해서 핸들러로 전달되고 StringBuilder에서 효과적으로 보간 문자열을 처리하게 됩니다.


#### 상수 보간 문자열

이제 보간 문자열의 연결 문자열이 상수면 보간 문자열로 상수가 됩니다. 이에 따라 특성에 보간 문자열을 적용할 수 있게 됩니다.

```csharp
[Obsolete($"Call {nameof(Discard)} instead")]
```

숫자 상수나 날짜 값과 같은 다른 유형은 Culture에 민감하거나 컴파일 시점에 계산할 수 없는 이유로 상수 보간 문자열에 사용할 수 없습니다.


### 해체(deconstruction)시 선언과 변수의 혼합

이전에는 해체시 모든 변수가 이전에 선언되거나 해체시 선언되어야 했었는데 이제 그 제한이 없어졌습니다. 다음의 코드는 C# 10에서 모두 정상 동작합니다.

```csharp
int x2;
int y2;
(x2, y2) = (0, 1);       // C# 9 이상에서 동작
(var x, var y) = (0, 1); // C# 9 이상에서 동작
(x2, var y3) = (0, 1);   // C# 10에서 동작
```


### 향상된 한정 할당

이전에는 한정된 할당에서 잘못된 오류가 발생했습니다.

```chsarp
if ((c != null) && c.GetDependentValue(out object obj) == true)
{
    representation = obj.ToString(); // undesired error
}

// Or, using ?.
if (c?.GetDependentValue(out object obj) == true)
{
    representation = obj.ToString(); // undesired error
}

// Or, using ??
if (c?.GetDependentValue(out object obj) ?? false)
{
    representation = obj.ToString(); // undesired error
}
```

이제 한정 할당에서 잘못된 오류가 개선되었습니다.


### 확장된 속성 패턴

이제 중첩 속성 값에 쉽게 접근할 수 있도록 속성 패턴이 개선되었습니다.

```csharp
object obj = new Person
{
    FirstName = "Kathleen",
    LastName = "Dollard",
    Address = new Address { City = "Seattle" }
};

if (obj is Person { Address: { City: "Seattle" } })
    Console.WriteLine("Seattle");

if (obj is Person { Address.City: "Seattle" }) // 확장된 속성 패턴
    Console.WriteLine("Seattle");
```


### 호출자 표현식 속성

`CallerArgumentExpressionAttribute` 특성을 이용해서 매개변수로 전달되는 코드를 컴파일 시점에서 별도의 매개변수로 전달할 수 있게 되었습니다.

```csharp
void CheckExpression(bool condition, 
    [CallerArgumentExpression("condition")] string? message = null )
{
    Console.WriteLine($"Condition: {message}");
}
```

위와 같이 `CallerArgumentExpression` 특성을 이용해 `message` 매개변수를 선언하면 다음처럼

```chsarp
var a = 6;
var b = true;
CheckExpression(true);
CheckExpression(b);
CheckExpression(a > 5);

// Output:
// Condition: true
// Condition: b
// Condition: a > 5
```

매개변수로 전달되는 코드 그대로 `message`에 전달되게 됩니다.


## 정리

오늘은 C# 10의 새로운 기능에 대해 살펴봤습니다. 본 시간에서 더 자세히 다루지 못한 .NET 6 기능 및 C# 10 기능 및 쓰임은 `참고자료`의 링크를 통해 살펴보시기 바랍니다.


## 참고자료

- [Welcome to C# 10](https://devblogs.microsoft.com/dotnet/welcome-to-csharp-10/) | Microsoft .NET Blog | Kathleen
- [C# 10의 새로운 기능](https://docs.microsoft.com/ko-kr/dotnet/csharp/whats-new/csharp-10#improved-definite-assignment) | Microsoft Docs
- [Dissecting Interpolated Strings Improvements in C# 10](https://sergeyteplyakov.github.io/Blog/c%2310/2021/11/08/Dissecing-Interpolated-Strings-Improvements-In-CSharp-10.html) | Dissecting the Code
- [How C# 10.0 and .NET 6.0 improve ArgumentExceptions](https://endjin.com/blog/2021/11/csharp-10-net-6-argument-exceptions.html) | endjin | lan Griffiths

