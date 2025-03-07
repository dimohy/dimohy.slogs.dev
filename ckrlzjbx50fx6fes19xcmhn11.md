---
title: "WinUI 3 소개"
datePublished: Tue Jul 27 2021 11:38:03 GMT+0000 (Coordinated Universal Time)
cuid: ckrlzjbx50fx6fes19xcmhn11
slug: winui-3
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1627457412836/0LAaZDj4m.png
tags: desktop, windows, dotnet, winui3

---

## 목차

1. [개요](#6rcc7jqu)
1. [WinUI 2](#winui-2)
1. [WinUI 3](#winui-3)
1. [WinUI 3과 WinUI 2 비교](#winui-3-winui-2)
1. [Windows 앱 SDK](#windows-sdk)
1. [체험](#7lk07zey)
1. [환경구성](#7zmy6rk96rws7isx)
1. [Hello World 프로젝트](#hello-world)
1. [로드맵](#66gc65oc66e1)
1. [정리](#7kcv66as)

## 개요
[WinUI](https://docs.microsoft.com/ko-kr/windows/apps/winui/) 라이브러리는 Windows 데스크톱 및 UWP 애플리케이션에서 사용할 수 있는 네이티브 UX(사용자 환경) 프레임워크 입니다. WinUI는 Microsoft의 [Fluent Design System](https://www.microsoft.com/design/fluent/#/)을 UI(사용자 인터페이스)에 사용하여 최신의 Windows 10 및 Windows 11의 UI 환경을 제공합니다.

## WinUI 2
WinUI 2는 UWP 애플리케이션에서 사용할 수 있으며 [XAML Islands](https://docs.microsoft.com/ko-kr/windows/apps/desktop/modernize/xaml-islands)를 이용해 신규 또는 기존 데스크톱 애플리케이션과 통합할 수 있습니다. WinUI 2 라이브러리는 [Windows 10 SDK](https://developer.microsoft.com/ko-kr/windows/downloads/windows-10-sdk/)와 밀접하게 연결되어 있으며, UWP 앱용 공식 네이티브 Windows UI 컨트롤 및 기타 UI 요소를 제공합니다.

Windows 10 이전 버전과의 하위 호환성을 유지하므로 사용자가 최신 OS가 아니라도 WinUI 2 컨트롤이 작동합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627261818979/8MzuRsTbB.png)
출처: docs.microsoft.com

## WinUI 3
WinUI 3는 Windows 10 SDK에서 완전히 분리된 네이티브 Windows 10 UI 플랫폼입니다. WinUI 3은 광범위한 Windows 10 OS 버전에서 모든 데스크톱 앱이 일관된 방식으로 동작하도록 하는 통합 API 및 도구 세트를 제공합니다. Windows 앱 SDK(구 Project Reunion)의 구성 요소이며, [최신 Windows 앱 SDK 정보](https://github.com/microsoft/WindowsAppSDK)를 확인하고 [Windows 앱 SDK를 설치](https://marketplace.visualstudio.com/items?itemName=ProjectReunion.MicrosoftProjectReunion)해서 사용할 수 있습니다.

WinUI 3는 모든 Windows 앱에서 사용할 수 있습니다. 네이티브 UWP 또는 Win32 앱에서 UI 계층으로 사용하거나, `XAML Islands`를 사용하여 데스크톱 앱을 점진적으로 현대화 할 수 있습니다.

WinUI 3은 WinUI 2와 다르게 모든 XAML 기능을 WinUI의 일부로 제공합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627262380821/icwsnOv7x.png)
출처: docs.microsoft.com

## WinUI 3과 WinUI 2 비교
WinUI 2와 WinUI 3 모두 `Fluent Design 시스템` 원칙 및 프로세스를 기반으로 사용자 인터페이스 및 환경을 제공하지만 WinUI 2는 UWP 어플리케이션만을 기본적으로 지원하며 `Windows 10 SDK`에 종속되어 있는 라이브러리인데 반해 WinUI 3는 UWP 어플리케이션 및 Win32 앱의 지원을 목표로 하고 있으며 `Windows 10 SDK`와 분리되어 지원 OS에 원활하게 대응할 수 있도록 설계되었습니다.

### 주요 차이점

|구분|WinUI 3|WinUI 2|
|-|-|-|
|Windows 10 SDK 종속성|UI 스택 및 컨트롤 라이브러리가 Windows 10 SDK과 완전히 분리|UX 스택 및 컨트롤 라이브러리가 Windows 10 SDK와 긴밀하게 결합|
|Windows 데스크톱/Win32 앱 빌드|빌드할 수 있음|빌드할 수 없음|
|패키지 제공|Windows 앱 SDK를  Visual Studio 확장(VSIX) 형태로 설치하여 제공|일부는 운영체제 자체에서 제공, 일부는 추가 라이브러리 제공, 핵심 XAML 프레임워크 및 입력, 컴포지션 레이어 등의 UI 스택의 중요 부분은 OS에서 기본 제공|
|.NET 5 지원유무|데스크톱 앱용 C# 및 .NET 5 지원|C# 및 .NET 네이티브 앱만 지원|
|WebView2 지원유무|지원|미지원|
|최소 OS|Windows 10 버젼 1809|Windows 10 1703|

## Windows 앱 SDK
WinUI 3는 Windows 앱 SDK의 구성요소입니다. WinUI 3을 이용하려면 반드시 Windows 앱 SDK를 이용해야 하는데요, 간략히 Windows 앱 SDK에 대해서 알아봅시다.

[Windows 앱 SDK](https://github.com/microsoft/WindowsAppSDK)는 다양한 Windows 버젼의 플랫폼 기능에 접근하기 위해 `Windows 10 SDK`에 종속되지 않고 사용할 수 있도록 구성된 라이브러리, 프레임워크, 구성 요소 및 도구 집합입니다. 이런 구성을 통해 Windows 앱 SDK는 최신 API 사용 기술과 함께 Win32 애플리케이션의 기능을 결합하므로 최소사양을 준수하는 한 사용자의 모든곳에서 앱이 동작하는 것을 보장합니다.
Windows 앱 SDK의 기능은 새로운 API,  수렴된 API 및 API 하위 집합의 세 가지 주요 범주로 제공됩니다.

### 새로운 API
새로운 API의 경우 가능하면 Windows 앱 SDK 제품군의 일부로 새로운 windows 기능이 제공됩니다. 이 새로운 API는 애플리케이션에 투명합니다. 새로운 기능은 배포 유형과 상관없이 사용할 수 있는 공통 인터페이스를 공유합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627281708856/LFePFDTkX.png)

### 수렴된 API
Windows 앱 SDK는 이미 플랫폼의 Win32와 UWP/AppContainer 기능 간의 격차를 해소하는 API 표면을 제공합니다. 이를 통해 더 많은 Windows 버전에서 앱을 실행할 수 있게 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627281836378/kUX9_LCr8.png)

### 하위 집합 API 제품군
Windows Kit API 파티션과 마찬가지로 Windows 앱 SDK는 모든 버전의 Windows에서 작동하는 Windows 플랫폼 API의 하위 집합을 정의합니다. 코드가 이 하위 집합을 대상으로 하고 Windows 앱 SDK 새로운 기능과 수렴 기능을 사용하는 경우 추가 작업 없이 Windows가 작동하는 모든 곳에서 동작을 보장합니다.

하위 집합 API 제품군은 다음과 같습니다.

- 윈도윙, 입력, 메시징, GDI 및 GUI 하위 시스템 기능
- 파일 시스템 및 스토리지 액세스
- 네트워킹
- 인쇄
- 프로세스, 스레딩, 메모리 관리, 기본 애플리케이션 서비스
- DirectX, D3D, DirectML

Microsoft는 `Windows 앱 SDK`를 이용해서 앞으로 제공하는 API를 정제하고 통일하고 추가해나갈 것으로 보입니다. Windows 앱 SDK는 아직 정식 릴리즈되지 않았으므로 추후 진행상황을 지켜보기로 합시다.

## 체험
`Microsoft Store`에서 [WinUI 3 Controls Gallery](https://www.microsoft.com/ko-kr/p/winui-3-controls-gallery/9p3jfpwwdzrc?activetab=pivot:overviewtab)를 통해 WinUI 3의 각종 컨트롤들을 체험할 수 있습니다. [Github 소스코드](https://github.com/microsoft/Xaml-Controls-Gallery/tree/winui3)를 통해 갤러리를 직접 컴파일 해볼 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627264300603/RPdtudKXu.png)

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627264342498/GloESQxn9.png)

## 환경구성
`WinUI 3`은 현태 Windows 앱 SDK 0.8.1이 최신 안정화 버젼이며 다음과 같은 시스템 요구 사항을 준수해야 합니다.

### 시스템 요구사항
- Windows 10 버젼 1089(빌드 17763) 이상
- Visual Studio 2019 버젼 16.9 이상
   - 유니버셜 Windows 플랫폼 개발
   - .NET 데스크톱 개발
   - C++를 사용한 데스크톱 개발
- Windows SDK 버전 2004(빌드 19041) 이상 (Visual Studio 2019와 함께 설치됨)

※ Windows 앱 SDK 0.8은 Visual Studio 2019 미리보기 16.11 이상에서 원활하게 지원됩니다. 또한 Visual Studio 2022 미리보기에서는 Windows 앱 SDK VSIX가 설치되지 않습니다.

### 환경구성
먼저 [Windows 앱 SDK](https://marketplace.visualstudio.com/items?itemName=ProjectReunion.MicrosoftProjectReunion)를 Visual Studio 2019 미리보기에서 `확장 관리`를 통해 설치합니다.

확장 관리에서 제공하는 Windows 앱 SDK는 아직 이름이 `Project Reunion`이므로 이 패키지를 설치하도록 합니다.
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627265878124/iLhbeXeTk.png)

`확장 관리`에서 `Windows Template Studio`를 설치할 수 있습니다. Windows Template Studio는 UWP, WPF및 WinUI 3에서 프로젝트를 생성할 때 마법사를 제공해서 데스크톱 앱의 생성을 가속화 하는 Visual Sutdio 2019의 확장 입니다. Windows Template Studio를 설치하면 `새 프로젝트 만들기`에 `App (WinUI 3 in Desktop)`이라는 프로젝트 템플릿이 추가됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627870983381/TzEzVL49L.png)

Visual Studio를 종료해야 VSIX가 설치가 되고 설치가 완료되면 Visual Studio를 재시작합니다.

정상적으로 설치가 완료하면 다음과 같이 `새 프로젝트 만들기`에서 플랫폼을 `Proejct Reunion`으로 했을 때 템플릿 프로젝트가 표시되어야 합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627266373673/gC9Uqk6Ej.png)

## `Hello World` 프로젝트
`새 프로젝트 만들기`에서 `App (WinUI 3 in Desktop)을 선택합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627268228128/VtXvU2ELh.png)

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627268246052/C_GMfaEB3.png)

다음처럼 WinUI 3의 마법사 창을 통해 다양한 프로젝트 유형 및 세부 설정을 변경할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627268324164/jrJl63or1.png)

`Design pattern` 및 `페이지`를 설정할 수 있습니다. 현재 Design pattern은 `MVVM Toolkit`만 지원하지만 차후 사용자의 요구에 따라 다양한 디자인 패턴이 추가될 것 같습니다. `페이지`의 경우 다양한 목적을 위한 페이지 구성을 선택할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627268748615/xOF84jfqt.png)

현재 Windows 앱 SDK 0.8.1의 패키징 방법은 MSIX만 지원합니다. 로드맵에 의해 버젼 1.0에서는 비패키징도 지원할 것으로 보입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627268469110/BBISaiFue.png)

WinUI 3 마법사에 의해 프로젝트가 생성되면 다음의 프로젝트가 생성됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627268993607/m6FwGBgG8.png)

컴파일하여 실행하면 정상적으로 잘 실행되고요,

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627269015715/i5QlRIjEy.png)

WinUI 3 마법사에 의해 생성된 메뉴 및 설정페이지, 웹뷰 페이지도 정상적으로 잘 동작함을 확인할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627269052609/ndG1nFWTt.png)

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627269070244/XJPQFYfzI.png)

### Win2D 적용
`WinUI 3`에서 흥미롭게 살펴볼 수 있는 내용은 바로 [Win2D](https://github.com/microsoft/Win2D)의 지원입니다. Win2D는 2D 렌더링을 GPU 가속을 통해 제공하는 Windows 런타임 API입니다. Win2D를 통해 Direct2D를 데스크톱 어플리케이션에 쉽게 통합할 수 있습니다.
Win2D를 이용하려면 NuGet에서 `Microsoft.Graphics.Win2D` 패키지를 별도로 설치해 사용할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627270600761/3Gmd77goU.png)

설치후 `MainPage.xaml`을 다음처럼 수정합니다.

```xaml
<Page
    x:Class="App6.Views.MainPage"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" xmlns:canvas="using:Microsoft.Graphics.Canvas.UI.Xaml"
    mc:Ignorable="d"
    Style="{StaticResource PageStyle}">

    <Grid x:Name="ContentArea" Margin="{StaticResource MediumLeftRightMargin}">
        <Grid.RowDefinitions>
            <RowDefinition Height="48" />
            <RowDefinition Height="*" />
        </Grid.RowDefinitions>

        <TextBlock
            Grid.Row="0"
            x:Uid="Main_Title"
            Style="{StaticResource PageTitleStyle}" />
        <Grid
            Grid.Row="1" 
            Background="{ThemeResource SystemControlPageBackgroundChromeLowBrush}">

            <canvas:CanvasControl Draw="CanvasControl_Draw" ClearColor="CornflowerBlue" />
        </Grid>
    </Grid>
</Page>
```

`MainPage.xaml.cs`에 `CanvasControl_Draw` 메소드를 다음처럼 추가합니다.
```csharp
        private void CanvasControl_Draw(Microsoft.Graphics.Canvas.UI.Xaml.CanvasControl sender, Microsoft.Graphics.Canvas.UI.Xaml.CanvasDrawEventArgs args)
        {
            args.DrawingSession.DrawEllipse(155, 115, 80, 30, Colors.Black, 3);
            args.DrawingSession.DrawText("Hello, world!", 100, 100, Colors.Yellow);
        }
```

이후 컴파일하여 실행하면 다음과 같이 Win2D의 동작 결과를 확인할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627270749410/CW_uIJMOW.png)

## 로드맵
`WinUI 3`는 올해 4분기를 목표로 개발이 진행되고 있는 `Windows 앱 SDK`의 기능중의 하나입니다. 다음의 [Windows 앱 SDK 로드맵](https://github.com/microsoft/WindowsAppSDK/blob/main/docs/roadmap.md) 및 [WinUI 로드맵](https://github.com/microsoft/microsoft-ui-xaml/blob/main/docs/roadmap.md#winui-3)을 통해 살펴볼 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627266796972/tis_dNxy_.png)
출처: github.com/microsoft/microsoft-ui-xaml

초기 로드맵에서는 `XAML Islands`가 버젼 1.0에 포함되어 있었지만, 최근의 로드맵에서는 향후 업데이트로 옮겨졌음을 알 수 있습니다. 이것의 의미는 올해 4분기에 릴리즈되는 `Windows 앱 SDK`에서는 WPF 및 Windows Forms등에서는 WinUI 3의 컨트롤들을 사용하지 못하는 것으로 보입니다.
하지만 버젼 1.0이 되면 WinUI 3에서 비패키지를 지원하게 되므로 [MSIX](https://docs.microsoft.com/ko-kr/windows/msix/overview)로 패키징 하지 않는 시나리오에서도 WinUI 3을 사용할 수 있게 됩니다.
이외에 다음의 2021년도 중점사항을 살펴볼 수 있습니다.

1. 일관되고 현대적인 상호작용 및 UX 디자인
   - [WinUI 3](https://github.com/microsoft/microsoft-ui-xaml/blob/main/docs/roadmap.md) - Win32 및 UWP용 Windows 10 기본 UI 플랫폼
   - [WebView2](https://docs.microsoft.com/ko-kr/microsoft-edge/webview2/) - 새로운 Edge(Chromium) 엔진을 사용하여 Windows 앱에 웹 컨텐츠 포함
   - [React Native Windows](https://github.com/microsoft/react-native-windows/projects/30) - 이제 WinUI를 대상으로 함
   - [모던 윈도잉](https://github.com/microsoft/WindowsAppSDK/discussions/370)
1. 기기 하드웨어에 최적화
   - 터치, 잉크, 디스플레이 개선
   - ARM64 지원
   - 입력
1. 뛰어난 시스템 성능과 배터리 수명
   - [앱 수명 주기 관리 및 전력 사용을 위한 더 나은 옵션](https://github.com/microsoft/WindowsAppSDK/issues/111)
   - [DirectWrite 텍스트 렌더링 플랫폼](https://github.com/microsoft/WindowsAppSDK/issues/112)
   - [로컬 푸쉬 알림](https://github.com/microsoft/WindowsAppSDK/discussions/371)
   - 백그라운드 작업
1. 간편한 앱 검색 및 관리
   - 향상된 앱 피키징
   - 프레임워크 패키지 배포
   - 모든 앱 유형에 대한 자동 업데이트
1. 플랫폼 통합 및 배포
   - OS에서 Windows 플랫폼 분리
   - 지원되는 모든 Windows 버전에서 기능이 작동하는지 확인
1. Github로 엔지니어링 이동

## 정리
- WinUI 2는 UWP 만의 프레임워크였다면 WinUI 3은 Windows 데스크톱을 개발하기 위한 통합 프레임워크 입니다.
- WinUI 3은 이미 예제소스를 확인할 수 있고 컴파일 할 수 있고 실행할 수 있는 단계입니다.
- 아쉽지만 Windows 앱 SDK 버젼 1.0에서 `XAML Islands`가 빠지면서 WPF 및 Windows Forms에서 WinUI 3를 사용할 수 없습니다. (차후 업데이트에서 지원 예정)
- UWP가 아닌 WinUI 3 Apps 프로젝트로 WinUI 3를 개발할 수 있습니다.
- .NET 5 C# 9 및 .NET 6 C# 10으로 개발할 수 있습니다.
- 배포 시나리오에 따라 MSIX 배포 및 다른 방식의 배포 방식을 선택할 수 있습니다. (현재는 MSIX 배포만 지원)
- WinUI 3를 사용할 수 있는 OS 최소사양은 `Windows 10 버젼 1809` 입니다.
- WinUI 템플릿 프로젝트에서 다양항 레이아웃의 템플릿을 제공합니다.
- Win2D를 WinUI 3에서 사용할 수 있습니다.
