---
title: "Blazor를 사용하여 상황에 맞는 경험 구축 (번역)"
datePublished: Sun Jun 13 2021 04:43:17 GMT+0000 (Coordinated Universal Time)
cuid: ckpupcn5z04kh0ms19vht540e
slug: building-contextual-experiences-wblazor
tags: dotnet, blazor-1

---

안녕! 내 이름은 Hassan Habib이고 Microsoft의 선임 엔지니어링 매니저 입니다. 이것은 ASP.NET 팀 블로그의 내 첫번째 블로그 입니다. [내 OData 게시물](https://devblogs.microsoft.com/odata/author/harezk/)에서 나를 알 수 있습니다. 몇 주 전에 [Daniel Roth](https://devblogs.microsoft.com/aspnet/author/danroth27/)에게 연락하여 Microsoft 엔지니어가 Microsoft 제품을 사용하여 자체 시스템을 구축하는 방법을 공유하는 것이 좋을지 궁금해 했습니다. 이것은 우리가 "Microsoft에서 Microsoft를 실행"이라고 부르는 작은 것입니다. Daniel은 적극적으로 지원했고 우리는 이를 가능하게 하기 위해 함께 노력했으며 그 점에 대해 매우 감사합니다.

End-to-End 엔터프라이즈 솔루션을 개발하기 위해 Microsoft 내부 팀과 계속 협력하면서 Microsoft 기술에 대한 나의 경험은 계속해서 다양한 방향으로 발전하고 있습니다. 이러한 경험은 실제 End-to-End 제품을 구축 할 때 Microsoft 기술을 관점으로 가져 오는 것이 어떤 것인지 알아 보는 모든 엔지니어에게 유용할 것이라고 생각 했습니다. 내 블로그 게시물에서는 Microsoft 엔지니어가 Microsoft 기술을 활용하여 가장 복잡한 문제를 해결하고 세계 최고의 경험을 이끌어내는 방법에 대한 관점을 제공하려고 합니다.

내부 Microsoft 프로젝트 외에도 [오픈 소스 커뮤니티](https://github.com/hassanhabib)에 많은 기여를 하고 있습니다. 블로그 게시물에서 소개하려는 대부분의 예제와 개념에 대해 샘플 코드가 포함되어 있는지 혹은 오픈 소스 프로젝트의 참조가 추가되어 원시 저장 데이터에서 모바일, 웹 및 데스크톱 응용 프로그램에 이르기까지 전체 그림을 볼 수 있도록 합니다. 그래서 더이상 고민하지 않고 여기에 첫 번째 게시물이 있습니다! 나는당신이 그것을 즐기기를 바랍니다!

몇 달 전에 ASP.NET 팀은 Hot Reload, WebAssembly 사전 컴파일 및 엔지니어링 및 웹 개발 경험을 엄청나게 향상시키는 기타 여러 기능을 포함하여 Blazor의 가장 인기있는 기능 몇 가지를 [발표](https://devblogs.microsoft.com/aspnet/asp-net-core-updates-in-net-6-preview-1/) 했습니다.

그러나 도입 된 가장 중요한 기능 중 하나는 DynamicComponent 기능이었습니다. Blazor 구성 요소의 클래스 유형을 전달하고 새로운 기본 제공 Blazor 기능이 해당 구성 요소 렌더링을 처리하도록 하여 Blazor 구성 요소를 동적으로 유창하게 렌더링 할 수 있는 .NET 6의 새로운 기능입니다.

이 새로운 기능을 통해 웹 개발자는 상황 별 디자인 패턴에 쉽게 적응할 수 있습니다. 특정 사용자가 필요로 하는 것과 정확히 일치하고 동일한 웹 어플리케이션을 사용하는 다른 사용자와 다를 가능성이 더 높은 개인화 된 사용자 경험 측면에서 웹 어플리케이션이 컨텍스트를 인식하도록 만드는 패턴입니다.

상황 별 경험의 개념은 프런트엔드에 데이터를 제공하는 백엔드 서비스가 제공한 데이터를 기반으로 사용자 인터페이스가 어떤 종류의 기능을 가질 것인지를 제어 할 수 있는 서비스인 데이터 기반 설계와 밀접한 관련이 있습니다. 이 패턴은 특정 사용자 프로필 또는 기본 설정에 따라 조정되는 동적 응용 프로그램을 구축 할 때 매우 효과적인 것으로 입증 되었습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623500637172/Pd80nfra9.png)

예를 들어 접근성이 필요한 사용자는 다른 사용자와 다른 특정 작업을 수행하기 위해 다른 구성 요소 집합이 필요할 수 있습니다. 마찬가지로 특정 기본 설정을 가진 사용자는 대쉬보드에서 특정 구성 요소를 추가하거나 제거하여 필요에 맞게 사용자 경험을 조정할 수 있습니다. 그러나 보다 일반적으로 동적 데이터 기반 UI 구성 요소는 사용자로부터 수집하려는 데이터 유형을 조정해야 하는 엔터프라이즈 응용 프로그램에서 매우 유용할 수 있습니다.

이에 대한 좋은 예는 특정 주제에 대한 정보를 요청하는 양식을 제공하는 경우입니다. 사용자의 답변 선택에 따라 새로운 질문 세트가 나타날 수 있습니다. 이 경우 상황 별 경험을 통해 각 사용자가 자신의 모험을 선택할 수 있습니다. 이 패턴은 대부분의 설문지, 요청 양식 및 현재 사용자 선택에 따라 조정되는 기타 모든 유형의 응용 프로그램에서 볼 수 있습니다.

다음 몇 섹션에서는 Blazor의 DynamicComponent기능을 사용하여 상황 별 사용자 경험을 설계, 디자인 및 개발하는 전체 프로세스를 안내할 것입니다.

## 높은 수준의 디자인
컨텍스트 인식 Blazor 어플리케이션을 구현하려면 최상의 환경을 제공하기 위해 API/UI 구성 요소를 설계하는 방법을 이해해야 합니다. 다음은 컨텍스트 인식 시스템 설계에 대한 몇가지 규칙입니다.

### 순수 데이터 (API 측)
API 시스템에 대한 모델을 디자인 할 때 API 측 모델에는 용어 또는 스타일 관점에서 UI 경험과 관련된 것이 없어야 합니다. 예를 들어 벡엔드 시스템에 있는 데이터에는 TextBox 또는 DropDronList을 가지지 않아야 합니다. 그것은 데이터 제공이 아닌 렌더링의 경계를 넘어선다. 그러나 API 데이터는 제공하는 정보의 원시 데이터 유형에 대해 알아볼 수 있습니다. 예를 들어, 원시 데이터 유형은 렌더링해야 하는 데이터 유형을 설명하는 역할을 할 수 있습니다. 그러나 제공된 구성 요소에 따라 여러 줄 텍스트와 한 줄 텍스트가 다르게 렌더링 될 수 있으므로 이는 항상 그런 것은 아닙니다. 이 경우 UI 클라이언트가 이를 매핑하고 궁극적으로 데이터를 동적으로 렌더링하는 방법을 알 수 있도록 사용자 지정, 원시 및 일반 형식을 배치해야 합니다.

### 매핑 (UI 측)
UI 측에서 Blazor 어플리케이션에는 원시 데이터 형식을 지정된 UI 구성 요소에 매핑하는 방법을 아는 매핑 구성 요소 또는 서비스가 포함될 수 있습니다. 예를 들어 Blazor 어플리케이션은 Text 형식의 원시 데이터를 수신하고 해당 형식을 TextBoxComponent UI 구성 요소에 매핑합니다. 이렇게 하면 데이터 형식의 일반적인 용어를 활용하여 현재 UI 클라이언트를 렌더링하려는 UI 구성 요소와 일치하도록 추상화 수준을 만들었습니다.

이 패턴을 사용하면 UI 어플리케이션 유형에 따라 다른 방식으로 동일한 원시 데이터를 렌더링 할 수도 있습니다. 예를 들어 모바일 앱용으로 빌드 된 UI 구성 요소는 Xamarin 앱의 경우 Text를 Entry로 렌더링 하도록 선택할 수 있습니다. 그리고 데스크톱 Windows Forms 응용 프로그램에서는 Text 데이터 형식을 TextBox로 렌더링하도록 선택할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623558075697/Xxpk_RkMI.png)

## 구현
앞서 언급한 이론을 현실로 바꿔봅시다. 다음 데이터 본문을 제공하는 API가 있다고 가정해 보겠습니다.

```json
{
    "options":
    [
       "Choice",
       "Choices",
       "Text"
    ]
}
```

Blazor 클라이언트에서 API 브로커는 위 모델을 다음을 수행하는 서비스에 넘깁니다.

```csharp
public IEnumerable<OptionView> RetrieveAllOptionViews()
{
    List<string> options = this.optionService.RetrieveAllOptions();

    return options.Select(option => new OptionView
    {
        Text = option,
        Value = $"{option}Base"
    });
}
```

위의 함수는 원시 API 형식의 모든 옵션을 검색하는 기본 종속성이라고 합니다. 그런 다음 드롭다운 목록의 옵션과 값을 UI구성 요소의 이름으로 각 결과와 모든 결과를 텍스트 값으로 매핑합니다.

예를 들어, 해당 목록의 값 중 하나가 Choice 드롭다운 옵션인 경우 텍스트는 Choice이고 값은 텍스트 값에 해당하는 UI 구성 요소의 ChoiceBase 문자열이 됩니다.

이제 다음과 같이 모든 것이 함께 묶이는 페이지 수준으로 살펴보겠습니다.

```csharp
public partial class Index : ComponentBase
{
    [Inject]
    public IOptionViewService OptionViewService { get; set; }

    public IEnumerable<OptionView> OptionViews { get; set; }
    public Type SelectedComponent { get; set; }

    protected override void OnInitialized()
    {
        SelectedComponent = typeof(ChoiceBase);
        this.OptionViews = this.OptionViewService.RetrieveAllOptionViews();
    }

    public void SetComponent(ChangeEventArgs changeEventArgs)
    {
        string fullComponentName = changeEventArgs.Value;
        SelectedComponent = Type.GetType(typeName: fullComponentName);
    }
}```

페이지의 백엔드 쪽의 OptionsViews 속성은 OptionViewService으로부터 사용할  준비가 된 모든 목록입니다. 또한 기본 선택항목의 속성인 SelectComponent 특정 속성은 DynamicComponent에서 렌더링할 수 있도록 일부 값으로 초기화 하는 것이 중요합니다.

이제 선택에 따라 SetComponent 함수는 값을 재설정하고 다음의 SelectedComponent으로 다시 렌더링 하도록 DynamicComponent를 트리거 합니다.

```csharp
<select @onchange="SetComponent">
    <Iterations Items="OptionViews" T="OptionView" Context="optionView">
        <option value="@optionView.Value">@optionView.Text</option>
    </Iterations>
</select>

<DynamicComponent Type="SelectedComponent" />
```

모든 OptionsView 옵션이 만들어지고 선택이 변경되면 드롭 다운 목록 옵션에 해당하는 새값을 트리거하여 그에 따라 SelectedComponent도 변경됩니다.

다음은 위 코드에서 이벤트의 이벤트 흐름을 시각화하는 시퀀스 다이어그램입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623558914263/vv7b-8w5c.png)

