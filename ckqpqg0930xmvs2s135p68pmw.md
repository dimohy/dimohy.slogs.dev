## [C# 10] global using : 좀 더 간결한 예시 코드 언어로 진화

`global using`은 프로젝트 범위 안에서 공통으로 사용하는 대상을 using할 수 있습니다. 개인적인 생각은 C# 9에서 추가된 [최상위문(top-level statement)](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/top-level-statements)과 더불어서 간결한 예시 코드를 제공하고 실행할 수 있도록 하는 문법적 진화라고 생각합니다. 가령,

최상위문을 이용해 다음처럼 코드를 작성할 수 있는데요,

```csharp
using System;

Console.WriteLine("Hello World");
```

`global using`을 이용해 별도의 파일에

```csharp
global using System;
```
한 후, 아래처럼 좀 더 간결하게 표현할 수 있습니다.

```chsarp
Console.WriteLine("Hello World");
```

이는 `using statc` 및 Alias `using`과 결합했을 때 더욱 더 강력해 지는데요, 

```csharp
global using System;
global using static Helpers;
global using Alias = TypeOrNamespace;
```

이 역시도 최상위문을 통해  입문 프로그래머가 `Python`과 유사한 느낌으로 개발을 시작할 수 있도록 하는 보조 장치가 될 것이라고 생각합니다.

별도의 파일에 (예를 들어 Globals.cs)
```csharp
global using static System.Console;
```
한후,

Program.cs를 다음처럼 쓸 수 있게 됩니다.
```csharp
WriteLine("Hello World");
```
