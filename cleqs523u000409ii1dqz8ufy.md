# 대량의 목록에서 다수의 삭제가 필요한 경우 최적화

다음의 코드로 예시를 들어보겠습니다.

천만 건의 목록에서 다수의 항목이 삭제가 될 때 생각보다 성능이 좋지 않습니다. `List<T>`의 구현이 항목 조회 및 추가에 최적화 되어 있기 때문입니다.

```csharp
using System.Diagnostics;

var list = Enumerable.Range(1, 200000).ToList();

Console.WriteLine(list.Count);


var sw = Stopwatch.StartNew();

var index = 0;
foreach (var item in list.ToArray())
{
    if (item % 3 is 0)
    {
        list.RemoveAt(index);
        index--;
    }

    index++;
}

Console.WriteLine($"{sw.ElapsedMilliseconds} ms");

Console.WriteLine(list.Count);
```

| 출력

```csharp
200000
5692 ms
133334
```

단지 20만 건 임에도 불구하고 제 컴퓨터에서 5초가 넘게 소요되었습니다.

`List<T>`의 내부 구현은 배열로 되어 있는데 항목 삭제 시 올바른 배열 인덱싱을 위해 배열 복사를 하기 때문입니다.

다음의 코드를 통해 처리 시간을 크게 개선할 수 있습니다. (다만 항목의 순서가 바뀝니다. 항목의 순서가 중요하지 않을 경우 사용할 수 있습니다)

```csharp
using System.Diagnostics;

var list = Enumerable.Range(1, 200000).ToList();

Console.WriteLine(list.Count);


var sw = Stopwatch.StartNew();

var index = 0;
foreach (var item in list.ToArray())
{
    if (item % 3 is 0)
    {
        list.RemoveUnorderedAt(index);
        index--;
    }
    index++;
}

Console.WriteLine($"{sw.ElapsedMilliseconds} ms");

Console.WriteLine(list.Count);


public static class ListExtensions
{
    public static void RemoveUnorderedAt<T>(this List<T> list, int index)
    {
        list[index] = list[^1];
        list.RemoveAt(list.Count - 1);
    }
}
```

| 출력

```csharp
200000
1 ms
133334
```

이렇게 빨라진 이유는 마지막 항목을 삭제 항목 위치로 옮기고 마지막 항목을 삭제하므로 `List<T>`의 구현 상 배열 복사가 필요 없어지기 때문입니다.