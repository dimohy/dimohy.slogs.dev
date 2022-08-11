## Fusion 튜토리얼

> 이 문서는 [Stl.Fusion](https://github.com/servicetitan/Stl.Fusion)의 [Fusion Tutorial](https://github.com/servicetitan/Stl.Fusion.Samples/tree/master/docs/tutorial)를 번역한 것입니다.

이것은 [Fusion] - 모든 연결되는 앱에 대해 실시간을 새로운 표준으로 만들려는 .NET 라이브러리입니다. 단순히 찾아볼 수도 있지만 여기에 나오는 모든 C# 코드를 실행하고 수정할 수도 있습니다. [Try .NET](https://github.com/dotnet/try/blob/main/DotNetTryLocal.md) 또는 [Docker](https://www.docker.com/)를 사용해 보기만 하면 됩니다.

이 튜토리얼을 실행하는 가장 간단한 방법:
- [Docker](https://docs.docker.com/get-docker/) 및 [Docker Compose](https://docs.docker.com/compose/install/) 설치
- 이 저장소의 루트 폴더에서 `docker-compose up --build tutorial` 실행
- `https://localhost:50005/README.md` 열기

또는 dotnet try CLI 도구를 사용해서 실행:
- [.NET 6.0 SDK](https://dotnet.microsoft.com/en-us/download)와 [.NET Core 3.1 SDK](https://dotnet.microsoft.com/en-us/download/dotnet) 설치
- [Try .NET](https://github.com/dotnet/try/blob/main/DotNetTryLocal.md) 설치. 릴리스 버전에서 코드를 실행하지 못하면 미리 보기 버전을 설치하세요.
- 이 저장소의 루트 폴더에서 `dotnet try --port 50005 docs/tutorial` 실행
- `https://localhost:50005/README.md` 열기


## 튜토리얼

Fusion 기반 코드는 처음에는 완전히 이상해 보일 수 있습니다 - 이는 코드를 파헤치기 전에 배워야 하는 추상화를 기반으로 하기 때문입니다.

작동 방식을 이해하면 더 많은 질문을 피할 수 있으므로 Fusion 샘플의 소스 코드를 파헤치기 전에 이 튜토리얼을 완료하는 것이 좋습니다.

더 이상 고민하지 말고:

- 빠른 시작: HelloCart 샘플을 통해 Fusion의 80% 알아보기
- 0부: NuGet 패키지
- 1부: Compute 서비스
- 2부: Computed 값: IComputed<T>
- 3부: 상태: IState<T> 느껴보기
- [4부: Replica 서비스](https://dimohy.slogger.today/fusion-4-replica)
- 5부: 서버 측에서만 캐싱 및 Fusion
- 6부: Blazor 앱의 실시간 UI
- 7부: JS / React 앱의 실시간 UI
- 8부: Fusion 서비스 확장
- [9부: CommandR](https://dimohy.slogger.today/fusion-9-commandr)
- 10부: 운영 프레임워크를 사용한 다중 호스트 무효화 및 CQRS
- 11부: Fusion에서 인증
- 에필로그

마지막으로 다음을 확인:

- [Fusion 치트 시트](https://dimohy.slogger.today/fusion-cheat-sheet) - 즐겨찾기에 추가해봅시다 :)
- [개요](https://dimohy.slogger.today/fusion-overview) - Fusion 추상화에 대한 높은 수준의 설명입니다.

> 튜토리얼에 대한 링크는 번역이 완료되면 연결됩니다.

> 오역은 [Stl.Fusion 튜토리얼 번역 - slog](https://forum.dotnetdev.kr/t/stl-fusion-slog/4184)을 통해 알려주세요!
