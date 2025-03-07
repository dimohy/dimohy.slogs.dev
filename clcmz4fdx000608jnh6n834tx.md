---
title: "is  연산자를 이용해서 중복 할당하지 않고 비교문에서 사용"
datePublished: Sun Jan 08 2023 06:08:03 GMT+0000 (Coordinated Universal Time)
cuid: clcmz4fdx000608jnh6n834tx
slug: is
tags: csharp

---

```csharp
var input = "15,2";
```

`,`으로 값을 취하고자 할 때 보통 다음의 코드로 취할 수 있습니다.

```csharp
var items = input.Split(',');
var (a, b) = (items[0], items[1]);
```

LINQ써서 다음처럼 할 수도 있습니다.

```csharp
(a, b) = input.Split(',').Chunk(2).Select(x => (x[0], x[1])).First();
```

그런데 `switch 식` 등 `식`에서 바로 처리하고 싶을 때가 있습니다. 그럴때 `is` 키워드를 이용해서 다음 처럼 할 수 있습니다.

```csharp
// 입력 값이 15이면서 3이 아닌 경우 참
var bResult = input.Split(',') is string[] items2 is true && items2[0] is "15" && items2[1] is not "3";
```