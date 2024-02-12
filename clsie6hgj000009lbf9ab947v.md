---
title: "[Uno] KeyEquality"
datePublished: Mon Feb 12 2024 03:45:59 GMT+0000 (Coordinated Universal Time)
cuid: clsie6hgj000009lbf9ab947v
slug: uno-keyequality
tags: dotnet, unoplatform

---

.NET 단일 코드로 크로스플랫폼 앱을 만들 수 있는 [Uno Platform](https://platform.uno/)에서는 동일한 엔터티임을 비교하는 [KeyEquality](https://github.com/unoplatform/uno.extensions/blob/main/doc/Learn/KeyEquality/concept.md)를 지원합니다. 사용방법은 다음과 같습니다.

```csharp
public partial record Entity(string Id, ...)
```

속성명 `Id`는 암시적으로 키로 인식하며 다음 처럼 `KeyEquals()`로 키 동등을 비교할 수 있습니다.

```csharp
entity.KeyEquals(...);
```

키 동등은 키 이외의 값이 다르더라도 동일한 엔터티라고 판단한다는 점에서 `record`의 `Equals`와는 다르게 동작합니다.

```csharp
public partial record Person([property: Key] string Name, int Age);

var john1 = new Person("John Doe", 20);
var john2 = john1 with { Age=21 };

Console.WriteLine("Are the same : " + john1.Equals(john2));
Console.WriteLine("Are the same person : " + john1.KeyEquals(john2));
```

`Equals()`는 `false`이지만 `KeyEquals()`의 결과는 `true`가 됩니다.

```plaintext
Are the same : false
Are the same person : true
```

그렇다면 왜 `KeyEquals()`가 필요할까요? `record`로 정의된 엔터티는 엔터티의 모든 값에 대한 동등성을 평가하기 보다는 키를 기준으로 동등함과 변경됨을 판단하는 것이 유용합니다. 예를 들어 데이터베이스의 레코드를 업데이트 하는 경우가 되겠습니다.

Uno Platform에서는 이러한 키 동등성을 소스 생성기를 이용해서 자동으로 생성합니다.

```csharp
partial record Person : global::Uno.Extensions.Equality.IKeyEquatable<BibleVerseCounter.Business.Models.Person>, global::Uno.Extensions.Equality.IKeyed<string>
{

	/// <inheritdoc cref="{NS.Equality}.IKeyed{T}" />
	[global::System.CodeDom.Compiler.GeneratedCodeAttribute("KeyEqualityGenerationTool", "1")]
	string global::Uno.Extensions.Equality.IKeyed<string>.Key
	{
		get
		{
			return Name;
		}
	}

	/// <inheritdoc cref="global::Uno.Extensions.Equality.IKeyEquatable{T}" />
	[global::System.CodeDom.Compiler.GeneratedCodeAttribute("KeyEqualityGenerationTool", "1")]
	public virtual int GetKeyHashCode()
	{
		unchecked
		{
			var hash = global::System.Collections.Generic.EqualityComparer<global::System.Type>.Default.GetHashCode(EqualityContract) * -1521134295;


			hash += global::System.Collections.Generic.EqualityComparer<string>.Default.GetHashCode(Name);
			hash *= -1521134295;

			return hash;
		}
	}

	/// <inheritdoc cref="global::Uno.Extensions.Equality.IKeyEquatable{T}" />
	[global::System.CodeDom.Compiler.GeneratedCodeAttribute("KeyEqualityGenerationTool", "1")]
	public bool KeyEquals(BibleVerseCounter.Business.Models.Person? other)
	{
		if (object.ReferenceEquals(this, other))
		{
			return true;
		}

		if (object.ReferenceEquals(null, other)
			|| EqualityContract != other.EqualityContract)
		{
			return false;
		}

		return global::System.Collections.Generic.EqualityComparer<string>.Default.Equals(Name, other.Name);
	}
}
```

자동 생성된 코드를 보면 `IKeyEquatable<>` 인터페이스와 `IKeyed<string>` 인터페이스가 구현 된 것을 볼 수 있습니다.

복합 키를 정의할 수도 있는데 사용 방법은 간단합니다. `[property:Key]` 특성을 키 속성에 다음처럼 복수로 장식하면,

```csharp
public partial record CompositeKeyEntity(
    [property:Key] string Key1,
    [property:Key] string Key2,
    string Value1
    );
```

다음처럼 Uno의 자동 생성 기능으로 코드가 생성 됩니다.

```csharp
partial record CompositeKeyEntity : global::Uno.Extensions.Equality.IKeyEquatable<BibleVerseCounter.Business.Models.CompositeKeyEntity>, global::Uno.Extensions.Equality.IKeyed<(string, string)>
{

	/// <inheritdoc cref="{NS.Equality}.IKeyed{T}" />
	[global::System.CodeDom.Compiler.GeneratedCodeAttribute("KeyEqualityGenerationTool", "1")]
	(string, string) global::Uno.Extensions.Equality.IKeyed<(string, string)>.Key
	{
		get
		{
			return (Key1, Key2);
		}
	}

	/// <inheritdoc cref="global::Uno.Extensions.Equality.IKeyEquatable{T}" />
	[global::System.CodeDom.Compiler.GeneratedCodeAttribute("KeyEqualityGenerationTool", "1")]
	public virtual int GetKeyHashCode()
	{
		unchecked
		{
			var hash = global::System.Collections.Generic.EqualityComparer<global::System.Type>.Default.GetHashCode(EqualityContract) * -1521134295;


			hash += global::System.Collections.Generic.EqualityComparer<string>.Default.GetHashCode(Key1);
			hash *= -1521134295;

			hash += global::System.Collections.Generic.EqualityComparer<string>.Default.GetHashCode(Key2);
			hash *= -1521134295;

			return hash;
		}
	}

	/// <inheritdoc cref="global::Uno.Extensions.Equality.IKeyEquatable{T}" />
	[global::System.CodeDom.Compiler.GeneratedCodeAttribute("KeyEqualityGenerationTool", "1")]
	public bool KeyEquals(BibleVerseCounter.Business.Models.CompositeKeyEntity? other)
	{
		if (object.ReferenceEquals(this, other))
		{
			return true;
		}

		if (object.ReferenceEquals(null, other)
			|| EqualityContract != other.EqualityContract)
		{
			return false;
		}

		return global::System.Collections.Generic.EqualityComparer<string>.Default.Equals(Key1, other.Key1)
			&& global::System.Collections.Generic.EqualityComparer<string>.Default.Equals(Key2, other.Key2);
	}
}
```