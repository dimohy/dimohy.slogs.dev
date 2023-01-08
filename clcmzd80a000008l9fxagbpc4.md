# Use the "is" operator to use non-duplicated expressions

```csharp
var input = "15,2";
```

If you want to get the value with ',' you can usually get it with the following code:

```csharp
var items = input.Split(',');
var (a, b) = (items[0], items[1]);
```

You can also use LINQ to do something like this:

```csharp
(a, b) = input.Split(',').Chunk(2).Select(x => (x[0], x[1])).First();
```

However, you may want to use it directly in 'expressions' such as 'switch expressions' without duplicate assignments. In this case, you can use the 'is keyword' to do something like this:

```csharp
// True if the input value is 15 and not 3
var bResult = input.Split(',') is string[] items2 is true && items2[0] is "15" && items2[1] is not "3";
```