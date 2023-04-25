---
title: "10가지 멋진 C# 리팩토링 팁 | Assis Zang"
datePublished: Tue Apr 25 2023 22:48:36 GMT+0000 (Coordinated Universal Time)
cuid: clgwuzaqj000a0al23ch2hdpi
slug: 10-awesome-csharp-refactoring-tips
canonical: https://www.telerik.com/blogs/10-awesome-csharp-refactoring-tips
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1682462911987/4223d241-e27c-49e7-a7b6-fc5f8bb34af9.png
tags: net, dotnet

---

> [Assis Zang](https://www.telerik.com/blogs/author/assis-zang)님의 [10 Awesome C# Refactoring Tips](https://www.telerik.com/blogs/10-awesome-csharp-refactoring-tips)을 번역하였습니다.

---

한 가지 확실하게 말할 수 있는 것은 코드가 생성되는 곳에는 리팩터링도 있다는 것입니다. 특히 매년 여러 차례 업데이트되는 .NET과 같은 Microsoft 기술의 경우, 오늘 작성된 코드는 내일의 유산이 될 수 있습니다.

개발자는 자신이 사용하는 기술의 최신 개선 사항을 항상 파악하고 이를 최대한 활용해야 합니다. 몇 년 또는 몇 달 안에 구식이 되지 않을 무언가를 만드는 것은 불가능합니다. 따라서 코드베이스가 하나 더 생기면 레거시가 되어 리팩터링 대상이 됩니다.

리팩터링을 효율적으로 수행하려면 코드를 재구성하는 방법과 Visual Studio IDE를 사용하는 방법 등 여러 가지 리소스가 도움이 될 수 있습니다. 효율적인 리팩터링에 도움이 되는 10가지 팁을 알아보려면 계속 읽어보세요.

## 리팩터링의 중요성

리팩터링은 기본적으로 코드의 동작을 변경하지 않는 방식으로 코드에 작은 변형을 수행하는 기술로 구성됩니다. 이러한 변형은 작지만 코드를 유지 관리하기 쉽게 만들고 애플리케이션의 성능을 개선하기 위한 것입니다.

리팩터링은 프로그래머가 기존 코드에서 성능의 실제 가치를 입증할 수 있는 방법이기 때문에 개발 팀에서 매우 중요합니다. 또한 코드 리팩터링에는 다음과 같은 장단기적인 이점이 많이 있습니다:

* **민첩성**: 잘 작성된 코드는 개발자가 코드의 문제를 더 빨리 식별하고 해결할 수 있게 해줍니다.
    
* **사용자 경험**: 충돌이 줄어들고 성능이 향상되면 소프트웨어가 사용하기 쉬워지고 최종 사용자에게 더 나은 경험을 제공할 수 있습니다.
    
* **현대화**: 리팩터링된 코드는 이미 시장에 통합된 최첨단 기술을 사용하기 때문에 라이브러리를 현대화하고 코드를 소유한 회사가 경쟁사보다 앞서 나갈 수 있는 가능성을 제공합니다.
    

## C#에서 리팩터링

C# 프로그래밍 언어는 항상 끊임없이 진화해 왔으며, 최근 C#을 형식은 줄이고 성능은 향상시킨 언어로 바꾸려는 Microsoft의 노력이 눈에 띕니다.

새로운 시스템의 경우 최신 버전의 C#에서 제공하는 모든 최신 기능을 자유롭게 사용할 수 있지만, 레거시 코드는 어떻게 되나요? 레거시 코드는 이러한 기능 중 상당수가 아직 존재하지 않거나 어떤 이유로 인해 사용되지 않을 때 작성된 코드입니다. 이러한 경우 리팩터링이 필요합니다.

| **레거시 애플리케이션 단위 테스트** | 레거시 코드를 확장하고 개선하는 데 대부분의 시간을 할애할 때 단위 테스트를 새 애플리케이션에 적용하는 방법에 대한 기사를 읽는 데 지치셨다면, [기존 애플리케이션으로 작업할 때 자동화된 테스트를 (최종적으로) 활용하기 위한 계획이 여기 있습니다.](https://www.telerik.com/blogs/unit-testing-legacy-applications-dealing-legacy-code-justmock) 특히 Visual Studio와 JustMock에 맡기면 생각보다 쉽게 만들 수 있습니다. |
| --- | --- |

다음은 C#에서 코드를 리팩터링하기 위한 팁 목록입니다. 먼저 잘못 작성된 코드를 살펴본 다음 더 나은 리팩터링 버전을 만드는 방법을 공유합니다. 또한 일부 예제에서는 Visual Studio 기능을 사용하여 리팩터링을 더 빠르게 수행하는 방법에 대한 팁을 강조합니다.

[이 링크에서 모든 예제가 포함된 소스 코드](https://github.com/zangassis/refactoring-csharp-tips)에 액세스할 수 있습니다.

## `1. foreach 대신 LINQ를 사용`

`foreach`는 프로그래머가 값 목록을 필터링하는 데 사용하는 가장 일반적인 방법 중 하나입니다. 그러나 실행은 효율적이지만 목록 내에서 특정 값을 찾으려면 예를 들어 `if` 및 `else`와 같은 조건 연산자를 사용해야 하기 때문에 구문이 일반적으로 코드를 오염시킵니다. 코드를 더 읽기 쉽게 만들려면 아래 예제에서 볼 수 있듯이 LINQ에서 사용할 수 있는 리소스를 사용하는 것이 좋습니다.

* 리팩토링 전
    

```csharp
var sellers = Seller.AllSellers();
var smallSellers = new List<Seller>();

foreach (var seller in sellers)
{
    if (seller.SmallSeller)
    {
        smallSellers.Add(seller);
    }
}
```

* 리팩토링 후
    

```csharp
var sellers = Seller.AllSellers();
var smallSellers = (from seller in sellers where seller.SmallSeller select seller).ToList();
```

또는

```csharp
var sellers = Seller.AllSellers();
var smallSellers = (sellers.Where(seller => seller.SmallSeller)).ToList();
```

💡Visual Studio를 사용하는 경우 리팩터링 작업을 대신 수행해 주는 리소스를 사용할 수 있습니다. `foreach` 코드 위에 커서를 올려놓고 전구 아이콘을 클릭하면 "LINQ로 변환" 옵션이 나타납니다. 이렇게 하면 아래 이미지와 같이 리팩터링 결과를 미리 볼 수 있습니다:

![Visual Studio refactoring](https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2022/2022-11/vs-refactoring.png?sfvrsn=5ba031e9_3 align="left")

## `2. 메서드에 단일 책임 할당`

리팩터링에서 발생하는 매우 일반적인 문제는 둘 이상의 책임이 있는 메서드, 즉 여러 작업을 수행하는 데 사용되어 코드를 혼란스럽고 이해하기 어렵게 만드는 메서드입니다. 하나의 책임이 있는 메서드를 만들면 코드를 정리하고 더 깔끔하고 이해하기 쉽게 만들 수 있습니다.

아래 예제에서는 첫 번째 버전에서 여러 가지 작업을 담당하는 메서드가 있습니다.

먼저 모든 판매자를 반환하기 위해 검색을 수행합니다. 그런 다음 새 판매자의 필수 입력란이 채워져 있는지 확인합니다. 필드 중 하나라도 비어 있으면 오류 메시지가 반환됩니다. 그렇지 않은 경우 이 판매자가 목록에 이미 존재하는지 확인하고, 이미 존재하면 오류 메시지가 반환되지만 그렇지 않은 경우 새 판매자가 데이터베이스에 추가됩니다.

```csharp
    // Before refactoring
    public string CreateSeller(Seller seller)
    {
        var sellers = _context.Sellers;

        string requiredFieldsMessage = string.Empty;

        if (string.IsNullOrEmpty(seller.Name))
        {
            requiredFieldsMessage += "Name is required";
        }
        if (string.IsNullOrEmpty(seller.ContactEmail))
        {
            requiredFieldsMessage += "Email is mandatory";
        }
        if (!string.IsNullOrEmpty(requiredFieldsMessage))
            return requiredFieldsMessage;

        if (sellers.Contains(seller))
            return "Seller already exists";

        _context.Add(seller);
        _context.SaveChanges();
        return "Success";
    }
```

여기에 각각 고유한 책임이 있는 별도의 방법을 만들 수 있는 기회가 있습니다.

이를 위해 세 가지 새로운 방법을 만들 것입니다. 또한 주요 메소드의 이름을 변경하여 주요 기능에 더 적합하도록 변경할 것입니다.

```csharp
    // After refactoring
    public string CreateSellerProcess(Seller seller)
    {
        bool sellerAlreadyExists = SellerAlreadyExistsVerify(seller);

        if (sellerAlreadyExists)
            return "Seller already exists";

        string requiredFieldsMessage = ValidateFields(seller);

        if (!string.IsNullOrEmpty(requiredFieldsMessage))
            return requiredFieldsMessage;

        CreateNewSeller(seller);
        return "Success";
    }

    public bool SellerAlreadyExistsVerify(Seller seller)
    {
        var sellers = Seller.AllSellers();

        return sellers.Contains(seller);
    }

    public string ValidateFields(Seller seller)
    {
        string requiredFieldsMessage = string.Empty;

        if (string.IsNullOrEmpty(seller.Name))
        {
            requiredFieldsMessage += "Name is required";
        }
        if (string.IsNullOrEmpty(seller.ContactEmail))
        {
            requiredFieldsMessage += "Email is mandatory";
        }
        return requiredFieldsMessage;
    }

    public void CreateNewSeller(Seller seller)
    {
        _context.Add(seller);
        _context.SaveChanges();
    }
```

이번 새 버전에서는 각 메서드에 대한 책임이 분리되었습니다. 판매자가 데이터베이스에 이미 존재하는지 확인하는 메서드와 입력된 필드의 유효성을 검사하는 메서드가 있습니다. 마지막으로, 이전 유효성 검사를 통과한 경우 레코드를 저장하는 메서드가 있습니다. 따라서 더 정교한 유효성 검사를 통해 코드를 훨씬 더 이해하기 쉽고 일관성 있게 만들 수 있었습니다.

## `3. 동기식에서 비동기식으로 이동`

비동기 프로그래밍은 빠른 작업이 완료되는 동안 백그라운드에서 긴 작업을 실행할 수 있다는 점에서 큰 장점이 있습니다. 종종 상당한 시간이 소요될 수 있는 시나리오 중 하나는 데이터베이스의 트랜잭션이므로 항상 [비동기 함수](https://www.telerik.com/blogs/making-asynchronous-breakfast-dotnet)를 사용하는 것이 좋습니다.

아래 예제에서는 먼저 데이터베이스의 레코드를 동기적으로 업데이트하는 예제를 살펴봅니다. 데이터베이스에 레코드가 적은 경우 이 실행은 아마도 빠를 것입니다. 그러나 레코드가 많으면 시간이 걸릴 수 있으며, 이 경우 다른 작업이 시작될 때까지 기다려야 합니다. 이는 성능이 좋지 않습니다.

```csharp
// Sync example
public void UpdateSellers(Seller seller)
{
    var sellerEntity = _context.Sellers.Find(seller.Id);

    _context.Entry(sellerEntity).CurrentValues.SetValues(seller);
}
```

아래에서는 동일한 메서드를 비동기적으로 수행한 것을 볼 수 있습니다. 이 예제에서는 네이티브 비동기 메서드가 있는 ORM 엔티티 프레임워크 코어를 사용하고 있다는 점에 유의하세요. 이렇게 하면 코드에서 다른 작업을 잠그지 않기 때문에 데이터베이스의 지속성이 더욱 향상됩니다.

```csharp
// Async example
public async void UpdateSellersAsync(Seller seller)
{
    var sellerEntity = await _context.Sellers.FindAsync(seller.Id);
      _context.Entry(sellerEntity).CurrentValues.SetValues(seller);
      _context.SaveChangesAsync();
}
```

## `4. 생성자 생성으로 클래스 및 값 단순화하기`

생성자를 사용하는 것은 좋은 습관으로 간주됩니다. 이를 통해 기본값을 정의하고, 액세스를 수정하고, 클래스를 인스턴스화하는 데 필요한 값을 보다 명시적으로 만들 수 있을 뿐만 아니라 클래스가 호출될 때 보다 깔끔한 코드를 유지할 수 있습니다.

아래 예제에서 생성자 없이 클래스가 어떻게 인스턴스화되는지 살펴보세요. 생성자가 있는 클래스는 필드를 선언할 필요가 없고 매개변수로 값을 전달하기만 하면 되므로 더 간단합니다.

```csharp
    public void OrderProcess()
    {
        var orderWithoutConstructor = new OrderWithoutConstructor()
        {
            CustomerId = "14797513080",
            ProductId = "p69887424099",
            Value = 100
        };

        var orderWithConstructor = new OrderWithConstructor("14797513080", "p69887424099", 100);
    }
```

💡Visual Studio를 사용하는 경우 아래 GIF와 같이 퀵 액션을 통해 빌더를 생성할 수 있습니다.

![Generate Constructor](https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2022/2022-11/generate-constructor.gif?sfvrsn=6e3e1534_3 align="left")

## `5. If/Else 체인 제거`

`if` 및 `else` 체인은 리팩터링에서 매우 일반적이며, 작동은 하지만 코드를 더럽고 이해하기 어렵게 만듭니다. 이러한 문제를 방지하려면 기본 C# 함수인 `switch`를 사용하는 것이 대안이 될 수 있습니다.

`switch`를 사용하면 `if`와 동일한 결과를 얻을 수 있지만 보다 체계적이고 이해하기 쉬운 방식으로 결과를 얻을 수 있습니다. 아래는 동일한 예제로, 먼저 `if` 조건을 사용하고 두 번째는 `switch`를 사용합니다.

* `if`와 `else` 체인
    

```csharp
if (customer.Step == Steps.Start)
        {
            //Do something 
        }
        if (customer.Step == Steps.InsertPhoneNumber)
        {
            //Do something 
        }
        if (customer.Step == Steps.PhoneNumberOrEmailToVerify)
        {
            //Do something 
        }
        if (customer.Step == Steps.VerifyToken)
        {
            //Do something 
        }
        if (customer.Step == Steps.DownloadApp)
        {
            //Do something 
        }
        if (customer.Step == Steps.WithLogin)
        {
            //Do something 
        }
        if (customer.Step == Steps.Finished)
        {
            //Do something 
        }
```

* `switch` 사용
    

```csharp
switch (customer.Step)
        {
            case Steps.Start:
                //Do something 
                break;
            case Steps.InsertPhoneNumber:
                //Do something 
                break;
            case Steps.PhoneNumberOrEmailToVerify:
                //Do something 
                break;
            case Steps.VerifyToken:
                //Do something 
                break;
            case Steps.DownloadApp:
                //Do something 
                break;
            case Steps.WithLogin:
                //Do something 
                break;
            case Steps.Finished:
                //Do something 
                break;
            default:
                //Do something 
                break;
        }
```

💡비주얼 스튜디오에서는 전체 `switch` 구조를 자동으로 채우는 기능을 사용할 수 있습니다. `switch` 조건을 입력하고 아래를 클릭하면 아래 GIF에서 볼 수 있듯이 전체 구조가 만들어집니다:

![Generate Switch](https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2022/2022-11/generate-switch.gif?sfvrsn=d7ad08e9_3 align="left")

## `6. 인터페이스 생성`

C#의 인터페이스는 컨트랙트를 나타내는 객체로, 인터페이스를 구현하는 클래스에서 인터페이스의 동작이 준수되도록 보장합니다.

리팩터링에서 인터페이스를 구현하지 않는 클래스를 찾는 것은 매우 흔한 일입니다. 이 문제를 해결하기 위해 아래 예제와 같이 선택한 클래스를 기반으로 인터페이스를 자동으로 생성하는 추출 인터페이스라는 Visual Studio 함수를 사용할 수 있습니다:

![Generate Interface](https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2022/2022-11/generate-interface.gif?sfvrsn=a4947ddf_3 align="left")

## `7. 불필요한 변수 제거`

변수는 유용하지만 불필요한 경우가 많고 코드를 오염시켜 읽기 어렵게 만드는 경우가 많습니다.

아래 예제에서 볼 수 있듯이 삼항 연산자(?)나 `Any()` 메서드와 같은 LINQ 기능을 사용할 수 있는 경우가 많으므로 변수를 꼭 사용해야 하는지 항상 고려하세요:

* 리팩토링 전
    

```csharp
    public bool SmallSellerVerify(List<Seller> sellers)
    {
        var result = false;

        foreach (var seller in sellers)
        {
            if (seller.SmallSeller == true)
            {
                result = true;
            }
            else
            {
                result = false;
            }
        }
        return result;
    }
```

* 리팩토링 후
    

```csharp
//Using Any() and ternary operator (?)

public bool SmallSellerVerifyRefactored(List<Seller> sellers) => sellers.Any(s => s.SmallSeller) ? true : false;

//Using only Any()
public bool SmallSellerVerifyRefactoredTwo(List<Seller> sellers) => sellers.Any(s => s.SmallSeller);
```

## `8. 효율적인 이름 바꾸기`

올바른 리소스를 사용하는 방법을 모른다면 메서드나 변수의 이름을 바꾸는 데 많은 시간이 소요될 수 있습니다. 수십 개의 클래스에서 사용하는 메서드의 이름을 바꾼다고 상상해 보세요. 이 작업을 간소화하기 위해 Visual Studio의 한 기능이 모든 참조에서 객체의 이름을 변경합니다.

이름을 바꾸려는 이름을 마우스 오른쪽 버튼으로 클릭하고 "이름 바꾸기"를 선택한 다음 새 이름을 입력하고 "적용"을 클릭합니다.

![Remane 1](https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2022/2022-11/rename-1.png?sfvrsn=94bb5e34_3 align="left")

![Remane 2](https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2022/2022-11/rename-2.png?sfvrsn=b2ca8da6_3 align="left")

## `9. 불필요한 using 제거`

불필요한 `using` 인스턴스를 제거하면 코드가 더 적은 줄로 깔끔해지므로 리팩터링에서 항상 좋은 관행이 됩니다.

Visual Studio에는 쉽게 제거할 수 있는 기능이 있습니다. 사용하지 않는 `using` 위치 위에 커서를 놓고 전구 아이콘을 클릭한 다음 "불필요한 사용 제거" 옵션을 선택하기만 하면 됩니다. 열린 창에서 문서, 프로젝트 또는 솔루션 수준에서 제거할 수 있는 옵션이 표시됩니다.

![Remove uneccessary usings](https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2022/2022-11/remove-uneccessary-usings.png?sfvrsn=dcba7841_3 align="left")

## `9. 메서드 자동 생성`

개발자는 시간을 절약해야 하는 경우가 많은데, Visual Studio의 매우 유용한 기능은 메서드를 자동으로 생성할 수 있다는 것입니다. 선언을 작성하고 입력 인수를 전달한 다음 전구 아이콘을 클릭하고 아래 GIF에 표시된 것처럼 '메서드 생성' 옵션을 선택하기만 하면 됩니다:

![Generate Method](https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2022/2022-11/generate-method.gif?sfvrsn=f79a3455_3 align="left")

## 결론

이 글에서 살펴본 바와 같이 리팩터링은 소프트웨어 개발 시대에 항상 존재하는 작업이며, 이 작업을 더 쉽게 수행할 수 있도록 많은 리소스(특히 Visual Studio를 사용하는 경우)가 있습니다.

따라서 다음에 리팩터링을 수행할 때는 이 팁을 검토하고 실행에 옮기는 것을 잊지 마세요.

---

%[https://www.telerik.com/blogs/10-awesome-csharp-refactoring-tips]