## 소스 생성기 만들기 5부 - 유형 선언의 네임스페이스 및 유형 계층 찾기

> **이 글은 Andrew Lock님의 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/) 연재를 번역한 글입니다.**

이것은 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/)의 다섯번째 게시물 입니다.

소스 생성기에 대한 다음 포스트에서는 소스 생성기를 만들 때 필요한 몇 가지 일반적인 패턴을 보여줍니다:

- 주어진 클래스/구조체/열거형 구문에 대한 유형의 네임스페이스를 결정하는 방법 
- 클래스/구조체/열거형의 이름을 계산할 때 중첩 유형을 처리하는 방법 

표면적으로는 간단한 작업처럼 보이지만 예상보다 까다로울 수 있는 미묘한 부분이 있습니다. 


## 클래스 구문의 네임스페이스 찾기 

소스 생성기에 대한 일반적인 요구 사항은 지정된 `class` 또는 기타 구문의 `namespace`를 결정하는 것입니다. 예를 들어 지금까지 이 시리즈에서 내가 설명한 `EnumExtensions` 생성기는 고정된 `namespace`인 `NetEscapades.EnumGenerators`에서 확장 메서드를 생성합니다. 한 가지 개선 사항은 원래 `enum`과 동일한 `namespace`에서 확장 메서드를 생성하는 것입니다. 

예를 들어 다음 `enum`이 있는 경우: 

```csharp
namespace MyApp.Domain
{
    [EnumExtensions]
    public enum Colour
    {
        Red = 0,
        Blue = 1,
    }
}
```

`MyApp.Domain` 네임스페이스에서 확장 메서드를 생성할 수 있습니다. 

```csharp
namespace MyApp.Domain
{
    public partial static class EnumExtensions
    {
        public string ToStringFast(this Colour colour)
            => colour switch
            {
                Colour.Red => nameof(Colour.Red),
                Colour.Blue => nameof(Colour.Blue),
                _ => colour.ToString(),
            }
        }
    }
}
```

표면적으로는 쉬울 것 같지만 불행히도 처리해야 하는 경우가 꽤 있습니다:
- 파일 범위 네임스페이스 - C# 10에 도입되었으며 중괄호를 생략하고 전체 파일에 네임스페이스를 적용합니다. 예: 

```csharp
public namespace MyApp.Domain; // 파일 범위 네임스페이스

[EnumExtensions]
public enum Colour
{
    Red = 0,
    Blue = 1,
}
```

- 여러 개의 중첩된 네임스페이스 - 다소 특이하지만 여러 개의 중첩된 네임스페이스 선언을 가질 수 있습니다:

```csharp
namespace MyApp
{
    namespace Domain // 중척된 네임스페이스
    {
        [EnumExtensions]
        public enum Colour
        {
            Red = 0,
            Blue = 1,
        }
    }
}
```

- 기본 네임스페이스 - 네임스페이스를 전혀 지정하지 않으면 기본 네임스페이스가 사용됩니다. 이 네임스페이스는 `global::`일 수 있지만 `<RootNamespace>`를 사용하여 csproj 파일에서 재정의될 수도 있습니다. 

```csharp
[EnumExtensions]
public enum Colour // 지정된 네임스페이스가 없으므로 기본값을 사용
{
    Red = 0,
    Blue = 1,
}
```

