## [WinUI 3] 앱 내 도구모음을 감추는 방법

WinUI 3에서는 디버그 시 도구모음이 아래 빨강 영역의 타이틀바에 붙지 않고 본문 영역에 붙습니다.


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1642695265657/lqwTciig9.png)

타이틀바는 win32 구성 요소기 때문일 텐데요, 본문 영역의 내용을 가려 은근히 불편합니다. 도구모음 화살표를 이용해 도구모음을 닫을 수는 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1642695272615/PKAHWiVzJ.png)

그런데 그러면 도구모음이 의미가 없죠.

다행히 WinUI 3는 `라이브 시각적 트리`를 지원하는데요, 다음과 같습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1642695282972/h_6_MMlyi.png)

왼쪽 상단의 빨강색 아이콘 메뉴를 클릭하면 앱 내 도구모음을 보이거나 감출 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1642695292958/UjpFX5_C3.png)

그리고 도구모음 관련 기능을 `라이브 시각적 트리`를 이용해 그대로 사용 할 수 있습니다.

하나 더, 라이브 시각적 트리에서 가장 오른쪽 아이콘 메뉴를 이용해 내 XAML 뿐만 아니라 전체 XAML을 탐색할 수도 있어서 이 기능을 이용하면 유용합니다.