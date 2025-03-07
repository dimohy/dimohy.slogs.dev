---
title: "Visual Studio 2022에 Uno 플랫폼 개발 환경 구성"
datePublished: Thu Apr 28 2022 03:17:13 GMT+0000 (Coordinated Universal Time)
cuid: cl2ifqirr029larnv74fq01sk
slug: visual-studio-2022-uno
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1651112889662/KPNPMCUYy.png
tags: dotnet, unoplatform

---

[Uno 플랫폼](https://platform.uno/)은 Pixel-Perfect을 지향하는 .NET 멀티 플랫폼 용 UI 오픈소스 플랫폼입니다.

Pixel-Perfect란 앱이 구동하는 장치와 상관없이 동일한 형태의 UI를 제공한다는 뜻으로 일장 일단은 있겠지만 이를 통해서 앱을 개발하면 어느 장치이던 동작화면을 기대하는 모습으로 만들 수 있어 장점입니다.

## 개발환경
- Visual Studio 2022 (최신의 기능을 확인하기 위해서는 Preview)


## 환경구성

Uno 플랫폼 개발 환경을 구성하는 것은 무엇보다 쉽습니다. Uno는 `uno.check`라는 도구를 통해 개발 환경을 구성할 수 있도록 하는데 다음처럼 관리자 권한을 통해 설치할 수 있습니다.

### 최신 워크로드 확인 및 설치
```shell
dotnet tool install -g uno.check
```

만약 이미 설치되어 있다면 최신 버젼으로 업데이트 하기 위한 명령 다음과 같습니다.

```shell
dotnet tool update -g uno.check
```

`uno.check`가 설치가 되었으면 다음의 명령으로 개발 환경을 구성할 수 있습니다.

```shell
uno-check
```

그런데 Visual Studio 2022 미리보기 환경일 경우 진행이 실패할 수 있는데 그런 경우 다음처럼 `--dev` 옵션을 줘서 설치를 진행 할 수 있습니다.

```shell
uno-check --dev
```

### 솔루션 템플릿 설치
`확장 > 확장관리` 메뉴에서 온라인을 선택한 후 `Uno Platform Solution Templates`를 검색해 설치합니다.

이제 새 프로젝트 만들기에서 Uno 관련 프로젝트를 확인할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651112889662/KPNPMCUYy.png)

## 템플릿을 이용해 Uno 앱 컴파일 및 실행

`새 프로젝트 만들기`에서 Multi-Platform App (Uno Platform|net6)`을 선택해 프로젝트가 만들어졌고 만약 추가 구성 요소를 설치해야 한다면 솔루션 탐색기 상단에 다음과 같은 화면을 볼 수 있는데요,

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651113076848/MbMWGAJTp.png)

설치 링크를 통해 추가 구성요소를 설치할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651113099864/Jf80Js6Lu.png)

설치가 완료되면 템플릿 프로젝트를 컴파일 할 수 있게 됩니다.

> - Windows SDK 관련 오류가 발생한다면 `Appxx.UWP` 프로젝트의 속성에서 `애플리케이션/대상 지정`의 대상 버전 및 최소 버전을 `Windows 10, version 1903`으로 변경해보세요.
> - 컴파일 시 `x64`관련 오류가 발생한다면 활성 솔루션 플랫폼을 `x64`로 변경해보세요.

솔루션 탐색기를 보면 `Mobile` 및 `Gtk`, `Wpf` 그리고 `UWP`와 `Wasm` 프로젝트를 확인할 수 있는데 각각을 시작 프로젝트로 설정하고 Uno 프로젝트를 시작할 수 있습니다.

저는 `Wasm`(웹어셈블리) 프로젝트의 동작이 궁금한데요, `Wasm` 프로젝트를 시작 프로젝트로 설정한 후 실행해보겠습니다.

다음처럼 인증서 관련 다이얼로그 창이 뜨는데 `예`를 선택해 설치를 하면, 웹브라우저에서 동작하는 Uno 앱을 확인할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651114066932/_UYGncXMW.png)

동작하는 모습을 좀 더 확인하고 싶은데요, 버튼을 누르면 카운트 되게 간단히 코딩 한 후 확인해보겠습니다.

화면에 대한 XAML은 `Appxx.Shared` 프로젝트의 `MainPage.xaml`입니다. 이것을 다음 처럼 수정 합니다.

```xaml
...
    <StackPanel
        HorizontalAlignment="Center"
        VerticalAlignment="Center"
        Orientation="Vertical">
        <TextBlock
            Margin="20"
            FontSize="30"
            Text="{x:Bind Greeting, Mode=OneWay}" />
        <Button x:Name="counterButton" Click="counterButtonClick">Count</Button>
    </StackPanel>
...
```

다음으로 `MainPage.xaml.cs`을 다음처럼 수정합니다.

```csharp
    public sealed partial class MainPage : Page, INotifyPropertyChanged
    {
        public event PropertyChangedEventHandler PropertyChanged;

        public string Greeting => $"Hello Uno! : {Count}";

        private int _count;

        public int Count
        {
            get => _count;
            set
            {
                _count++;

                OnPropertyChanged(nameof(Count));
                OnPropertyChanged(nameof(Greeting));
            }
        }

        private void OnPropertyChanged(string propertyName)
        {
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }

        private void counterButtonClick(object sender, RoutedEventArgs e)
        {
            Count++;
        }

        public MainPage()
        {
            this.InitializeComponent();
            //this.DataContext = this;
        }
    }
```

이제 실행해봅시다. 버튼을 눌렀을 때 카운트가 잘 올라가는 것을 확인할 수 있습니다!

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651115592179/tN2NE9ULQ.png)

## 정리
Uno 플랫폼을 안지 상당히 오래 되었는데요 앱을 처음 시작할 때의 시작 시간과 동작 속도가 느려서 아쉬웠었습니다. 그런데 오래간만에 확인을 해보니 시작 시간도 쓸 수 있을 만큼 빨라졌고 동작 속도도 빠름을 확인할 수 있었습니다.

Visual Studio 개발환경에서 아직은 XAML 편집기와 완전히 통합이 안된 모습과 소스 생성기의 결과가 편집기에 반영이 늦게 되는 등의 문제는 아직 있지만 곧 개선될 것으로 보이고 Uno 플랫폼의 지향점인 Pixel-Perfect이 매력적임을 강조하면서 마무리 하겠습니다.