이제 사용자의 선택에 따라 DynamicComponent 렌더링 구성 요소를 처리하는 방법을 살펴보겠습니다.

![DynamicComponentDemo-1.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1623558948754/4cN7SW-fc.gif)

이제 위의 예는 원래 패턴의 작은 표현일 뿐입니다. 진정으로 상황에 맞는 경험을 제공하는 것은 양방향입니다. 동적 구성 요소만 제공하는 것은 읽기 전용 모드에서만 유익할 수 있습니다. 그러나 선택항목을 저장된 값으로 API에 다시 보내야 하는 경우 어떻게 합니까? 이러한 저장된 값을 사전 설정된 매개 변수로 동적 구성 요소에 다시 렌더링 하는 방법은 무엇입니까? DynamicComponent는 중첩된 구성 요소를 지원할 수 있습니까?

이 모든 질문과 그 이상은 이 기사의 다음 부분에서 다룰 것이므로 계속 지켜봐주십시요!

다음은 몇가지 최종 참고 사항입니다.
- 위 데모의 소스코드는 [여기](https://github.com/hassanhabib/ContextualExperience)에서 찾을 수 있습니다.
- 다음은 동일한 패턴에 대한 전체 End-to-End 데모 [비디오](https://www.youtube.com/watch?v=Wcc14aoylME) 입니다.

## 원문
- Microsoft ASP.NET Blog | [Building Contextual Experiences w/Blazor | Hassan](https://devblogs.microsoft.com/aspnet/building-contextual-experiences-w-blazor/)
