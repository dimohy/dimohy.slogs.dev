---
title: ".NET Foundation 프로젝트 소개(5): Polly"
datePublished: Fri Dec 10 2021 12:06:10 GMT+0000 (Coordinated Universal Time)
cuid: ckx0cfgmc000ozvs1ed1c0vqe
slug: net-foundation-5-polly
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1639137918623/SuFZxkmBo.png
tags: dotnet

---

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1639137918623/SuFZxkmBo.png)

여러분의 시간을 아낄 수 있는 .NET Foundation에서 후원하는 유용한 프로젝트를 소개하는 시간입니다.

오늘 소개할 프로젝트는 폴리(Polly)입니다.

폴리는 개발자가 유용하게 사용할 수 있는 재시도, 회로(실행) 차단, 시간초과, 격벽 격리 및 대체와 같은 정책을 Fluent 스타일을 통해 스레드에 안전한 방식으로 사용할 수 있는 복원 및 일시적 오류를 처리하는 라이브러리 입니다.

재시도라던가 시간초과 처리는 소스코드에서 빈번히 사용할 수 있는 패턴입니다. 이외에 폴리에서 제공하는 복원 관련 기능은 다음과 같습니다.

- [Retry](https://github.com/App-vNext/Polly#circuit-breaker) / 재시도 : 자동 재시도를 구성할 수 있습니다.
- [Circuit-breaker](https://github.com/App-vNext/Polly#circuit-breaker) / 서킷브레이커, 회로(실행)차단 : 오류가 임계값을 초과하면 일정 시간 동안 실행을 차단 합니다.
- [Timeout](https://github.com/App-vNext/Polly#timeout) / 시간초과 : 시간초과가 발생하면 더이상 기다리지 않습니다.
- [Bulkhead Isolation](https://github.com/App-vNext/Polly#bulkhead) / 격벽 격리 : 통제되는 작업을 고정 크기 리소스 풀로 제한해서 다른 영역에 영향을 주지 않도록 합니다.
- [Cache](https://github.com/App-vNext/Polly#cache) / 캐시 : 캐시를 이용해 자동으로 응답을 저장하고 사용할 수 있도록 합니다.
- [Fallback](https://github.com/App-vNext/Polly#fallback) / 대체 : 실패시 대체 값을 반환하거나 작업을 실행 합니다.
- [PolicyWrap](https://github.com/App-vNext/Polly#policywrap) : 위의 정책들을 유연하게 결합할 수 있습니다.

폴리는 강력하고 충분히 안정적이므로 위의 빠른시작 링크를 통해 간단히 체험 후 코드에 바로 적용해볼 수 있습니다.
