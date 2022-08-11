## [WinUI 3] 타이틀바를 포함한 테마를 지원하는 WinUI 3 만들기

## 개요
`WinUI 3`은 윈도의 테마와 연계된 테마 시스템을 제공합니다. WinUI 3에서 제공하는 테마 기능과 함께 타이틀바도 테마에 맞게 색을 조정하는 방법을 알아봅시다.

## 개발환경
- Visual Studio 2022 미리보기 4
- .NET 6 RC1
- Windows App SDK 미리보기 2

## WinUI 3에서 테마 동작
WinUI 3에서는 테마의 기본 동작은 `Default`입니다. `Default`일 경우 윈도 테마를 그대로 적용한다는 의미인데요, 가령 윈도 테마 컬러가 `어둡게`일 경우 각종 Element의 스타일에 다크 테마가 적용되는 식입니다.

| 윈도 테마 색이 `밝게`일 경우

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633920669064/5SdRtuy3W.png)

| 윈도 테마 색이 `어둡게`일 경우

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633920680069/BvSmz9W7X.png)

그런데 타이틀바가 테마를 따라가주질 못하네요!

## WinUI 3에서 테마 스타일 지정
윈도10에서는 Light 및 Dark 테마를 지원합니다. 윈도 어플리케이션에서는 이 테마에 맞게 자신의 테마를 조정하기를 원할 텐데요. 다음의 기능을 통해 가능합니다.

| FrameworkElement.cs
```csharp
public ElementTheme ActualTheme { get; }
public ElementTheme RequestedTheme { get; set; }
```

WinUI 3은 `FrameworkElement`를 통해 구현된 컨트롤은 `ActualTheme` 속성을 이용해 현재 선택된 태마를 확인할 수 있습니다.  그리고 `RequestedTheme` 속성에 테마를 설정할 수 있는데요, 설정 가능한 테마는 다음과 같습니다.

| ElementTheme.cs
```csharp
    public enum ElementTheme
    {
        // 기본(윈도 설정 테마)
        Default,
        // 밝게
        Light,
        // 어둡게
        Dark
    }
```

테마 설정은 일반적으로 `Window.Content`의 `RequestedTheme` 속성을 설정하여 Content 이하의 모든 컨트롤에 적용합니다. 지역적으로 적용하려면 지역적으로 적용할 컨테이너의 `RequestedTheme`를 설정하는 것으로 가능합니다.

테마가 변경되었을 때 관련 작업을 하기 위해서 이벤트를 구독할 수 있습니다.

```csharp
rootElement.ActualThemeChanged += (s, e) => { };
```

이 이벤트는 사용자가 테마를 변경하는 것 뿐만 아니라 시스템에서 테마가 변경된 것을 감지할 수 있습니다.

## WinUI 3 타이틀바 테마 적용

위에서 확인한 것처럼 윈도 색 테마가 `어둡게`로 되어 있어도 어두운 테마가 타이틀바 까지는 적용되지 않는데요, 타이틀바는 `WinUI 3`의 구성이 아니기 때문입니다. 이를 개선하기 위해서는 `ActualTheme` 속성과 `ActualThemeChanged ` 이벤트를 통해 현재 설정된 테마의 종류와 윈도 테마 색이 변경되었을 때 감지해서 타이틀 바의 색을 변경해야 합니다.

타이틀바를 변경하기 위해선 `AppWindow`의 도움을 받아야 하는데요, `AppWindow`는 다음처럼 획득할 수 있습니다.

```csharp
    public AppWindow AppWindow
    {
        get
        {
            var hWnd = WinRT.Interop.WindowNative.GetWindowHandle(this);
            var winId = Win32Interop.GetWindowIdFromWindow(hWnd);
            return AppWindow.GetFromWindowId(winId);
        }
    }
```

그리고 타이틀바의 각종 색은 다음처럼 변경할 수 있습니다.

