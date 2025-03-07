---
title: "Fusion 튜토리얼 - 0부: NuGet 패키지"
datePublished: Thu Aug 11 2022 13:34:17 GMT+0000 (Coordinated Universal Time)
cuid: cl6p30j1u00b3fdnvdjcm8aep
slug: fusion-0-nuget
tags: dotnet, stlfusion

---

모든 Fusion 패키지는 [NuGet에서 사용](https://www.nuget.org/packages?q=Owner%3Aservicetitan+Tags%3Astl_fusion)할 수 있습니다.

다음을 참조해야 합니다.

- `Stl.Fusion.Server` – 서버 측 어셈블리
  - .NET Framework 4.X를 사용하는 경우 대신 `Stl.Fusion.Server.NetFx`를 참조
- `Stl.Fusion.Client` – 클라이언트 측 어셈블리
  - Blazor 클라이언트는 대신 `Stl.Fusion.Client`를 참조하는 `Stl.Fusion.Blazor`를 참조
- `Stl.Fusion` – 양쪽에서 사용하는 공유 어셈블리
- `Stl.Fusion.EntityFramework` – [EF Core](https://docs.microsoft.com/en-us/ef/)를 사용하려는 경우 서버 측 어셈블리에서 사용

Fusion 전체 패키지 목록:

- [Stl](https://www.nuget.org/packages/Stl/) - "ServiceTitan Library"를 나타냄(예, 모든 회사에는 자체 [STL](https://en.wikipedia.org/wiki/Standard_Template_Library)이 필요). BCL에서는 찾을 수 없었던 비교적 고립된 추상화 및 도우미 모음. `Stl`은 [Castle.Core](https://www.nuget.org/packages/Castle.Core/)에 의존함. 기타 타사 패키지일 수도 있음
- [Stl.Net](https://www.nuget.org/packages/Stl.Net/) - `Stl`에 의존적. `WebSocketChannel`은 현재 포함된 유일한 타입
- [Stl.Interception](https://www.nuget.org/packages/Stl.Interception/) - `Stl`에 의존적. [Castle DynamicProxy](http://www.castleproject.org/projects/dynamicproxy/)를 기반인 호출 차단 도우미
- [Stl.CommandR](https://www.nuget.org/packages/Stl.CommandR/) - `Stl` 및 `Stl.Interception`에 의존적. CommandR은 인터페이스 기반 명령 핸들러 뿐 만 아니라 일반 메소드로 작성된 AOP 스타일 핸들러도 지원하도록 설계된 "MediatR 스테로이드"입니다. 그 외에도 명령 핸들러 API를 통합하고(파이프라인 동작과 핸들러가 동일함) 그렇지 않으면 가질 수 있는 거의 모든 상용구 코드를 제거하는 데 도움이 됩니다.
- [Stl.Fusion](https://www.nuget.org/packages/Stl.Fusion/) - `Stl`, `Stl.Interception` 및 `Stl.CommandR`에 의존적. Fusion과 관련된 거의 모든 것.
- [Stl.Fusion.Server](https://www.nuget.org/packages/Stl.Fusion.Server/) - `Stl.Fusion` 및 `Stl.Net`에 의존적. 클라이언트 측 상대방이 Fusion `Publisher`와 통신할 수 있도록 서버 측 WebSocket 끝점을 구현합니다. 또한 퓨전 API 컨트롤러(`FusionController`)에 대한 기본 클래스와 웹 앱에 이 모든 것을 등록하는 데 도움이 되는 몇 가지 확장 메서드를 제공합니다.
- [Stl.Fusion.Server.NetFx](https://www.nuget.org/packages/Stl.Fusion.Server.NetFx/) - .NET Framework 4.X 버젼의 `Stl.Fusion.Server`
- [Stl.Fusion.Client](https://www.nuget.org/packages/Stl.Fusion.Client/) - `Stl.Fusion` 및 `Stl.Net`에 의존적. `FusionControler` 기반 API 끝점과 호환되는 클라이언트 측 WebSocket 통신 채널 및 RestEase 기반 API 클라이언트 빌더를 구현함. 이 모든 것을 함께 사용하면 클라이언트에서 서버 측 대상을 "미러링"하는 컴퓨팅 인스턴스를 얻을 수 있음
- [Stl.Fusion.Blazor](https://www.nuget.org/packages/Stl.Fusion.Blazor/) - `Stl.Fusion.Client`에 의존적. 편리한 Blazor 구성 요소를 구현합니다. 현재 `StatefulCompontentBase<TState>`와 2개의 하위 항목인 `ComputedStateComponent<T>` 및 `ComputedStateComponent<T, TLocals>`가 있습니다.
- [Stl.Fusion.EntityFramework](https://www.nuget.org/packages/Stl.Fusion.EntityFramework/) - `Stl.Fusion`에 의존적. Fusion 인증, 작업 지속성, IKeyValueStore 등의 [EF Core](https://docs.microsoft.com/en-us/ef/) 기반 구현을 포함
- [Stl.Fusion.EntityFramework.Npgsql](https://www.nuget.org/packages/Stl.Fusion.EntityFramework.Npgsql/) - `Stl.Fusion.EntityFramework` 의존적.
Npgsql 기반 작업 로그 변경 추적 구현을 포함. PostgreSQL에는 메시지 대기열로 사용할 수 있는 [NOTIFY / LISTEN](https://www.postgresql.org/docs/13/sql-notify.html) 명령이 있으므로 이 데이터베이스를 사용하는 경우 Fusion이 작업 로그 변경에 대해 피어 호스트에 알릴 수 있도록 별도의 메시지 대기열이 필요하지 않음
