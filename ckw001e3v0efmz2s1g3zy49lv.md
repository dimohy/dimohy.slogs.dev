---
title: ".NET Foundation 프로젝트 소개(1): Verify"
datePublished: Mon Nov 15 2021 01:39:41 GMT+0000 (Coordinated Universal Time)
cuid: ckw001e3v0efmz2s1g3zy49lv
slug: net-foundation-1-verify
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1636940629539/THJzyb_xL.jpeg
tags: dotnet

---

여러분의 시간을 아낄 수 있는 .NET Foundation의 유용한 프로젝트 소개를 소개하는 시간입니다.

오늘 소개하는 프로젝트는 [Verify](https://github.com/VerifyTests/Verify)인데요, Verify는 복잡한 데이터 모델 및 문서의 어설션을 단순화하는 스냅숏 도구입니다.

어설션 단계에서 테스트 결과에 대한 확인이 호출됩니다. 해당 결과를 직렬화 하고 테스트 이름과 일치하는 파일에 저장합니다. 다음 테스트 실행에서 결과는 다시 직렬화 되고 기존 파일과 비교됩니다. 두 스냅숏이 일치하지 않거나 변경이 예기치 않거나 참조 스냅숏을 새 결과로 업데이트 해야 하는 경우 테스트가 실패합니다.


## NuGet 패키지

Verify는 다음의 테스트 도구에 대한 NuGet 패키지를 제공합니다.

- [Verify.Xunit](https://nuget.org/packages/Verify.Xunit/)
- [Verify.NUnit](https://nuget.org/packages/Verify.NUnit/)
- [Verify.Expecto](https://nuget.org/packages/Verify.Expecto/)
- [Verify.MSTest](https://nuget.org/packages/Verify.MSTest/)


## 스냅샷 관리

스냅샷 파일을 수락하거나 거절하는 것은 Verify의 핵심 워크플로의 일부입니다. 이 작업을 수행하는 방법에는 여러 가지가 있으며 개인 기본 설정에 의해 접근할 수 있습니다.
- [DiffEngineTray](https://github.com/VerifyTests/DiffEngine/blob/main/docs/tray.md)의 윈도우 트레이
- [ReShaper 테스트 러너](https://plugins.jetbrains.com/plugin/17241-verify-support) 지원
- [Rider 테스트 러너](https://plugins.jetbrains.com/plugin/17240-verify-support) 지원
- [클립보드](https://github.com/VerifyTests/Verify/blob/main/docs/clipboard.md)를 통해
- 실행된 [diff 도구](https://github.com/VerifyTests/DiffEngine#supported-tools)에서 수동으로 변경합니다. 복사 붙여 넣기를 하거나 일부 도구에는 바로 가기 또는 버튼을 통해 자동화 하는 명령이 있습니다.
- `.received.` 파일을 `.verified.`로 이름 변경을 해서 파일 시스템에서 수동으로 할 수 있습니다. 스크립트를 통해 `.received.` 파일을 일괄 수락하도록 자동화 할 수 있습니다.

## 사용법

### 테스트 중일 때

테스트할 클래스:
```csharp
public static class ClassBeingTested
{
    public static Person FindPerson()
    {
        return new()
        {
            Id = new("ebced679-45d3-4653-8791-3d969c4a986c"),
            Title = Title.Mr,
            GivenNames = "John",
            FamilyName = "Smith",
            Spouse = "Jill",
            Children = new()
            {
                "Sam",
                "Mary"
            },
            Address = new()
            {
                Street = "4 Puddle Lane",
                Country = "USA"
            }
        };
    }
}
```

### xUnit

```csharp
[UsesVerify]
public class Sample
{
    [Fact]
    public Task Test()
    {
        var person = ClassBeingTested.FindPerson();
        return Verifier.Verify(person);
    }
}
```

### NUnit

```csharp
[TestFixture]
public class Sample
{
    [Test]
    public Task Test()
    {
        var person = ClassBeingTested.FindPerson();
        return Verifier.Verify(person);
    }
}
```

### Expecto

```fsharp
open Expecto
open VerifyTests
open VerifyExpecto

[<Tests>]
let tests =
  testTask "findPerson" {
    let person = ClassBeingTested.FindPerson();
    do! Verifier.Verify("findPerson", person)
  }
```

#### 주의사항
Expecto 구현 특성으로 Verify의 다음 API는 지원되지 않습니다.
- `settings.UseTypeName()`
- `settings.UseMethodName()`
- `settings.UseParameters()`
- `settings.UseTextForParameters()`
대신 사용자 `name` 매개변수를 사용하세요.

### MSTest

```csharp
[TestClass]
public class Sample :
    VerifyBase
{
    [TestMethod]
    public Task Test()
    {
        var person = ClassBeingTested.FindPerson();
        return Verify(person);
    }
}
```

### 초기 검증

테스트가 처음 실행되면 다음과 같이 실패합니다.

```
First verification. Sample.Test.verified.txt not found.
Verification command has been copied to the clipboard.
```

검증 할 `.verified.` 파일이 없어 발생한 것으로 `.received.` 파일을 이용해 `.verified.` 파일을 생성해야 합니다.

[클립보드](https://github.com/VerifyTests/Verify/blob/main/docs/clipboard.md)에 자동으로 다음이 포함됩니다.

```
cmd /c move /Y "C:\Code\Sample\Sample.Test.received.txt" "C:\Code\Sample\Sample.Test.verified.txt"
```

클립보드 대신 [DiffEngineTray 도구](https://github.com/VerifyTests/DiffEngine/blob/master/docs/tray.md)를 사용할 수 있습니다.

[Diff 도구](https://github.com/VerifyTests/DiffEngine)를 사용할 경우 다음과 같이 감지됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1636939453066/m0wp5g3fX.png)

테스트 결과로 검증 상태를 만들기 위해서 다음과 같이 할 수 있습니다.
- 클립보드의 명령을 실행
- diff 도구를 사용해서 변경 사항을 수락
- 텍스트를 새 파일에 수동으로 복사

### 후속 검증

후속 검증을 확인하기 위해 `FindPerson()`을 다음과 같이 변경하고 테스트를 실행하면,

```csharp
public static class ClassBeingTested
{
    public static Person FindPerson()
    {
        return new()
        {
            Id = new("ebced679-45d3-4653-8791-3d969c4a986c"),
            Title = Title.Mr,
            // Middle name added
            GivenNames = "John James",
            FamilyName = "Smith",
            Spouse = "Jill",
            Children = new()
            {
                "Sam",
                "Mary"
            },
            Address = new()
            {
                // Address changed
                Street = "64 Barnett Street",
                Country = "USA"
            }
        };
    }
}
```

다음처럼 실패합니다.

```
Verification command has been copied to the clipboard.
Assert.Equal() Failure
                                  ↓ (pos 21)
Expected: ···\n  GivenNames: 'John',\n  FamilyName: 'Smith',\n  Spouse: 'Jill···
Actual:   ···\n  GivenNames: 'John James',\n  FamilyName: 'Smith',\n  Spouse:···
                                  ↑ (pos 21)
```

클립보드에는 다시 자동으로 다음이 포함됩니다.

```
cmd /c move /Y "C:\Code\Sample\Sample.Test.received.txt" "C:\Code\Sample\Sample.Test.verified.txt"
```

diff 도구는 변경된 부분을 다음처럼 표시합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1636939823840/ctaXaRa6j.png)

동일한 접근 방식으로 결과를 확인할 수 있으며 `ClassBeingTested`의 변경과 함께 `Sample.Test.verified.txt`가 소스 제어에 의해 커밋됩니다.

## 마무리

Verify를 잘 활용하면 복잡한 모델이나 문서에 대한 어설션을 단순화 할 수 있습니다. `VerifyJson` 기능으로 JSON도 지원합니다. 

좀 더 자세한 내용은 [Verify의 GitHub 페이지](https://github.com/VerifyTests/Verify)를 참조하세요. 