```csharp
private void ModifyTitlebarTheme()
    {
        var content = (Content as FrameworkElement)!;
        var value = content.ActualTheme;

        var titleBar = AppWindow.TitleBar;
        if (value == ElementTheme.Light)
        {
            titleBar.ForegroundColor = Colors.Black;
            titleBar.BackgroundColor = Colors.White;
            titleBar.InactiveForegroundColor = Colors.Gray;
            titleBar.InactiveBackgroundColor = Colors.White;

            titleBar.ButtonForegroundColor = Colors.Black;
            titleBar.ButtonBackgroundColor = Colors.White;
            titleBar.ButtonInactiveForegroundColor = Colors.Gray;
            titleBar.ButtonInactiveBackgroundColor = Colors.White;

            titleBar.ButtonHoverForegroundColor = Colors.Black;
            titleBar.ButtonHoverBackgroundColor = Color.FromArgb(255, 245, 245, 245);
            titleBar.ButtonPressedForegroundColor = Colors.Black;
            titleBar.ButtonPressedBackgroundColor = Colors.White;
        }
        else if (value == ElementTheme.Dark)
        {
            titleBar.ForegroundColor = Colors.White;
            titleBar.BackgroundColor = Color.FromArgb(255, 31, 31, 31);
            titleBar.InactiveForegroundColor = Colors.Gray;
            titleBar.InactiveBackgroundColor = Color.FromArgb(255, 31, 31, 31);

            titleBar.ButtonForegroundColor = Colors.White;
            titleBar.ButtonBackgroundColor = Color.FromArgb(255, 31, 31, 31);
            titleBar.ButtonInactiveForegroundColor = Colors.Gray;
            titleBar.ButtonInactiveBackgroundColor = Color.FromArgb(255, 31, 31, 31);

            titleBar.ButtonHoverForegroundColor = Colors.White;
            titleBar.ButtonHoverBackgroundColor = Color.FromArgb(255, 51, 51, 51);
            titleBar.ButtonPressedForegroundColor = Colors.White;
            titleBar.ButtonPressedBackgroundColor = Colors.Gray;
        }

        // 아이콘 배경 색이 적용 안되는 버그 수정
        titleBar.IconShowOptions = IconShowOptions.HideIconAndSystemMenu;
        titleBar.IconShowOptions = IconShowOptions.ShowIconAndSystemMenu;
    }
```

그리고 윈도의 테마 색이 변경될 때를 감지하기 위해 이벤트 구독을 합니다.

```csharp
    public MainWindow()
    {
        this.InitializeComponent();

        var content = (Content as FrameworkElement)!;
        content.ActualThemeChanged += (s, e) =>
        {
            ModifyTitlebarTheme();
        };
        ModifyTitlebarTheme();
    }
```

## 동작 화면

| 윈도 테마 색이 `밝게`일 경우

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633920694336/NIVgPTjwuH.png)

| 윈도 테마 색이 `어둡게`일 경우

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633920702883/wR14TI2jd.png)

## 정리
`WinUI 3`에서 타이틀바에 테마를 적용하는 방법을 알아보았습니다.

`WinUI 3`이 어느덧 `버젼 1.0 미리보기 2` 단계입니다. 그 사이 UWP 미지원 및 UWP와 WPF에 비해서 성
능이 현저히 떨어지는 등의 이슈가 있었지만 조금씩 안정화 되어가고 있는 모습입니다.

우리가 `Windows`라는 운영체제를 사용하는 이상 여전히 윈도 어플리케이션 개발을 위해 새로운 환경에 익숙해질 필요가 있습니다. 물론 여전히 윈폼이나 WPF로도 거뜬히 윈도 어플리케이션을 만들 수는 있지만 새로운 윈도의 지원을 활용하기엔 제약이 따르죠. `WPF`개발자 이면서 `UWP`의 폐쇄적인 여러 정책 때문에 UWP에 접근하지 못했던 분들은 이번에 `WinUI 3`를 체험해 볼 만합니다. `WinUI 3`도 XAML 기반이므로 WPF에서 사용하고 있는 여럿 경험을 공유하거든요. 자! 자신의 앱을 `WinUI 3`으로 개발 해봅시다!