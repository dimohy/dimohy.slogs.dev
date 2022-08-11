## .NET Foundation 프로젝트 소개(4): Avalonia

여러분의 시간을 아낄 수 있는 .NET Foundation에서 후원하는 유용한 프로젝트를 소개하는 시간입니다.

오늘 소개하는 프로젝트는 Avalonia 인데요, [Avalonia UI Framework](https://avaloniaui.net/index.html) 는 XAML 스타일의 유연한 스타일링 시스템을 제공하고 Windows (.NET Framework, .NET) 및 Linux(Xorg를 통한) 및 macOS와 같은 광범위한 운영 체제를 지원하는 크로스 플랫폼 XAML 기반 UI 프레임워크입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1638577688896/eaDCk4CAQ.png)

MAUI는 모바일 환경에 집중한다면, Avalonia는 크로스플렛폼 데스크탑 애플리케이션을 만든는데 좀 더 특화되어 있습니다.

## Avalonia 특징
- WPF와 유사한 XAML 스타일로 개발을 바로 시작할 수 있습니다!
- 윈도우 및 리눅스, iOS에서 동작 (라즈베리파이에서도 동작합니다!)
- 이제 Avalonia는 현업에서 사용할 수 있을 정도로 안정화 되었습니다.
- 활발한 커뮤니티에 의해 성장하고 있습니다.

## Avalonia XAML 컨트롤 갤러리
- Microsoft Store
https://www.microsoft.com/store/productId/9PGHN6QNJ3RP

## Avalonia 간단하게 체험하기

### Avalonia 프로젝트 템플릿 설치
만약 `Visual Studio 2022`를 사용하신다면 Avalonia은 아직 확장기능 설치를 제공하지 않습니다. 다음 처럼 `dotnet`을 이용해 프로젝트 템플릿을 설치할 수 있습니다.

```shell
$ dotnet new -i Avalonia.Templates
다음 템플릿 패키지가 설치됩니다.
   Avalonia.Templates

성공:Avalonia.Templates::0.10.10이(가) 다음 템플릿을 설치했습니다.
템플릿 이름                        약식 이름                      언어       태그
----------------------------  -------------------------  -------  ---------------------------
Avalonia .NET Core App        avalonia.app               [C#],F#  ui/xaml/avalonia/avaloniaui
Avalonia .NET Core MVVM App   avalonia.mvvm              [C#],F#  ui/xaml/avalonia/avaloniaui
Avalonia Resource Dictionary  avalonia.resource                   ui/xaml/avalonia/avaloniaui
Avalonia Styles               avalonia.styles                     ui/xaml/avalonia/avaloniaui
Avalonia TemplatedControl     avalonia.templatedcontrol  [C#]     ui/xaml/avalonia/avaloniaui
Avalonia UserControl          avalonia.usercontrol       [C#],F#  ui/xaml/avalonia/avaloniaui
Avalonia Window               avalonia.window            [C#],F#  ui/xaml/avalonia/avaloniaui
```

## Avalonia 프로젝트 템플릿을 이용해 실행

`새 프로젝트 만들기`에서 `Avalonia .NET Core App (AvaloniaUI)`를 선택하면 기폰 프로젝트가 생성됩니다.

![image|690x430](upload://trIwQsCcNTktqwu7FgtVV5Uf3Fo.png)

![image|690x455](upload://nTb5rmS6RHAaNd4CHyvCS6YqdvX.png)

WPF와 유사한 구조로 프로젝트가 만들어졌음을 확인할 수 있는데요, 실행해 볼 수 있습니다.

![image|690x453](upload://aILdXj875n2aubPiJu3kSvXN73q.png)

## WSL을 통해 실행
Avalonia는 크로스플랫폼 UI Framework입니다. 리눅스에서도 그대로 동작하는데요, WSL이 설치되어 있고 [WSLg](https://devblogs.microsoft.com/commandline/wslg-architecture/)이 동작할 수 있는 환경이라면 특별한 작업 없이 바로 WSL로 실행해볼 수 있습니다!

WSL 프로파일을 선택한 후 실행하면,
![image|280x218](upload://xzubPtGRlxUEdtHr1o0DqhoUTIH.png)

다음처럼 WSL에서 실행되는 Avalonia 애플리케이션을 볼 수 있습니다!

![image|649x500](upload://luptZREcjtqjVnMTn5skPYyj7nm.png)