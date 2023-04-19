---
title: "Avalonia UI 및 NXUI로 크로스 플랫폼 시계 앱 작성하기 | Khalid Abuhakmeh"
datePublished: Wed Apr 19 2023 02:26:52 GMT+0000 (Coordinated Universal Time)
cuid: clgn2p16200000ajv6ifiawbo
slug: writing-a-cross-platform-clock-app-with-avalonia-ui-and-nxui
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1681870442464/952b9c1c-3589-4ca3-8801-bc204059612a.webp
tags: net-cikag7ck9004u4153550rzs6c, dotnet

---

> [Khalid Abuhakmeh](https://khalidabuhakmeh.com/)님의 [**Writing a Cross-Platform Clock App With Avalonia UI and NXUI**](https://khalidabuhakmeh.com/writing-a-cross-platform-clock-app-with-avalonia-ui-and-nxui)을 번역하였습니다.

---

![Writing a Cross-Platform Clock App With Avalonia UI and NXUI](https://res.cloudinary.com/abuhakmeh/image/fetch/c_limit,f_auto,q_auto,w_800/https://khalidabuhakmeh.com/assets/images/posts/misc/dotnet-avalonia-ui-nxui-clock-app.jpg align="left")

[Avalonia UI](https://www.avaloniaui.net/)는 일반적인 웹 개발 워크플로에서 벗어난 신선한 충격이었습니다. 물론 웹은 항상 제 첫사랑이겠지만 데스크톱 앱 개발의 쉽고 직관적인 점이 정말 마음에 듭니다. 또한 Avalonia 팀은 진정한 크로스 플랫폼 개발 환경을 만들기 위해 엄청난 노력을 기울였습니다. 따라서 첫 번째 Avalonia 기반 애플리케이션을 구축하려는 경우 이 포스팅이 도움이 될 수 있습니다.

여기서는 [Wiesław Šoltés](https://github.com/wieslawsoltes)가 작성한 NXUI 라이브러리를 사용하겠습니다. 이 라이브러리는 유창한 인터페이스를 사용하여 뷰를 정의하는 동시에 XAML의 표현력을 높여주는 API를 유지하므로 XAML에 알레르기가 있는 경우 데스크톱 개발로 우회할 수 있는 대안이 될 수 있습니다.

이제 시작해보겠습니다!

## NXUI란 무엇인가요?

NXUI는 미니멀리즘의 트렌드를 데스크톱 애플리케이션 개발에 도입하기 위한 시도로, Avalonia UI API를 중심으로 구축되었습니다. NXUI 도움말에서 저자 Wiesław Šoltés는 다음과 같이 말합니다.

> C# 10 및 .NET 6 및 7을 사용하여 최소한의 Avalonia 차세대(NXUI, 차세대 UI) 애플리케이션 만들기

그렇다면 차세대 아발로니아 애플리케이션의 코드는 어떤 모습일까요?

```csharp
Window Build() => Window().Content(Label().Content("NXUI"));

AppBuilder.Configure<Application>()
  .UsePlatformDetect()
  .UseFluentTheme()
  .StartWithClassicDesktopLifetime(Build, args);
```

와우! 비교적 최소한의 기능이지만 좀 더 유용한 것을 작성해 봅시다. 시계는 어때요? 이제 코드를 작성할 시간입니다(말장난입니다).

첫 번째 NXUI 애플리케이션 빌드를 시작하려면 .NET 콘솔 앱의 `.csproj`에 다음을 추가해야 합니다.

```xml
<PackageReference Include="NXUI" Version="11.0.0-preview5" />
```

참고: NuGet에서 검색할 때 버전이 다를 수 있습니다. 최신 버전의 패키지를 사용해야 합니다.

## 아발로니아 UI 시계 앱

NXUI는 `Observable` (관찰가능한) 요소에 크게 의존하여 뷰의 동적 값을 채웁니다. 이 데모의 경우 `System.Timers.Timer` 클래스를 사용하여 현재 `DateTime.Now` 결과를 사용하여 새 문자열 값을 만들 것입니다. 지정된 간격으로 값을 반환하는 `Observable<string>`을 만드는 방법을 살펴보겠습니다.

```csharp
var currentTime = Observable.Create<string>(
    observer =>
    {
        var timer = new System.Timers.Timer {
            Interval = 250,
        };
        timer.Elapsed += (_, _) => observer.OnNext($"{DateTime.Now:hh:mm:ss tt}");
        timer.Start();
        return Disposable.Empty;
    });
```

NXUI로 작업할 때 동적 값에 대해 `Observable.Create` 메서드를 사용하는 경우가 많습니다. 또한 많은 호출을 별도의 클래스와 헬퍼 메서드로 리팩터링할 수도 있습니다.

다음으로 시간 문자열을 담을 뷰를 작성해 보겠습니다.

```csharp
Window Build() =>
    Window()
        .Width(400).Height(200).CanResize(false)
        .WindowStartupLocation(WindowStartupLocation.CenterScreen)
        .Content(
            Border()
                .Margin(25, 0, 25, 0)
                .Height(100)
                .CornerRadius(10)
                .BoxShadow(BoxShadows.Parse("5 5 10 2 Black"))
                .Background(Brushes.White)
                .Child(
                    TextBlock()
                        .Foreground(Brushes.Black)
                        .TextAlignmentCenter()
                        .ZIndex(1)
                        .FontSize(40)
                        .FontStretch(FontStretch.Expanded)
                        .VerticalAlignment(VerticalAlignment.Center)
                        // Set The Observable<string> to Text
                        .Text(currentTime)
                )
        );
```

`TextBlock`을 감싸는 단일 `Border` 요소가 있는 창이 있습니다. 코드에서 가장 중요한 부분은 `TextBlock.Text`를 호출할 때 `currentTime` 인수를 받는다는 점입니다. 타이머가 똑딱거리면 UI가 최신 값으로 업데이트됩니다.

이 데모의 코드는 40줄의 긴 코드입니다.

```csharp
var currentTime = Observable.Create<string>(
    observer =>
    {
        var timer = new System.Timers.Timer {
            Interval = 250,
        };
        timer.Elapsed += (_, _) => observer.OnNext($"{DateTime.Now:hh:mm:ss tt}");
        timer.Start();
        return Disposable.Empty;
    });

Window Build() =>
    Window()
        .Width(400).Height(200).CanResize(false)
        .WindowStartupLocation(WindowStartupLocation.CenterScreen)
        .Content(
            Border()
                .Margin(25, 0, 25, 0)
                .Height(100)
                .CornerRadius(10)
                .BoxShadow(BoxShadows.Parse("5 5 10 2 Black"))
                .Background(Brushes.White)
                .Child(
                    TextBlock()
                        .Foreground(Brushes.Black)
                        .TextAlignmentCenter()
                        .ZIndex(1)
                        .FontSize(40)
                        .FontStretch(FontStretch.Expanded)
                        .VerticalAlignment(VerticalAlignment.Center)
                        // Set The Observable<string> to Text
                        .Text(currentTime)
                )
        );

AppBuilder
    .Configure<Application>()
    .UsePlatformDetect()
    .UseFluentTheme()
    .StartWithClassicDesktopLifetime(Build, args);
```

코드를 실행하면 시계 애플리케이션이 작동하는 것을 볼 수 있습니다. 꽤 멋지네요! (이미지는 최종 샘플의 시계를 반영한 것입니다).

![Avalonia Clock](https://github.com/khalidabuhakmeh/AvaloniaClock/raw/main/screenshot.png align="left")

## 결론

아발로니아의 가장 환상적인 부분은 다양한 방법으로 기술을 이용할 수 있는 다양한 방법을 구축하는 사람들의 풍부한 에코시스템입니다. XAML이 마음에 드신다면 더할 나위 없이 좋습니다. C#을 원하신다면 C#으로도 Avalonia에 액세스할 수 있습니다. F#을 사용하신다면 Avalonia를 F#으로 구현하는 사람들도 있습니다. NXUI는 아발로니아 생태계의 여러 접근 방식 중 하나입니다. 플루언트 인터페이스(Fluent Interface)를 통해 구성 요소의 속성과 구성 요소가 서로 어떻게 연결될 수 있는지 좀 더 쉽게 알아볼 수 있습니다.

이 게시물에 있는 코드를 직접 사용해보고 싶은 분들을 위해, 제 [GitHub 리포지토리에 작동하는 완전한 Avalonia Clock 샘플](https://github.com/khalidabuhakmeh/AvaloniaClock)을 올려놓았습니다.

---

[Writing a Cross-Platform Clock App With Avalonia UI and NXUI | Khalid Abuhakmeh](https://khalidabuhakmeh.com/writing-a-cross-platform-clock-app-with-avalonia-ui-and-nxui)