다음 주석 스니펫은 이러한 모든 경우를 처리하기 위해 [`LoggerMessage` 생성기가 사용하는 코드](https://github.com/dotnet/runtime/blob/25c675ff78e0446fe596cea25c7e3969b0936a33/src/libraries/Microsoft.Extensions.Logging.Abstractions/gen/LoggerMessageGenerator.Parser.cs#L438)를 기반으로 합니다. `BaseTypeDeclarationSyntax`(`EnumDeclarationSyntax`, `ClassDeclarationSyntax`, `StructDeclarationSyntax`, `RecordDeclarationSyntax` 등 포함)에서 파생된 일종의 "유형" 구문이 있는 경우 사용할 수 있으므로 대부분의 경우를 처리해야 합니다. 

```csharp
// 있는 경우 class/enum/struct가 선언된 네임스페이스를 결정
static string GetNamespace(BaseTypeDeclarationSyntax syntax)
{
    // 네임스페이스가 전혀 없으면 빈 문자열을 반환
    // 이것은 "기본 네임스페이스"의 경우를 설명함
    string nameSpace = string.Empty;

    // 유형 선언을 위한 포함하는 구문 노드를 가져옴
    // (예를 들어 중첩 유형일 수 있음)
    SyntaxNode? potentialNamespaceParent = syntax.Parent;

    // 네임스페이스에 도달할 때까지 중첩된 클래스 등에서 "밖으로" 계속 이동
    // 또는 부모가 없을 때까지
    while (potentialNamespaceParent != null &&
            potentialNamespaceParent is not NamespaceDeclarationSyntax
            && potentialNamespaceParent is not FileScopedNamespaceDeclarationSyntax)
    {
        potentialNamespaceParent = potentialNamespaceParent.Parent;
    }

    // 더 이상 네임스페이스 선언이 없을 때까지 반복하여 최종 네임스페이스를 빌드
    if (potentialNamespaceParent is BaseNamespaceDeclarationSyntax namespaceParent)
    {
        // 네임스페이스가 있으므로 유형으로 사용
        nameSpace = namespaceParent.Name.ToString();

        // 중첩된 네임스페이스가 없을 때까지 네임스페이스 선언을 "밖으로" 계속 이동
        while (true)
        {
            if (namespaceParent.Parent is not NamespaceDeclarationSyntax parent)
            {
                break;
            }

            // 외부 네임스페이스를 최종 네임스페이스에 접두사로 추가
            nameSpace = $"{namespaceParent.Name}.{nameSpace}";
            namespaceParent = parent;
        }
    }

    // 최종 네임스페이스 반환
    return nameSpace;
}
```

이 코드를 사용하면 위에서 정의한 모든 네임스페이스 사례를 처리할 수 있습니다. 기본/전역 네임스페이스의 경우 소스 생성기에 네임스페이스 선언을 내보내지 않음을 나타내는 `string.Empty`를 반환합니다. 이렇게 하면 생성된 코드가 `global::`이든 `<RootNamespace>`에 정의된 다른 값이든 대상 유형과 동일한 `namespace`에 있게 됩니다. 

이 코드를 사용하여 이제 원래 `enum`과 동일한 네임스페이스에서 확장 메서드를 생성할 수 있습니다. 지정된 `enum`에 대한 확장 메서드가 기본적으로 `enum`과 동일한 네임스페이스에 있는 경우 더 쉽게 검색할 수 있으므로 소스 생성기 소비자에게 더 나은 사용자 경험을 제공할 수 있습니다. 


## 유형 선언 구문의 전체 유형 계층 찾기 

지금까지 이 시리즈에서는 중첩 형식을 설명하는 `INamedTypeSymbol`에서 `ToString()`을 호출했기 때문에 확장 메서드에 대해 중첩 `enum`을 암시적으로 지원했습니다. 예를 들어 다음과 같이 정의된 `enum`이 있는 경우: 

```csharp
public record Outer
{
    public class Nested
    {
        [EnumExtensions]
        public enum Colour
        {
            Red = 0,
            Blue = 1,
        }
    }
}
```

그런 다음 Colour 구문에서 `ToString()`을 호출하면 `Outer.Nested.Colour`가 반환되며 확장 메서드에서 행복하게 사용할 수 있습니다:

```csharp
public static partial class EnumExtensions
{
    public static string ToStringFast(this Outer.Nested.Colour value)
        => value switch
        {
            Outer.Nested.Colour.Red => nameof(Outer.Nested.Colour.Red),
            Outer.Nested.Colour.Blue => nameof(Outer.Nested.Colour.Blue),
            _ => value.ToString(),
        };
}
```

불행히도, 예를 들어 `Outer<T>`와 같은 제네릭 외부 유형이 있는 경우 실패합니다. 위 스니펫에서 `Outer`를 `Outer<T>`로 바꾸면 컴파일되지 않는 `EnumExtensions` 클래스가 생성됩니다. 

```csharp
public static partial class EnumExtensions
{
    public static string ToStringFast(this Outer<T>.Nested.Colour value) // 👈 유효하지 않은 C#
    // ...
}
```

이를 처리하는 몇 가지 방법이 있지만 대부분의 경우 유형의 전체 계층 구조를 이해하는 것이 필요합니다. 확장 클래스의 계층 구조를 단순히 "복제"할 수는 없지만(확장 메서드는 중첩 유형에서 정의할 수 있음), 다른 방식으로 유형을 확장하는 경우 문제를 해결할 수 있습니다. 예를 들어, [StronglyTypedId](https://github.com/andrewlock/StronglyTypedId)를 호출하는 구조체 유형에 멤버를 추가하는 소스 생성기 프로젝트가 있습니다. 다음과 같이 중첩 구조체를 장식하는 경우: 

```csharp
public partial record Outer
{
    public partial class Generic<T> where T: new()
    {
        public partial struct Nested
        {
            [StronglyTypedId]
            public partial readonly struct TestId
            {
            }
        }
    }
}
```

그런 다음 계층을 복제하는 다음과 유사한 코드를 생성해야 합니다. 

```csharp
public partial record Outer
{
    public partial class Generic<T> where T: new()
    {
        public partial struct Nested
        {
            public partial readonly struct TestId
            {
                public TestId (int value) => Value = value;
                public int Value { get; }
                // ... etc
            }
        }
    }
}
```

이렇게 하면 제네릭 유형이나 이와 유사한 것에 대한 특별한 처리를 추가할 필요가 없으며 일반적으로 매우 다양합니다. [`LoggerMessage` 생성기가 .NET 6에서 고성능 로깅을 구현하는 데 사용](https://andrewlock.net/exploring-dotnet-6-part-8-improving-logging-performance-with-source-generators/)하는 것과 동일한 접근 방식입니다. 

소스 생성기에서 이것을 구현하려면 중첩 대상(`Colour`)의 각 "상위" 유형에 대한 세부 정보를 보유하기 위해 도우미(`ParentClass`라고 함)가 필요합니다. 3가지 정보를 기록해야 합니다. 

- 유형의 키워드, 예: `class`/`stuct`/`record`
- 유형의 이름(예: `Outer`, `Nested`, `Generic<T>`)
- 제네릭 유형에 대한 모든 제약 조건, 예: `T: new()`

또한 클래스 간의 부모/자식 참조를 기록해야 합니다. 이를 위해 `stack`/`queue`을 사용할 수 있지만 아래 구현에서는 대신 연결 목록 접근 방식을 사용합니다. 여기서 각 `ParentClass`에는 자식에 대한 참조가 포함됩니다. 

```csharp
internal class ParentClass
{
    public ParentClass(string keyword, string name, string constraints, ParentClass? child)
    {
        Keyword = keyword;
        Name = name;
        Constraints = constraints;
        Child = child;
    }

    public ParentClass? Child { get; }
    public string Keyword { get; }
    public string Name { get; }
    public string Constraints { get; }
}
```

`enum` 선언 자체에서 시작하여 다음과 유사한 코드를 사용하여 `ParentClasses`의 연결 목록을 작성할 수 있습니다. 이전과 마찬가지로 이 코드는 모든 유형(`class`/`struct` 등)에서 작동합니다:

```csharp
static ParentClass? GetParentClasses(BaseTypeDeclarationSyntax typeSyntax)
{
    // 부모 구문을 시도하고 가져옵니다. 클래스/구조체와 같은 유형이 아닌 경우 null이 됨
    TypeDeclarationSyntax? parentSyntax = typeSyntax.Parent as TypeDeclarationSyntax;
    ParentClass? parentClassInfo = null;

    // 지원되는 중첩 유형에 있는 동안 계속 반복
    while (parentSyntax != null && IsAllowedKind(parentSyntax.Kind()))
    {
        // 부모 유형 키워드(클래스/구조체 등), 이름 및 제약 조건을 기록
        parentClassInfo = new ParentClass(
            keyword: parentSyntax.Keyword.ValueText,
            name: parentSyntax.Identifier.ToString() + parentSyntax.TypeParameterList,
            constraints: parentSyntax.ConstraintClauses.ToString(),
            child: parentClassInfo); // set the child link (null initially)

        // 다음 외부 유형으로 이동
        parentSyntax = (parentSyntax.Parent as TypeDeclarationSyntax);
    }

    // 가장 바깥쪽 부모 유형에 대한 링크를 반환
    return parentClassInfo;

}

// 클래스/구조체/레코드에만 중첩될 수 있음
static bool IsAllowedKind(SyntaxKind kind) =>
    kind == SyntaxKind.ClassDeclaration ||
    kind == SyntaxKind.StructDeclaration ||
    kind == SyntaxKind.RecordDeclaration;
```

이 코드는 대상 유형에 가장 가까운 유형부터 시작하여 목록을 작성합니다. 따라서 이전 예의 경우 다음과 동일한 `ParentClass` 계층을 생성합니다. 

```csharp
var parent = new ParentClass(
    keyword: "record",
    name: "Outer",
    constraints: "",
    child: new ParentClass(
        keyword: "class",
        name: "Generic<T>",
        constraints: "where T: new()",
        child: new ParentClass(
            keyword: "struct",
            name: "Nested",
            constraints: "",
            child: null
        )
    )
);
```

그런 다음 출력을 생성할 때 소스 생성기에서 이 계층 구조를 재구성할 수 있습니다. 다음은 이전 섹션에서 추출한 네임스페이스와 `ParentClass` 계층을 모두 사용하는 간단한 방법을 보여줍니다:

```csharp
static public GetResource(string nameSpace, ParentClass? parentClass)
{
    var sb = new StringBuilder();

    // 네임스페이스가 없으면 "default"에 코드를 생성
    // 전역:: 또는 다른 <RootNamespace> 네임스페이스
    var hasNamespace = !string.IsNullOrEmpty(nameSpace)
    if (hasNamespace)
    {
        // 여기에서 파일 범위 네임스페이스를 사용할 수 있습니다.
        // 좀 더 간단하지만 사용하지 못할 수 도 있는 C#이 필요합니다.
        // 지원하는 대상에 따라 다릅니다!
        sb
            .Append("namespace ")
            .Append(nameSpace)
            .AppendLine(@"
    {");
    }

    // 가장 바깥쪽부터 시작하여 전체 상위 유형 계층 구조를 반복
    while (parentClass is not null)
    {
        sb
            .Append("    partial ")
            .Append(parentClass.Keyword) // 예: class/struct/record
            .Append(' ')
            .Append(parentClass.Name) // 예: Outer/Generic<T>
            .Append(' ')
            .Append(parentClass.Constraints) // 예: where T: new()
            .AppendLine(@"
        {");
        parentsCount++; // 얼마나 많은 레이어가 있는지 추적
        parentClass = parentClass.Child; // 다음 자식으로 반복
    }

    // 여기에 실제 타겟 생성 코드를 작성합니다. 간결함을 위해 표시되지 않음
    sb.AppendLine(@"public partial readonly struct TestId
    {
    }");

    // 각 부모 유형을 "닫아야"하므로 필요한 수의 '}'를 작성
    for (int i = 0; i < parentsCount; i++)
    {
        sb.AppendLine(@"    }");
    }

    // 네임스페이스가 있는 경우 닫음
    if (hasNamespace)
    {
        sb.Append('}').AppendLine();
    }

    return sb.ToString();
}
```

위의 예는 완전한 예가 아니며 모든 상황에서 작동하지 않을 것이지만 여러 상황에서 유용하다는 것을 알았기 때문에 귀하에게 효과가 있을 수 있는 한 가지 가능한 접근 방식을 보여줍니다


## 요약

이 게시물에서는 소스 생성기에서 유용한 두 가지 특정 기능인 유형 선언 구문의 네임스페이스와 유형 선언 구문의 중첩된 유형 계층 구조를 계산하는 방법을 보여주었습니다. 항상 필요한 것은 아니지만 일반 부모 유형과 같은 복잡성을 처리하거나 원본과 동일한 네임스페이스에서 코드를 생성하도록 하는 데 유용할 수 있습니다. 


## 원문
Andrew Lock's [Series: Creating a source generator](https://andrewlock.net/series/creating-a-source-generator/) - [Part 5 - Finding a type declaration's namespace and type hierarchy](https://andrewlock.net/creating-a-source-generator-part-5-finding-a-type-declarations-namespace-and-type-hierarchy/)