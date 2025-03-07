---
title: "nullable 참조 형식 적응하기"
datePublished: Sun Aug 22 2021 02:54:05 GMT+0000 (Coordinated Universal Time)
cuid: cksmm9nou0taz2xs14xgvh1ol
slug: nullable-reference-types
tags: csharp

---

이제까지 강제 사항은 아니지만 C# 8부터 [nullable 참조 형식](https://docs.microsoft.com/ko-kr/dotnet/csharp/nullable-references)을 사용할 수 있었습니다.

그런데 `.NET 6 Preview 7`부터 생성하는 `.NET 6` 프로젝트 템플릿에 `nullable 참조형식`이 기본 `enable` 되어 있습니다.

```xaml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    ...
    <Nullable>enable</Nullable>
    ...
  </PropertyGroup>
  ...
</Project>
```

C# 8부터 도입이 되어서 저 기능을 알고는 있었지만 적응이 안돼 안쓰고 있었는데, 이제는 적응해야 겠군요;

`nullable 참조형식`이 `enable`되면 기본적인 참조형은 null일 수 있을 경우 `CS8618` 경고가 발생합니다. 그 값이 null값이 될 수 있다는 경고인데요, 이런 경고는 null 관련 버그를 예방할 수 있는 좋은 기능이기는 하나 적응이 필요합니다. 이제 앞으로 null일 수 있는 참조형은 예를 들어 `string`의 경우 명시적으로 `string?`라고 써주는게 좋습니다.

다음의 코드들을 예로 보시죠.

```csharp
class NullableTestClass
{
   public string Name { get; set; } // CS8618 경고 발생
}
```

"응 아니야~ 참조형이 null일 수 있을 경우 명시적으로 표시해야 돼!" 라고 경고를 줍니다. 흠...;; 익숙해질 때까지 조금 시간이 걸리는데요, 코드를 저 속성 값이 null일 수 없게 다음 처럼 수정할 수도 있고,

```csharp
class NullableTestClass
{
    public string Name{ get; }

    public NullableTestClass(string name)
    {
        name = stringValue;
    }
};
```

또는 초기값을 줄 수 도 있고,

```csharp
class NullableTestClass
{
   public string Name { get; set; } = "Default Name";
}
```

아니면 Name 속성이 null일 수 있음을 `string?` 처럼 명시할 수 있습니다.

```csharp
class NullableTestClass
{
   public string? Name { get; set; } // CS8618 경고 발생 하지 않음
}
```

그러고 보니 어느사이 .NET 기본 라이브러리가 `nullable 참조 형식`에 맞게 변해 있군요.
String 클래스를 예로 보자면, 인자가 null일 수 있는 것은 명시적으로 `?`을 붙이고 null이 아니어야 하는 경우 `?`이 빠진 모습을 볼 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1629600789060/FqGsyV98R.png)

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1629600798218/bVXkZWEqS.png)
