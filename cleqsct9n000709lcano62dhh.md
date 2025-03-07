---
title: "Optimizing for multiple deletions in a large list"
datePublished: Thu Mar 02 2023 07:29:06 GMT+0000 (Coordinated Universal Time)
cuid: cleqsct9n000709lcano62dhh
slug: optimizing-for-multiple-deletions-in-a-large-list
tags: csharp

---

The implementation of `List<T>` is optimized for looking up and adding items, so it doesn't perform as well as you might think when a large number of items are deleted from a list of hundreds of thousands.

Here's an example with the following code:

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

| Output

```csharp
200000
5692 ms
133334
```

On my machine, it took over 5 seconds, even though it was only 200,000.

Because the internal implementation of `List<T>` is an array, when an item is deleted, the array is copied internally for correct array indexing.

The following code can significantly improve processing time (but only if the order of the items doesn't matter, since deleting reverses the order of the items).

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

| Output

```csharp
200000
1 ms
133334
```

The reason for this speedup is that the implementation of `List<T>` moves the last item to the delete item location and deletes the last item, eliminating the need to copy the array.