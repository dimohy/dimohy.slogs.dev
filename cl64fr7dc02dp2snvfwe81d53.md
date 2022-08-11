## 클린 코드 팁: 더 나은 단위 테스트를 위한 f.i.r.s.t. 약어 (번역)

> CODE4IT의 [Clean Code Tip: F.I.R.S.T. acronym for better unit tests](https://www.code4it.dev/cleancodetips/f-i-r-s-t-unit-tests)를 번역하여 공유합니다.

FIRST는 깨끗하고 확장 가능한 테스트를 작성하려는 경우 항상 기억해야 하는 약어입니다.

이 약어는 단위 테스트가 신속하고(Fast), 독립적(Independent)이며, 반복 가능(Repeatable)하고, 자체 검증(Self-validating)되고, 철저(Thorough)해야 함을 알려줍니다.


## 신속 (Fast)

설정과 시작 단계에서 오랜 시간이 필요한 테스트를 생성해서는 안됩니다: 이상적으로 전체 테스트 모음을 1분 이내에 실행할 수 있어야 합니다.

만약 단위 테스트를 실행하는데 너무 많은 시간이 걸린다면 뭔가 문제가 있는 것입니다. 많은 가능성이 있습니다:

1. 원격 소스 (실제 API, 데이터베이스 등)에 액세스 하려고 합니다: 테스트를 더 빠르게 만들고 실제 리소스에 액세스하지 않도록 이런 종속성을 mock 해야 합니다. 실제 데이터가 필요한 경우 통합/e2e 테스트를 만드는 것이 좋습니다.

1. 테스트 중인 시스템이 빌드 되기에는 복잡합니다: 종속성이 너무 많나요? [DIT Value](https://www.code4it.dev/blog/measure-maintainability-with-ndepend#depth-of-inheritance-tree-dit)가 너무 높습니까?

1. 테스트 중인 메서드가 너무 많은 작업을 수행합니다. 별도 독립적인 메서드로 분할할 것을 고려하고 호출자가 필요에 따라 메서드 호출을 [오케스트레이션](https://ko.wikipedia.org/wiki/%EC%98%A4%EC%BC%80%EC%8A%A4%ED%8A%B8%EB%A0%88%EC%9D%B4%EC%85%98_(%EC%BB%B4%ED%93%A8%ED%8C%85)) 하도록 해야 합니다.


## 독립적 (Independent) 또는 격리됨 (Isolated)

테스트 메서드는 다른 것과 독립적이여야 합니다.

다음과 같이 하는 것은 피하세요:

```csharp
MyObject myObj = null;

[Fact]
void Test1()
{
    myObj = new MyObject();
    Assert.True(string.IsNullOrEmpty(myObj.MyProperty));

}

[Fact]
void Test2()
{

    myObj.MyProperty = "ciao";
    Assert.Equal("oaic", Reverse(myObj.MyProperty));
}
```

여기에서 Test2가 올바르게 작동하려면 Test1이 먼저 실행되어야 합니다. 그렇지 않으면 `myObj`이 널 이 됩니다. Test1과 Test2는 종속성이 있습니다.

어떻게 그것을 피합니까? 모든 테스트에 새 인스턴스를 생성하세요! 일부 사용자 정의 메서드 또는 시작 단계에 있을 수 있습니다. 그리고 **모의 항목(mock)도 재설정해야 한다는 것을 기억**합시다.


## 반복 가능 (Repeatable)

단위 테스트는 반복 가능해야 합니다. 이 의미는 언제 어디서나 실행될 때 바르게 작동해야 합니다.

따라서 파일 시스템, 현재 날짜 등의 종속성을 제거애햐 합니다.

이 테스트로 예를 들어 봅시다.

```csharp
[Fact]
void TestDate_DoNotDoIt()
{

    DateTime d = DateTime.UtcNow;
    string dateAsString = d.ToString("yyyy-MM-dd");
    
    Assert.Equal("2022-07-19", dateAsString);
}
```

이 테스트는 현재 날짜에 엄격하게 구속됩니다. 만약 한달 후에 테스트를 다시 진행한다면 이것은 실패할 것입니다.

대신 종속성을 제거하고 더미 값이나 mock을 사용해야 합니다.

```csharp
[Fact]
void TestDate_DoIt()
{

    DateTime d = new DateTime(2022,7,19);
    string dateAsString = d.ToString("yyyy-MM-dd");

    Assert.Equal("2022-07-19", dateAsString);
}
```

.NET에 DateTime(및 유사한 종속성)을 주입하는 많은 방법이 있습니다. 이 기사에서 몇 가지를 나열합니다: ["3 ways to inject DateTime and test it"](https://www.code4it.dev/blog/inject-and-test-datetime-dependency/)


## 자체 검증 (Self-validating)

자체 검증은 테스트가 작업을 수행하고 결과를 프로그래밍 방식으로 확인해야 함을 의미합니다.

예를 들어 파일에 무언가를 기록했는지 테스트 할 때 테스트 자체가 올바르게 동작하는지 확인하는 역할을 합니다. **수동 조작을 해서는 안됩니다**

또한 테스트는 명시적인 피드백을 제공해야 합니다: 테스트는 통과하거나 실패합니다. 그 사이는 없습니다.


## 철저한 (Thorough)

단위 테스트는 행복한 진행과 실패한 진행 모두를 검증해야 한다는 점에서 철저해야 합니다.

따라서 유효한 입력과 잘못된 입력으로 함수를 테스트해야 합니다.

또한 테스트를 진행하면서 발생하는 예외로 인해 어떤 일이 일어나는지 검사해야 합니다. 오류를 올바르게 처리하고 있나요?

단순한 하나의 메서드를 가진 클래스를 살펴봅시다:

```csharp
public class ItemsService
{

    readonly IItemsRepository _itemsRepo;

    public ItemsService(IItemsRepository itemsRepo)
    {
        _itemsRepo = itemsRepo;
    }
    
    public IEnumerable<Item> GetItemsByCategory(string category, int maxItems)
    {

        var allItems = _itemsRepo.GetItems();

        return allItems
                .Where(i => i.Category == category)
                .Take(maxItems);
    }
}
```

`GetItemsByCategory`에 대해 어떤 테스트를 작성해야 할까요?

- `category`가 널 이거나 비었으면?
- `maxItems`가 0보다 작으면?
- 만약 `allItems`가 널 이라면?
- `allItems` 항목 중 하나가 널 이라면?
- `_itemsRepo.GetItems()`이 예외를 발생한다면?
- `_itemsRepo`가 널 이라면?

보시다시피 이런 간단한 메서드의 경우에도 누락된 항목이 없는지 확인하기 위해 많은 테스트를 작성해야 합니다.


## 마무리

F.I.R.S.T는 좋은 단위 테스트 모음을 만들 수 있는 성질을 기억하는 좋은 방법입니다.

항상 그것을 고수하려고 노력하고 [테스트는 프로덕션 코드보다 훨씬 잘 작성되어야 함](https://www.code4it.dev/cleancodetips/tests-should-be-readable-too)을 기억합시다.

즐거운 코딩!

🐧
