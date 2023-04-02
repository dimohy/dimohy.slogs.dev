---
title: "SIMD를 사용한 스테로이드의 LINQ | Steven Giesel"
datePublished: Sun Apr 02 2023 02:26:07 GMT+0000 (Coordinated Universal Time)
cuid: clfys6krq000509kx6mq6dll9
slug: simd-linq-steven-giesel
tags: csharp

---

> 본 글은 Steven Giesel님의 [LINQ on steroids with SIMD](https://steven-giesel.com/blogPost/faf06188-bae9-484d-804d-a42d58d18cad) 글을 번역한 것입니다.
> C#의 `Generic Math` 기능은 `제네릭 연산`으로 번역하였습니다.

이 블로그 게시물에서는 LINQ 쿼리 속도를 높이기 위해 SIMD 명령어를 사용하는 방법을 살펴봅니다. 데이터 배열에 SIMD 연산을 수행하는 [Vector](https://learn.microsoft.com/en-us/dotnet/api/system.numerics.vector-1?view=net-7.0) 타입을 사용하겠습니다. 또한 [BenchmarkDotNet](https://benchmarkdotnet.org/) 라이브러리를 사용해 코드의 성능을 측정할 것입니다. 또한 이것이 C# 10의 새로운 "제네릭 연산" 기능과 함께 어떻게 작동하는지 살펴볼 것입니다.

## "문제"

숫자 목록이 있고 모든 숫자의 합을 구하고 싶다고 가정해 보겠습니다. 이 작업을 가장 빠르게 수행하는 방법은 무엇일까요? `for` 루프를 사용하여 각 숫자를 변수에 추가할 수 있습니다. 하지만 이것이 가장 빠른 방법은 아닙니다. 가장 빠른 방법은 SIMD 명령어를 사용하는 것입니다. SIMD는 "단일 명령어 다중 데이터"의 약자입니다. 여러 데이터 포인트에 대해 동시에 동일한 연산을 수행할 수 있습니다. 이는 각 데이터 포인트에 대해 동일한 연산을 하나씩 수행하는 것보다 훨씬 빠릅니다. 경고 한마디: 여기에는 두 가지 함정이 있습니다. 1. 복잡성. 분명히 이 접근 방식은 단순한 루프나 LINQ를 직접 사용하는 것보다 훨씬 더 고급입니다. 2. 작은 데이터 집합이나 성능이 중요하지 않은 경우에는 이 접근 방식을 권장하지 않습니다. 전에도 말씀드렸고 다시 말씀드립니다: 엉뚱한 곳에서 최적화하기 전에 먼저 측정하세요!

다시 본론으로 돌아가서 어떻게 할 수 있는지 알아보겠습니다.

## `Vector<T>` 타입

SIMD 연산에 대한 진입점은 `Vector<T>` 타입입니다. 이는 데이터 벡터를 담을 수 있는 일반적인 타입입니다. `T` 타입 인자는 모든 숫자 타입이 될 수 있습니다. `Vector<T>` 타입에는 포함할 수 있는 요소의 수를 알려주는 `Count` 속성이 있습니다. 이 속성이 중요한 이유는 예를 들어 서로 다른 두 개의 벡터를 추가하는 경우 "하나의" 연산으로 이 작업을 수행하기 때문입니다. 결과는 입력 벡터와 동일한 수의 요소를 가진 벡터가 됩니다. 따라서 더 많은 요소를 보유할수록 코드의 속도가 빨라집니다. 벡터에 포함할 수 있는 요소의 크기는 아키텍처에 따라 다릅니다. CPU에는 **SSE**라는 것이 있습니다: "스트리밍 SIMD 확장"입니다. 이것은 SIMD 연산을 수행할 수 있는 일련의 명령어입니다. 보유할 수 있는 벡터의 크기는 CPU가 지원하는 SSE 버전에 따라 다릅니다. 예를 들어, CPU가 SSE2를 지원하는 경우 벡터에 두 개의 요소를 보유할 수 있습니다. CPU가 SSE4.1을 지원하는 경우 벡터에 4개의 요소를 담을 수 있습니다. 등등. 제가 얼마 전에 작성한 글에서도 이에 대해 좀 더 자세히 설명했습니다: ["목록 합계의 예에서 C#에서 SSE 사용하기"](https://steven-giesel.com/blogPost/d80d9367-3a1f-407f-9bdb-067fae9ea527).

## 제네릭 연산과 SIMD

C# 10의 도입으로 이제 "제네릭 연산"이라는 새로운 기능이 추가되었습니다. 이를 통해 모든 숫자 타입에 SIMD 연산을 사용할 수 있습니다. 이제 각 타입에 대한 함수를 만들지 않고도 `int`, `float`, `double` 등에 SIMD 연산을 사용할 수 있기 때문에 매우 유용합니다. 또한 사용자 정의 타입에 SIMD 연산을 사용할 수 있다는 점도 좋습니다. 어떻게 작동하는지 살펴보겠습니다.

## `Min` 함수 만들기

SIMD에서는 데이터가 가능한 한 독립적인 것이 중요합니다. 문제를 더 작은 하위 문제로 나누고 그 하위 문제를 한 번의 연산으로 해결한다는 개념입니다. 따라서 4개의 요소로 구성된 벡터가 있는 경우 각 요소에 대해 동시에 동일한 연산을 수행하려고 합니다.

이 경우 배열에서 가장 작은 값을 검색해야 하는 `Min` 함수가 있습니다. 이를 위해 전체 배열을 작은 덩어리로 분할하고 각각에 대해 가장 작은 값이 무엇인지 확인합니다.

```csharp
public static T Min<T>(this Span<T> span)
    where T : unmanaged, IMinMaxValue<T>, INumber<T>
```

단순화를 위해 `Span<T>`를 사용합니다. 중요한 부분은 메모리 블록이 임의의 메모리 주소를 처리할 수 없는 SIMD 프로세서로 스트리밍되기 때문에 연속적인 메모리가 필요하다는 것입니다. `INumber<T>`는 타입이 숫자라는 것을 알려주는 마커 인터페이스입니다. `IMinMaxValue<T>`는 타입에 최소값과 최대값이 있음을 알려주는 마커 인터페이스입니다. 이는 타입의 최대값으로 결과를 초기화해야 하므로 중요합니다. `unmanaged`는 타입이 값 타입이고 타입에 참조 타입이 포함되어 있지 않음을 알려주는 제약 조건입니다. 이는 타입을 벡터에 저장할 수 있어야 하기 때문에 중요합니다. 엄밀히 말하면 `Vector<T>`에는 `구조체` 제약 조건만 있지만 `구조체`에는 참조 타입이 포함될 수 있으므로 `관리되지 않는` 제약 조건을 추가해야 합니다. 그래야 `stackalloc`과 같은 것을 사용해 참조 타입을 가진 구조체가 예기치 않은 결과를 낳는 시나리오를 방지할 수 있습니다.

함수 자체는 매우 간단합니다. 먼저 해당 타입의 최대값을 가진 벡터를 생성합니다. 여기서 제네릭 연산이 실제로 시작됩니다! 또한 `Span`을 `Vector`로 직접 캐스팅합니다.

```csharp
var spanAsVectors = MemoryMarshal.Cast<T, Vector<T>>(span);
Span<T> vector = stackalloc T[Vector<T>.Count];
vector.Fill(T.MaxValue);
var minVector = new Vector<T>(vector);
```

이 경우 나중에 최소값을 검색하고 싶기 때문에 `T.MaxValue`를 사용하고 있습니다. `T.MinValue`나 `T.Zero`를 사용하면 가장 작은 값이 존재할 수 있으므로 잘못된 결과가 나올 수 있다는 문제가 발생합니다. 그런데 `T.MinValue`나 `T.Zero`와 같은 것은 제네릭 연산을 통해 가능합니다.

다음 단계는 각 벡터 항목을 다른 모든 벡터와 비교하여 최소값을 검색하는 것입니다.

```csharp
foreach (var spanAsVector in spanAsVectors)
{
    minVector = Vector.Min(spanAsVector, minVector);
}
```

결국 n개의 엔트리가 있는 벡터를 갖게 되고, 그 중 하나가 총 최소값이 됩니다. 하지만 아직 끝나지 않았습니다. `MemoryMarshal.Cast<T, Vector<T>>(span);` 에는 큰 단점이 있습니다: `Vector<T>.Length`가 4이고 입력이 9개의 항목이 있는 목록이라고 가정해 봅시다. 이 함수는 2개의 벡터만 반환합니다. 따라서 결국 하나의 요소가 누락됩니다. 따라서 "남은 요소"가 있다면 지금까지 가지고 있는 벡터와 비교해야 합니다. 이 작업은 다음 코드를 통해 수행됩니다.

```csharp
var remainingElements = spanAsVectors.Length % Vector<T>.Count;
if (remainingElements > 0)
{
    Span<T> lastVectorElements = stackalloc T[Vector<T>.Count];
    lastVectorElements.Fill(T.MaxValue);
    span[^remainingElements..].CopyTo(lastVectorElements);
    minVector = Vector.Min(minVector, new Vector<T>(lastVectorElements));
}
```

이제 마지막 부분입니다: 벡터에서 최소값을 검색해야 합니다.

```csharp
var minValue = T.MaxValue;
for (var i = 0; i < Vector<T>.Count; i++)
{
    minValue = T.Min(minValue, minVector[i]);
}

return minValue;
```

여기까지입니다! `Min` 함수의 SIMD 버전을 성공적으로 구현했습니다. 이제 그 성능을 확인해 봅시다.

## 성능

성능을 측정하기 위해 [BenchmarkDotNet](https://benchmarkdotnet.org/) 라이브러리를 사용합니다. `Min` 함수의 SIMD 버전의 성능과 비 SIMD(LINQ) 버전의 `Min` 함수의 성능을 비교하겠습니다. 다음은 설정 코드입니다.

```csharp
public class MinBenchmark
{
    private readonly int[] _numbers = Enumerable.Range(0, 1000).ToArray();

    [Benchmark(Baseline = true)]
    public int LinqSum() => Enumerable.Min(_numbers);

    [Benchmark]
    public int LinqSIMDSum() => LinqSIMDExtensions.Min(_numbers);
}
```

다음은 제 MacBook M1의 결과입니다.

```
|      Method |     Mean |   Error |  StdDev | Ratio |
|------------ |---------:|--------:|--------:|------:|
|     LinqMin | 166.9 ns | 0.94 ns | 0.78 ns |  1.00 |
| LinqSIMDMin | 125.3 ns | 0.69 ns | 0.61 ns |  0.75 |
```

나쁘지 않네요! LINQ 버전보다 25% 더 빠릅니다. 이것은 매우 간단한 예시라는 점을 명심하세요.

## 더 많은 사용 사례 - 평균

제네릭 연산 덕분에 사용자 정의 타입에 대해 `합계` 또는 `평균`과 같은 작업을 수행할 수도 있습니다. 어쨌든 블로그 게시물 마지막에 전체 라이브러리와 코드 샘플을 링크하겠습니다. `Sum`은 `Min` 메서드와 거의 비슷하므로 여기서는 보여드리지 않겠습니다. 하지만 조금 특별한 것은 `평균`일 수 있습니다. 다음은 코드입니다.

```csharp
public static T Average<T>(this Span<T> span)
    where T : unmanaged, INumberBase<T>, IDivisionOperators<T, T, T>
{
    var length = T.CreateChecked(span.Length);
    var divisionOperators = span.Sum() / length;
    return divisionOperators;
}
```

다음은 몇 가지 핵심 사항입니다. `IDivisionOperators<T, T, T>` 인터페이스를 사용하고 있습니다. 이 인터페이스는 타입에 나눗셈 연산자가 있음을 알려주는 마커 인터페이스입니다. 따라서 일반 합계를 구하고 이를 목록에 있는 항목의 양으로 나눕니다. `span.Length`는 단순한 정수이므로 목록의 타입으로 변환해야 합니다. 이 작업은 `T.CreateChecked` 메서드를 통해 수행됩니다. 이 메서드는 다른 숫자와 유사한 타입을 받아 체크된 방식으로 우리 타입으로 변환하려고 시도합니다. 즉, 변환으로 인해 오버플로가 발생하면 예외가 발생합니다. 이는 오버플로를 조용히 무시하고 싶지 않기 때문에 중요합니다. 제네릭 연산에서 `/`는 동일한 타입에 대해서만 유효하기 때문에 이렇게 하는 것입니다. 따라서 `int`를 `int`로 나누고 결과적으로 `double`을 얻을 수 없습니다. 여기에는 큰 함정이 있습니다: 정수만 사용하고 평균을 검색하면 결과적으로 정수를 얻게 됩니다. 따라서 `[1, 2]`의 `평균`은 1입니다.

## 성능

이전과 마찬가지로 벤치마크를 생성하고 기존 LINQ 방법과의 차이점을 확인합니다.

```csharp
public class AverageBenchmark
{
    private readonly float[] _numbers = Enumerable.Range(0, 1000).Select(f => (float)f).ToArray();

    [Benchmark(Baseline = true)]
    public float LinqAverage() => Enumerable.Average(_numbers);

    [Benchmark]
    public float LinqSIMDAverage() => LinqSIMDExtensions.Average(_numbers);
}
```

결과:
```
|          Method |       Mean |   Error |  StdDev | Ratio |
|---------------- |-----------:|--------:|--------:|------:|
|     LinqAverage | 1,055.2 ns | 4.79 ns | 4.48 ns |  1.00 |
| LinqSIMDAverage |   179.9 ns | 1.00 ns | 0.94 ns |  0.17 |
```

그래요. 이것은 큰 차이입니다. LINQ 버전보다 거의 6배 더 빠릅니다.

## 결론

이제 제네릭 연산을 도입하여 코드를 채택하지 않고도 사용자 정의 타입과 더 광범위한 타입에 대해 SIMD 연산을 수행할 수 있게 되었습니다. 이는 .NET 생태계에 큰 진전입니다. 여기서 보여드린 예제를 기반으로 작은 라이브러리를 만들었습니다: [LinqSIMDExtensions](https://github.com/linkdotnet/LinqSIMDExtensions)라는 작은 라이브러리를 만들었는데, 이 라이브러리에는 `최소값`과 `평균` 외에 사용할 수 있는 몇 가지 메서드가 더 있습니다. 사용 설명서에서 nuget 패키지 다운로드도 찾을 수 있습니다.

## 자료

- 소스 코드와 라이브러리는 여기에서 찾을 수 있음: [LinqSIMDExtensions](https://github.com/linkdotnet/LinqSIMDExtensions)
- nuget 다운로드는 여기에서 찾을 수 있음: [LinqSIMDExtensions](https://www.nuget.org/packages/LinkDotNet.LinqSIMDExtensions/)
