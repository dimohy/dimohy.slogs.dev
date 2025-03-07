---
title: "부분 메서드(partial method) 사용해보기"
datePublished: Wed Dec 01 2021 02:38:59 GMT+0000 (Coordinated Universal Time)
cuid: ckwmx7a7804ax2ds1fky97aj3
slug: partial-method
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1640841302232/7gClIlL_c.png
tags: csharp

---

C#은 [부분 메서드(partial method)](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/partial-method)를 지원합니다. `partial class`의 경우 UI의 코드를 자동 생성해주는 WinForms나 WPF에서 이미 사용하는 방법인데요, 부분 메서드(partial method)는 좀 낯섭니다.

부분 메서드는 다음의 경우 유용할 수 있습니다.

- 서브 클래스로 모듈화 하기에는 복수개의 서브 인스턴스가 생길 일이 없어 단일 클래스로 표현하고 싶을 때
- 소스 생성기를 이용하고자 할 때

먼저 소스 생성기를 사용하고자 하는 경우를 살펴볼께요.

특성으로 소스 생성기에 해당 메소드가 자동 생성되는 것을 알리고 다음처럼 표현할 수 있습니다.

```csharp
[RegexGenerated("(dog|cat|fish)")]
partial bool IsPetMatch(string input);
```

해당 메소드는 구현되지 않았지만 소스 생성기에 의해 다음 형태로 코드가 생성될 것입니다.

| 자동 생성됨
```csharp
patial class UserClass
{
   ...
   partial bool IsPetMatch(string input)
   {
      ...
   }
   ...
}
```

마치 내가 코드를 작성한 것 처럼 다음처럼 사용할 수 있게 됩니다.

```csharp
var inst = new UserClass();
var result = inst.IsPetMatch("dog");
```

첫 번째의 경우는,

필요에 따라 동일한 `partial class DatabaseSynchronizer` 선언으로 파일단위로 모듈을 나누고 

- DatabaseSynchronizer.cs
- DatabaseSynchronizer.Forward.cs
- DatabaseSynchronizer.Reverse.cs

Forward와 Reverse의 기능을 구현한 뒤 `DatabaseSynchronizer.cs`에 구현한 메소드를 `partial method` 선언으로 표현할 수 있습니다.

| DatabaseSynchronizer.Forward.cs
```csharp
...
public partial DDLSyntaxTree Forward(Model model)
{
   ...
}
```

그리고 ` DatabaseSynchronizer.cs`는 선언만 해주는 것으로,

```csharp
...
public partial DDLSyntaxTree Forward(Model model);
...
```

어떤 메서드를 제공 하는지 알 수 있도록 해줍니다.

이렇게 하면 좀 더 소스 코드 관리가 쉬워지고 각각의 기능은 결국에는 합쳐지겠지만 구분된 파일에 의해 좀 더 구조적으로 코딩이 가능해 집니다.