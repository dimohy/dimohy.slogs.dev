---
title: "Visual Studio Code 에서 C#을 프로그래밍 하는 방법 | Claudio Bernasconi"
datePublished: Sat Jul 15 2023 00:47:34 GMT+0000 (Coordinated Universal Time)
cuid: clk3agfvp000409ml7sxmbzcf
slug: visual-studio-code-c-claudio-bernasconi
canonical: https://www.claudiobernasconi.ch/2023/07/13/how-to-program-csharp-in-visual-studio-code/
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1689381299213/8e98b900-2e1b-4c89-a979-43902c5204df.png
tags: net, dotnet, vscode-extensions

---

> [Claudio Bernasconi](https://www.claudiobernasconi.ch/author/berni16/)님의 [**How to Program C# in Visual Studio Code**](https://www.claudiobernasconi.ch/2023/07/13/how-to-program-csharp-in-visual-studio-code/)**를 번역하였습니다.**

Visual Studio Code에서 C#을 개발하는 것은 간단하고 비용도 들지 않으며 뛰어난 크로스 플랫폼 개발자 환경을 제공합니다.

%[https://youtu.be/1PJ5lT4nCbM] 

## C# Dev Kit 확장 설치

[Visual Studio Code](https://code.visualstudio.com/)를 열고 [C# Dev Kit 확장](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)을 설치합니다. 이 확장 프로그램은 C# 애플리케이션을 개발하는 데 필요한 모든 것을 제공하는 유일한 확장 프로그램입니다.

설치가 완료되면 시작 화면에서 Microsoft 계정을 연결하고, .NET SDK를 설치하고, 새 .NET 프로젝트를 만들 수 있습니다.

dotnet --version 명령을 사용하여 이미 .NET SDK가 설치되어 있는지 확인할 수 있습니다. 그렇지 않은 경우 현재 버전을 설치해야 합니다.

## 콘솔 애플리케이션 생성 및 실행

[명령 팔레트](https://learn.microsoft.com/en-us/windows/terminal/command-palette)를 열려면 Ctrl+Shift+P 단축키를 사용할 수 있습니다. [C# Dev Kit 확장](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)에는 **.NET 새 프로젝트** 명령을 비롯한 새로운 명령이 추가되었습니다. 이 명령을 사용하여 콘솔 애플리케이션 프로젝트 유형을 선택합니다.

디렉터리를 열지 않은 경우 Visual Studio Code에서 프로젝트를 만들 위치를 묻습니다. 프로젝트 이름을 입력할 수도 있습니다.

![Visual Studio Code - New Console App: Project Structure](https://www.claudiobernasconi.ch/wp-content/uploads/2023/07/vscode-new-console-app.png align="left")

확장은 **Program.cs** 파일 및 솔루션 파일을 포함하여 프로젝트(ConsoleApp1.csproj)를 만듭니다.

**Program.cs** 파일에서 코드를 "Hello World"에서 "Hello from Visual Studio Code!"로 변경합니다.

터미널을 열고 프로젝트 폴더로 디렉터리를 변경합니다. 다음으로 **dotnet run** 명령을 사용하여 콘솔 애플리케이션을 실행합니다.

![Running a console app in Visual Studio Code](https://www.claudiobernasconi.ch/wp-content/uploads/2023/07/vscode-run-console-app.png align="left")

보시다시피 출력에 "Hello from Visual Studio Code"라는 텍스트가 나타납니다.

## Visual Studio Code로 C# 디버깅

예제를 빠르게 확장해 보겠습니다. **age**라는 **int** 변수를 정의하고 32를 할당합니다. 그런 다음 콘솔에 값을 출력합니다. 행 번호 왼쪽을 클릭하여 중단점을 설정합니다.

![Console App with a breakpoint](https://www.claudiobernasconi.ch/wp-content/uploads/2023/07/vscode-breakpoint.png align="left")

다음으로 **실행 및 디버그** 메뉴를 열고 **모든 자동 디버그 구성 표시** 버튼을 누른 다음 콘솔 애플리케이션을 선택합니다. 사용자 인터페이스가 변경되고 녹색 **실행** 버튼을 사용하여 디버거가 연결된 애플리케이션을 시작할 수 있습니다.

![Visual Studio Code: Debugging C# Code](https://www.claudiobernasconi.ch/wp-content/uploads/2023/07/vscode-debugging.png align="left")

시각적 오버레이를 사용하거나 키보드 단축키를 사용하여 코드를 탐색할 수 있습니다.

## 코드 탐색 및 코드 스니펫

컨텍스트 메뉴는 다양한 옵션을 제공합니다. 예를 들어 **모든 참조 찾기** 명령을 사용할 수 있습니다. 또한 **F2**와 같은 다른 단축키를 사용하여 전체 코드 파일에서 변수 이름을 바꿀 수도 있습니다.

[C# Dev Kit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)는 개발자 경험을 향상시키는 많은 유용한 기능을 제공합니다.

더 많은 도움이 필요하다면 Visual Studio Code에서도 사용할 수 있는 GitHub Copilot 확장 프로그램을 설치하면 됩니다.

C# 개발자 키트 확장 프로그램과 함께 제공되는 코드 스니펫을 사용할 수도 있습니다. 예를 들어 **try** 코드 조각을 사용하여 시도-캐치 코드 블록을 생성할 수 있습니다. 클래스, 생성자, 속성 등을 생성하기 위한 추가 코드 조각도 있습니다.

**CTRL+T** 바로 가기 키를 사용하여 코드 기호를 검색할 수 있습니다. 예를 들어 클래스, 속성 또는 다른 코드 파일로 이동할 수 있습니다.

## Blazor 서버 애플리케이션 만들기

간단한 튜토리얼용으로도 좋지만 프로덕션 사용 사례에도 사용할 수 있을까요? 명령 팔레트를 사용하여 **Blazor Server 애플리케이션**을 만들어 보겠습니다.

![Visual Studio Code: Blazor Server App Project Structure](https://www.claudiobernasconi.ch/wp-content/uploads/2023/07/vscode-blazor-server-app.png align="left")

Visual Studio Code가 전체 Blazor Server 애플리케이션을 생성했습니다. **dotnet run** 명령을 사용하여 웹 애플리케이션을 컴파일하고 실행할 수 있습니다.

코드를 몇 가지 변경해 보겠습니다. 보시다시피 지금은 브라우저에 반영되지 않습니다.

애플리케이션을 중지하고 대신 **dotnet watch** 를 사용해 봅시다. 그러면 브라우저에서 웹 애플리케이션이 다시 실행됩니다. 하지만 이번에는 Blazor 컴포넌트에서 코드를 변경하고 파일을 저장하면 변경 사항이 즉시 반영됩니다.

![Visual Studio Code Live Update using dotnet watch for Blazor Server](https://www.claudiobernasconi.ch/wp-content/uploads/2023/07/vscode-dotnet-watch.png align="left")

## 결론

앞서 살펴본 바와 같이 Visual Studio Code를 사용하여 최신 .NET 응용 프로그램을 개발할 수 있습니다.

[C# Dev Kit 확장](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)은 Visual Studio Code 내에서 새 프로젝트를 생성하고 **코드 스니펫**을 추가하며 C# 코드를 편리하게 탐색하고 변경할 수 있는 명령을 제공합니다.

이제 핑계를 버리고 C# 애플리케이션 개발을 시작하세요.

.NET 개발에 대해 자세히 알아보려면 제 [YouTube 채널](https://www.youtube.com/claudiobernasconi)을 **구독**하세요.

---

%[https://www.claudiobernasconi.ch/2023/07/13/how-to-program-csharp-in-visual-studio-code/]