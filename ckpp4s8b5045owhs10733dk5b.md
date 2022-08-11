## .net 생산성의 새로운 내용 알아보기 (번역)

.NET 생산성 팀(일명 Roslyn)은 Visual Studio 2019의 최신 도구 업데이트를 통해 개발자의 생산성을 지속적으로 향상시키고 있습니다. 지난 릴리즈에서는 귀하의 피드백을 듣고 .NET 개발자 환경을 개선하기 위해 열심히 노력했습니다. 최신 .NET 생산성 향상 기능을 사용해 보려면 [최신 Visual Studio 릴리즈를 다운로드](https://visualstudio.microsoft.com/vs/)하십시오.

## 툴링 개선 사항

내가 가장 흥분하는 기능은 [상속 여백(Inheritance margin)](https://docs.microsoft.com/visualstudio/ide/reference/options-text-editor-csharp-advanced?view=vs-2019#inheritance-margin) 입니다. 상속 여백은 상속 체인을 탐색하고 검사하기 위한 시각적 표현을 제공합니다. 상속 여백은 기본적으로 해제되어 있으므로 **도구 > 옵션 > 텍스트 편집기 > C# 또는 기본 > 고급**에서 선택해야 합니다. 상속 여백을 사용하도록 설정하면 코드의 구현 및 재정의를 나타내는 여백에 새 아이콘이 추가됩니다. 상속 여백 아이콘을 클릭하면 탐색할 수 있는 상속 옵션이 표시됩니다.

![Untitled-Project.gif](https://devblogs.microsoft.com/visualstudio/wp-content/uploads/sites/4/2022/06/Untitled-Project.gif)

또다른 흥미로운 기능은 새로운 [사용하지 않는 참조 제거 명령](https://docs.microsoft.com/visualstudio/ide/reference/remove-unused-references?view=vs-2019)입니다. 이 명령을 사용하면 사용이 없는 프로젝트 참조 및 NuGet 패키지를 정리할 수 있습니다. 사용량이 없는 프로젝트 참조를 제거하면 공간을 절약하고, 응용 프로그램의 시작 시간을 줄이며, NuGet을 더 빠르게 복원할 수 있습니다. **사용하지 않는 참조  제거** 명령은 솔루션 탐색기의 프로젝트 이름 또는 종속성 노드의 마우스 오른쪽 클릭 메뉴에 나타납니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623218656464/GDkU-VUJp.png)
**사용하지 않는 참조 제거**를 선택하면 제거될 모든 참조를 볼 수 있는 대화 상자가 열리지만 유지하려는 참조를 보존 할 수도 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623218720568/KAEjUMzpb.png)

[EditConfig](https://docs.microsoft.com/visualstudio/ide/create-portable-custom-editor-options?view=vs-2019) 파일은 코드베이스에서 일관된 코드 스타일과 코드 품질 기본 설정을 정의하는 좋은 방법입니다. 구성 프로세스가 쉽지 않다는 피드백을 받았기 때문에 16.10 Preview 2에서 EditConfig 디자이너를 추가하여 코드 분석 기본 설정을 쉽게 보고 구성할 수 있게 하였습니다. 새 디자이너를 사용하려면 솔루션에서 C# 또는 시각적 기본 편집기 Config 파일을 엽니다. EditConfig 디자이너는 서식, 코드 스타일, 타사 분석기(예: StyleCop 또는 xUnit이 설치된 경우) 및 .NET 5.0 SDK에 포함된 [.NET 코드 품질 분석기](https://docs.microsoft.com/dotnet/fundamentals/code-analysis/overview)를 표시합니다. **F7**을 눌러 편집기 텍스트 보기로 다시 전환할 수 있는 옵션도 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623218881852/UFCpUO2-K.png)

**스마트 브레이크 라인(Smart Break Line)**이라는 새로운 명령이 있습니다. 이 명령은 자동으로 중괄호 세트를 삽입하고 **Shift + Enter**키를 커밋 문자로 사용할 때 해당 중괄호 안에 캐럿을 배치합니다. **스마트 브레이크 라인**은 속성, 이벤트, 필드 및 객체 생성 표현식 뿐만 아니라 중괄호가 필요한 모든 유형 선언에 대해 작동합니다.

![Animation1.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1623219793531/G7v2VMH4F.gif)

[지시문 사용](https://docs.microsoft.com/en-us/visualstudio/ide/visual-csharp-intellisense?view=vs-2019#add-missing-using-directives-on-paste)은 이제 유형을 복사하여 새 파일에 붙여 넣을 때 자동으로 추가됩니다. 먼저 **도구 > 옵션 > 텍스트 편집기 > C# 또는 기본 > 고급**에서 이 옵션을 켜고 **붙여넣을 때 누락된 using 지시문 추가**를 선택해야 합니다.

![Animation2.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1623220175861/ZbBzhxIIA.gif)

[이동(Go To)](https://docs.microsoft.com/visualstudio/ide/go-to?view=vs-2019)은 더 이상 .NET Core 2.0 및 3.1 앱에서 중복 결과를 표시하지 않으며 다른 중첩 유형을 래핑하기 위해 존재하는 부분 유형에 대한 결과도 표시하지 않습니다. 이렇게 하면 결과를 정리하는데 도움이되므로 코드를 쉽게 찾고 탐색 할 수 있습니다. 결과에는 이제 부분 기호의 파일 이름도 포함됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623220370460/RZ-ksb-q2.png)

EnC([Edit and Continue](https://docs.microsoft.com/visualstudio/debugger/edit-and-continue?view=vs-2019))를 사용하면 전체 프로그램을 중지하고 다시 컴파일 한 다음 디버깅 세션을 다시 시작하지 않고 중단 모드에 있는 동안 디버깅 세션 중에 소스 코드를 변경할 수 있습니다. **편집하고 계속**하기에는 몇 가지 제한 사항이 있으므로 모든 편집이 지원되는 것은 아니지만, 16.10에서는 부분 클래스, 레코드 편집에 대한 지원을 추가하고 지시문을 사용하여 누락된 항목을 추가했습니다. EnC는 또한 [.NET Hot Reload 소개 블로그](https://devblogs.microsoft.com/dotnet/introducing-net-hot-reload/)에서 자세히 알아볼 수 있는 [Hot Reload](https://devblogs.microsoft.com/dotnet/introducing-net-hot-reload/)를 지원합니다.

## IntelliSense 완성
IntelliSense는 오타 및 기타 일반적인 실수를 줄여 애플리케이션 코딩 프로세스의 속도를 높이는 상황 인식 코드 완성 기능입니다.
16.10에서는 메서드 호출을 작성할 때 인수를 자동으로 삽입하는 오나료 옵션을 추가했습니다. 이 기능은 기본적으로 꺼져 있으므로 **도구 > 옵션 > 텍스트 편집기 > C# > IntelliSense**에서 **두 번 탭하여 인수 삽입(실험적)**을 활성화 해야 합니다. 이 기능을 사용하려면 메서드 호출을 작성하고 탭을 두 번 누릅니다.

![Animation3.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1623220935043/QIDPihAyL.gif)

메소드 호출에는 메소드의 기본값을 기반으로 하는 인수가 포함됩니다. 매게 변수 정보를 사용하여 위쪽 및 아래쪽 화살표 키를 눌러 삽입 할 인수 목록을 순환합니다. 세미콜론을 입력하면 인수를 커밋하고 메서드 호출 끝에 세미콜론을 추가합니다.
16.9에서는 Enum 이름을 입력하지 않은 경우에도 형식이 알려지면 Enum 값에 IntelliSense 완료를 추가했습니다. 우리는 캐스트, 인덱서 및 연산자에 대한 IntelliSense 완성 기능을 추가하여 이 기능을 한 단계 더 발전 시키기로 결정하였습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623221063478/nf7uscOFN.png)

또한 최근에는 전 처리기 기호에 대한 IntelliSense 완성 기능을 추가했습니다. #if 지시문 입력을 시작하고 현재 범위에 정의된 기호에 대한 새로운 완성 옵션을 확인합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623221139612/LnQUrW5QQ.png)

## 코드 수정 및 리팩토링
코드 수정 및 리팩토링은 컴파일러가 전구 및 드라이버 아이콘을 통해 제공하는 코드 제안입니다. **빠른 작업 및 리팩토링** 메뉴를 트리거하려면 (**Ctrl + .**) 또는 (**Alt _ Enter**)를 누릅니다. 놓친 경우를 대비하여 흥미로운 새 코드 수정 및 리팩토링을 추가했습니다!

[단순화 LINQ식](https://docs.microsoft.com/en-us/visualstudio/ide/reference/simplify-linq-expression?view=vs-2019)의 리팩토링은 성능과 가독성을 개선하는데 도움이 되도록 .Where() 메소드에 대한 Enumerable의 불필요한 호출을 제거합니다. LINQ식에 커서를 놓고 (**Ctrl + .**)를 눌러 **빠른 작업 및 리팩토링** 메뉴를 트리거 합니다. 이후 **LINQ 식 단순화**를 선택 합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623221522600/IXuRxB_RK.png)

새로운 ** 코드 수정을 사용**하여 문서에서 **추가 공백을 제거**합니다. 추가 빈 줄에 커서를 놓습니다. (**Ctrl + .**)를 눌러 **빠른 작업 및 리팩토링** 메뉴를 트리거 한 후 **불필요한 삭제 제거**를 선택 합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623222226139/G1fomIRhv.png)

C# 9.0에서는 특정 [패턴 일치](https://docs.microsoft.com/dotnet/csharp/pattern-matching) 사례에서 삭제가 필요하지 않습니다. 이제 불필요한 폐기를 제거하고 제거할 코드 수정을 제공합니다. 희미해진 폐기에 커서를 놓습니다. (**Ctrl + .**)를 눌러 **** 메뉴를 트리거 한 후 ****를 선택합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623222297944/l_JlDNiW7.png)

## Blazor 도구
16.7 Preview 4에서는 MVC, Razor 페이지 및 Blazor를 사용한 로컬 개발을 위해 새로운 Razor 편집기를 추가했습니다. 새로운 Razor 편집기에는 기존 편집기의 기능을 넘어서는 수많은 새로운 도구 지원이 있습니다. 예를 들어, 우리는 많은 C# 코드 수정 및 리팩토링 (예: 이름 바꾸기 및 누락된 using 지시문 추가)을 추가했습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1623222321042/SjZFbuUrt.png)

기타 개선 사항으로는 닫힌 Razor파일에서 모든 참조 찾기, Blazor 구성 요소 태그 이름에 대한 정의로 이동, Razor 형식 지정, 더 나은 진단, Razor 고유 완성, 대규모 프로젝트 및 솔루션에 대한 편집 성능 및 안정성이 포함됩니다. 새로운 Razor 편집기를 사용해보고 여러분의 생각을 알려주세요! 새로운 Blazor 편집기를 사용하려면, ****로 이동하여 ****을 선택한 후 비쥬얼 스튜디오를 다시 시작합니다. ASP.NET Core 레파지토리에서 [GitHub에서 Razor도구 문제](https://github.com/dotnet/aspnetcore/issues/new?template=razor_tooling.md)를 만들어 피드백을 공유할 수 있습니다.

## 참여하기
이것은 [비쥬얼 스튜디오 2019](https://visualstudio.microsoft.com/downloads/)의 새로운 기능을 살짝 엿본것일 뿐입니다. 새로운 기능의 전체 목록은 [릴리즈 노트](https://docs.microsoft.com/visualstudio/releases/2019/release-notes)를 참조하세요. [개발자 커뮤니티](https://developercommunity.visualstudio.com/spaces/8/index.html) 웹 사이트에 피드백을 제공하거나 Visual Studio에서 [문제보고](https://docs.microsoft.com/visualstudio/ide/how-to-report-a-problem-with-visual-studio) 도구를 사용하여 자유롭게 피드백을 제공할 수 있습니다.

## 문서 원문
> Microsoft Visual Studio Blog | [Learn What's New in .NET Productivity | Mika](https://devblogs.microsoft.com/visualstudio/learn-whats-new-in-net-productivity/)