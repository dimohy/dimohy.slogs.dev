---
title: "[Uno] MVUX"
datePublished: Tue Feb 20 2024 12:16:03 GMT+0000 (Coordinated Universal Time)
cuid: clsubx8y6000c08jvdz60hmde
slug: uno-mvux
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1708431400381/459d5b6c-61d6-499a-9797-74dc675dc687.png
tags: dotnet, unoplatform, mvux

---

**MVUX**(**M**odel **V**iew **U**pdate e**X**tended)는 [Elm 아키텍처](https://en.wikipedia.org/wiki/Elm_(programming_language)#The_Elm_Architecture)를 따르는 Uno의 MVU 아키텍처의 확장입니다. MVUX는 데이터 바인딩 기능을 그대로 사용하면서 불변 모델을 기반으로 애플리케이션 상태를 정의하고 사용할 수 있도록 합니다. MVVM의 고질적인 상용구 코드의 번잡함과 스레딩 문제를 해결합니다.

MVUX를 이해하기 앞서서 MVVM 아키텍쳐를 살펴보고 MVVM의 단점에 대해 이야기해봅시다.

![출처: Uno Platform](https://uno-website-assets.s3.amazonaws.com/wp-content/uploads/2023/07/25142313/Untitled-2023-06-06-1402.png align="left")

MVVM 모델은 비즈니스 로직과 화면을 나눠서 로직과 화면의 결합도를 없애 단위 테스트를 용이하게 하는 장점이 있습니다. 반면, ViewModel이 변경되었을 때 View가 반응하도록 하기 위한 메커니즘인 데이터 바인딩을 달성하기 위해 상당한 상용구 코드가 필요하고 비동기 데이터의 경우 다양한 문제가 발생할 수 있습니다.

이에 반해 Uno Platform의 MVUX는 불변 모델을 사용해서 애플리케이션 상태를 정의합니다. 상태가 변경되었을 경우 Uno Platform이 자동으로 생성해주는 바인딩 프록시를 이용해 MVVM의 장점인 데이터 바인딩을 그대로 사용해 상태에 따라 올바르게 View를 갱신합니다.

![](https://uno-website-assets.s3.amazonaws.com/wp-content/uploads/2023/07/25142127/mvux-1.png align="left")

> Uno Platform의 MVUX는 MVU의 장점인 단방향성과 MVVM의 장점인 데이터 바인딩을 결합했다고 할 수 있습니다.

MVUX는 또한 `IFeed<>` 인터페이스를 통해 상태에 대한 피드를 제공해서 상태에 따라 진행중, 데이터 유/무, 오류발생을 적절하게 처리할 수 있습니다.

MVVM에 비해 MVUX가 어떤 장점이 있는지 좀 더 살펴보려면 아래 [참조 글](https://platform.uno/docs/articles/external/uno.extensions/doc/Learn/Mvux/Overview.html?tabs=viewmodel%2Cmodel)을 살펴볼 수 있습니다.

---

이미지 출처 :  
[https://platform.uno/blog/demystifying-mvvm-and-introducing-mvux-approach/](https://platform.uno/blog/demystifying-mvvm-and-introducing-mvux-approach/)

참조 글 : MVUX 개요  
[https://platform.uno/docs/articles/external/uno.extensions/doc/Learn/Mvux/Overview.html?tabs=viewmodel%2Cmodel](https://platform.uno/docs/articles/external/uno.extensions/doc/Learn/Mvux/Overview.html?tabs=viewmodel%2Cmodel)