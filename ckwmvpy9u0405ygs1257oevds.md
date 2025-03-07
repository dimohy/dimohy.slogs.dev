---
title: ".NET Foundation 프로젝트 소개(3): AutoMapper"
datePublished: Wed Dec 01 2021 01:57:31 GMT+0000 (Coordinated Universal Time)
cuid: ckwmvpy9u0405ygs1257oevds
slug: net-foundation-3-automapper
tags: dotnet

---

여러분의 시간을 아낄 수 있는 .NET Foundation에서 후원하는 유용한 프로젝트를 소개하는 시간입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1638323815151/vpdaEfskz.png)

오늘 소개하는 프로젝트는 AutoMapper 인데요, AutoMapper는 규칙 기반의 객체 대 객체 매퍼입니다.
우리는 데이터베이스 로직과 비지니스 로직을 분리하기 위해 데이터베이스 관련 질의/결과값을 DTO(Data Transfer  Object) 개체를 통해 하고 비지니스 로직에서는 DAO(Data Access Object) 개체를 이용합니다. 그렇기 때문에 DTO <--> DAO 간의 변환 작업이 필요한데 이는 굉장히 지루한 작업이 됩니다.

AutoMapper는 간단한 규칙을 통해 수백 또는 수천 줄의 코드를 제거하고 DTO 설계 정책을 적용하여 매핑을 쉽게 테스트 할 수 있도록 합니다.

https://automapper.org/

## AutoMapper란?

객체-객체 매퍼로, 한 유형의 입력 객체를 다른 유형의 출력 객체로 변환해줍니다.  이런 변환 작업이 AutoMapper의 규칙을 따르는 한 거의 제로 구성으로 가능하게 됩니다.

## AutoMapper를 사용하는 이유는요?

매핑 코드는 아주 지루한 작업입니다. 매핑 코드를 테스트하는 것은 더욱 더 지루합니다. AutoMapper는 유형의 간단한 구성과 매핑 테스트를 제공합니다. 진짜 질문은 "왜 개체-개체 매핑을 사용하는가?"일 텐데요, 여러가지 다른 계층을 통과하는, 예를 들어 UI/도메인 계층 또는 서비스/도메인 계층과 같은 경계에서 발생합니다. 각각의 계층은 관심사가 다르므로 개체-개체 매핑은 각 계층의 관심사가 해당 유형에만 영향을 줄 수 있는 분리된 모델이 가능하게 합니다.

## AutoMapper는 어떻게 사용하나요?
개체-개체 매핑이므로 먼저 소스와 대상 유형이 필요합니다. 그다음 가장 간단한 규칙은 `동일명의 속성이 있을 때 그대로 값이 매핑`되는 것입니다. 가령 소스에 `FirstName` 속성이 있다면 이 값은 대상 `FirstName`으로 그대로 매핑이 됩니다. AutoMapper는 또한 유용한 기능인 [Flattening(평면화)](https://docs.automapper.org/en/latest/Flattening.html)를 지원합니다.

AutoMapper는 기본적으로 null 참조 예외를 무시합니다. 이것은 의도된 설계로 원하지 않을때는 [사용자 값 해석기](https://docs.automapper.org/en/latest/Custom-value-resolvers.html)를 이용해 변경할 수 있습니다.

소스와 대상이 있다면 `MapperConfiguration`및 `CreateMap()`을 통해 매핑할 수 있습니다.

```csharp
var config = new MapperConfiguration(cfg => cfg.CreateMap<Order, OrderDto>());
```

왼쪽 유형은 소스 유형이고 오른쪽 유형은 대상 유형입니다. `Map()` 메소드를 호출해서 매핑을 수행할 수 있습니다.

```csharp
var mapper = config.CreateMapper();
// 또는
var mapper = new Mapper(config);

OrderDto dto = mapper.Map<OrderDto>(order);
```

대부분의 애플리케이션에서는 종속성 주입을 이용해서 `IMapper` 인스턴스를 통해 사용할 수 있습니다.

AutoMapper는 또한 컴파일 시점에 유형을 모르는 경우에 대한 비제네릭 버전도 있습니다.


## AutoMapper는 어디에서 구성합니까?

구성은 AppDomain당 한번만 수행하면 됩니다. 일반적으로 응용 프로그램의 시작 시점입니다.

## 매핑을 어떻게 테스트합니까?

매핑을 테스트 하려면 다음 두가지 작업을 수행하는 테스트를 만들어야 합니다.

- 부트스트래퍼 클래스를 호출하여 모든 매핑 생성
- MapperConfiguration.AssertConfigurationsValid 호출

```csharp
var config = AutoMapperConfiguration.Configure();

config.AssertConfigurationIsValid();
```

## 문서

https://docs.automapper.org/

## 소스코드

https://github.com/AutoMapper/AutoMapper

## 더 많은 지원

- [AutoMapper.Data](https://github.com/AutoMapper/AutoMapper.Data) : ADO.NET 지원
- [AutoMapper.EF6](https://github.com/AutoMapper/AutoMapper.EF6): EF6용 확장 메서드
- [AutoMapper.Collection](https://github.com/AutoMapper/AutoMapper.Collection): 동등성을 통한 컬렉션 매핑
- [ExpressionMapping](https://github.com/AutoMapper/AutoMapper.Extensions.ExpressionMapping): Linq 방식 매핑