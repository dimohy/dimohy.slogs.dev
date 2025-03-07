---
title: ".NET Foundation 프로젝트 소개(2): AngleSharp"
datePublished: Thu Nov 25 2021 00:22:05 GMT+0000 (Coordinated Universal Time)
cuid: ckwe7o4a001mdd2s11qpo5kow
slug: net-foundation-2-anglesharp
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1637799781303/uPGSN2xT5.jpeg
tags: dotnet

---

여러분의 시간을 아낄 수 있는 .NET Foundation의 유용한 프로젝트를 소개하는 시간입니다.

오늘 소개하는 프로젝트는 AngleSharp 인데요, AngleSharp는 .NET 표준 라이브러리 형태로 .NET 애플리케이션에서 사용할 수 있는 최신 웹 도구 기반 .NET 브라우저 엔진 코어 입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1637799747868/WqOgA9Bcw.png)
[image|600x400](upload://wiTFhwy8lqfO3TjoOql8hfBm7ip.png)

라이브러리에는 완전히 구현된 HTML5 파서와  L4 쿼리 선택기를 사용하여 탐색할 수 있는 동적 DOM 구현이 포함되어 있습니다. AngleSharp는 W3C 사양과 WHATWG 참조를 완전히 준수하여 에버그린 브라우저와 최대 호환성을 보장합니다.

AngleSharp의 생태계는 통합 CSS3 파서, XPath 지원 및 실험적으로 JavaScript 엔진과 같은 확장 라이브러리를 제공합니다.

AngleSharp의 장기적인 비전은 .NET 애플리케이션 내에서 표준 웹 자산을 다운로드 및 검사, 실행 및 렌더링하기 위한 모든 빌딩 블록을 제공하는 것입니다.

프로젝트 정보 사이트는 다음과 같습니다.
https://anglesharp.github.io/

그리고 프로젝트 소스코드는 이곳에서 살펴볼 수 있고요,
https://github.com/AngleSharp

다음의 코드 처럼 사용할 수 있습니다.

```csharp
var config = Configuration.Default.WithDefaultLoader();
var address = "https://en.wikipedia.org/wiki/List_of_The_Big_Bang_Theory_episodes";
var context = BrowsingContext.New(config);
var document = await context.OpenAsync(address);
var cellSelector = "tr.vevent td:nth-child(3)";
var cells = document.QuerySelectorAll(cellSelector);
var titles = cells.Select(m => m.TextContent);
```

AngleSharp 문서는 이곳을 통해 확인하실 수 있습니다.
https://github.com/AngleSharp/AngleSharp/blob/devel/docs/README.md

주요 기능은 다음과 같습니다.

- 이식성 (.NET Standard 2.0 사용)
- 표준 준수 (에버그린 브라우저와 동일하게 작동)
- 뛰어난 성능 (대부분의 시나리오에서 유사한 파서를 능가)
- 확장 가능 (자체 서비스로 확장)
- 유용한 추상화 (유형 도우미, jQuery와 같은 구성)
- 완전한 기능의 DOM (목록, 반복자, 이벤트 등 알고 있는 모든 것)
- 양식 제출 (어디서나 쉽게 로그인)
- 탐색 (`BrowsingContext`는 브라우저 탭과 같습니다. .NET에서 제어합니다!)
- 향상된 LINQ (DOM 요소와 함께 LINQ 사용, 자연스럽게 래퍼 없이 사용)