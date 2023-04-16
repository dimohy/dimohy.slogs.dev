---
title: "문자열을 .NET 개체로 변환하기 - IParsable 및 ISpanParsable"
datePublished: Sun Apr 16 2023 03:42:11 GMT+0000 (Coordinated Universal Time)
cuid: clgiv2c3g000109kwd276cah3
slug: net-iparsable-ispanparsable
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1681618464298/2de13e33-c944-488c-9455-446f9c86f75b.jpeg
tags: csharp

---

> 본 글은 Christian Nagel님의 [Converting Strings to .NET Objects – IParsable and ISpanParsable](https://csharp.christiannagel.com/2023/04/14/iparsable/)을 번역한 것입니다.

---

C# 11의 새로운 기능으로 인터페이스 추상 정적 멤버를 사용할 수 있습니다. 이를 통해 + 및 - 연산자를 사용하는 등 제네릭 클래스 구현에서 계약으로 사용할 클래스 메서드를 정의할 수 있습니다. .NET 7에서는 숫자 유형이 많은 새로운 인터페이스를 구현합니다. 이 C# 11의 기능은 수학에만 국한된 것이 아닙니다! 새로운 `IParsable` 및 `ISpanParsable` 인터페이스를 사용하면 문자열에서 개체를 만들 수 있습니다. 이러한 인터페이스는 제네릭 타입의 제약 조건과 함께 사용할 수 있으므로 이제 제네릭 구현을 통해 문자열을 구문 분석하여 개체를 생성하는 작업이 쉬워졌습니다. 이 문서에서는 구문 분석 인터페이스의 문자열 버전과 스팬 버전을 모두 구현하고 이를 제네릭 타입과 함께 사용하는 방법을 보여줍니다.

![Parse strings to objects](https://csharpdotchristiannageldotcom.files.wordpress.com/2023/04/parsestringstoobjects.jpg align="left")

## IFormattable

새로운 것은 아니지만 문자열에서 구문 분석의 다른 측면인 개체를 문자열로 변환하는 인터페이스부터 시작하겠습니다. `IFormattable` 인터페이스는 .NET 초기부터 사용 가능했습니다. 구문 분석 가능 인터페이스와 달리 `IFormattable` 인터페이스에 정의된 멤버는 인스턴스 멤버입니다.

샘플 코드에서는 `partial` 키워드를 사용하여 `ColorResult` 레코드를 여러 파일로 분할합니다. 한 파일에는 다음 코드 스니펫에 표시된 대로 `ColorResult`의 멤버가 정의되어 있습니다. `ColorResult` 유형을 정의하는 다른 파일은 필요한 인터페이스를 구현합니다. `ColorResult`에는 두 바이트 값이 포함됩니다. 게임 유형에 따라 최대 개수는 다릅니다. 기존 게임 유형에서 이 값은 0에서 5 사이의 범위입니다.

```csharp
public readonly partial record struct ColorResult(byte Correct, byte WrongPosition);
```

다음 `IFormattable` 인터페이스 구현을 사용하면 레코드의 두 값을 모두 포함하는 문자열이 반환됩니다. `IFormattable` 인터페이스는 형식 문자열 및 문화권에 대한 매개 변수를 지정합니다. `ColorResult`를 구현하면 다른 형식 문자열이 사용되지 않으며 문화권에 따른 차이가 없으므로 매개 변수가 무시됩니다.

```csharp
public string ToString(string? format, IFormatProvider? formatProvider = default) => $"{Correct}:{WrongPosition}";
```

## ISpanFormattable

가비지 컬렉션이 필요한 새 개체를 만들지 않도록 하기 위해 .NET 6부터 `ISpanFormattable` 인터페이스를 사용할 수 있습니다. 이 인터페이스를 사용하면 개체의 문자열 표현을 `Span<char>`인스턴스에 쓸 수 있습니다. 다음 코드 스니펫은 `ISpanFormattable` 인터페이스의 구현을 보여줍니다. 스팬의 크기가 충분히 크지 않은 경우 `TryFormat` 구현은 false를 반환합니다. 그렇지 않으면 구분 기호로 구분된 숫자가 스팬에 기록됩니다.

```csharp
public bool TryFormat(Span<char> destination, out int charsWritten, ReadOnlySpan<char> format = default, IFormatProvider? provider = default)
{
   if (destination.Length < 3)
   {
      charsWritten = 0;
      return false;
   }

   destination[0] = (char)(Correct + '0');
   destination[1] = ':';
   destination[2] = (char)(WrongPosition + '0');
   charsWritten = 3;
   return true;
}
```

이제 `IFormattable` 인터페이스 구현이 필요한 크기의 문자 배열을 할당하여 스팬 버전을 사용하고 이 배열을 참조하는 `Span`을 `TryFormat` 메서드에 전달하도록 변경되었습니다.

```csharp
public string ToString(string? format, IFormatProvider? formatProvider = default)
{
   var destination = new char[3].AsSpan();
   if (TryFormat(destination, out _, format.AsSpan(), formatProvider))
   {
      return destination.ToString();
   }
   else
   {
      throw new FormatException();
   }
}
```

## IParsable 및 ISpanParsable

문자열 표현에서 새 개체를 만들려면 .NET 7에서 `IParsable` 및 `ISpanParsable` 인터페이스를 사용할 수 있습니다. 이러한 인터페이스는 다음 코드 조각에 표시된 대로 문자열과 스팬을 제네릭 유형으로 변환하는 정적 추상 멤버를 정의합니다.

```csharp
public interface IParsable<TSelf> where TSelf : PArsable<TSelf>?
{
   static abstract TSelf Parse(string s, IFormatProvider? provider);

   static abstract bool TryParse(
      [NotNullWhen(true)] string? s,
      IFormatProvider? provider,
      [MaybeNullWhen(false)] out TSelf result);
}
```

```csharp
public interface ISpanParsable<TSelf> : IParsable<TSelf>
   where TSelf : ISpanParsable<TSelf>?
{
   static abstract TSelf Parse(ReadOnlySpan<char> s, IFormatProvider? provider);

   static abstract bool TryParse(
      ReadOnlySpan<char> s,
      IFormatProvider? provider,
      [MaybeNullWhen(false)] out TSelf result);
}
```

---

`NotNullWhen` 및 `MaybeNullWhen` 특성에 대해 궁금할 수 있습니다. 이러한 특성은 컴파일러에게 메서드가 개체가 널이 아닌 경우 참을 반환하고, 개체가 널인 경우 거짓을 반환한다는 것을 알려주는 데 사용됩니다. 컴파일러는 이 정보를 사용하여 널 검사를 피합니다. MaybeNullWhen 속성은 메서드가 거짓을 반환하는 경우 컴파일러에 개체가 널임을 알리는 데 사용됩니다.

---

## ISpanParsable 구현

`ToString` 메서드는 콜론으로 구분된 `ColorType`의 멤버가 포함된 문자열을 반환합니다. C# 10부터 패턴 일치와 함께 사용할 수 있는 **목록 패턴**을 다음 코드 스니펫과 함께 사용하면 패턴이 콜론으로 구분된 두 문자와 일치하면 새 `ColorField` 인스턴스가 생성되어 반환됩니다. 그렇지 않으면 메서드는 널을 반환합니다.

```csharp
public static bool TryParse(
   ReadOnlySpan<char> s,
   IFormatProvider? provider,
   [MaybeNullWhen(false)] out ColorResult result)
{
   result = s switch
   {
      { Length: > 3 or < 3 } => default,
      { var correct, ':', var wrongpos } => new ColorResult((byte)(correct - '0'), (byte)(wrongpos - '0'),
      _ => default,
   };

   return result != default;
}

public static ColorResult Parse(ReadOnlySpan<char> s, IFormatProvider? provider = default)
{
   if (TryParse(s, provider, out var result))
   {
      return result;
   }
   else
   {
      throw new FormatException($"Cannot parse {s}");
   }
}
```

## ISpanParsable 사용

이 구현을 사용하면 `IParsable` 인터페이스를 구현하는 모든 유형을 사용하도록 제네릭 메서드를 구현할 수 있습니다. 제네릭 확장 메서드 `AddMove`를 사용하면 `IParsable<TResult>`가 제네릭 유형 `TResult`로 구현되어야 한다는 제약 조건이 정의됩니다. 그런 다음 클래스 이름 `TResult`를 사용하여 `Parse` 메서드를 호출할 수 있습니다. 다음 코드 스니펫에서는 일부 테스트 문자열 값을 전달하여 구문 분석하고 새로운 `ColorField` 인스턴스를 생성합니다. 게임 구현에서는 이동과 게임 유형에 따라 올바른 문자열 값을 가져오는 알고리즘이 호출됩니다.

```csharp
public static TResult AddMove<TField, TResult>(
   this Game<TField, TResult> game,
   IEnumerable<TField> fields)
   where TResult: IParsable<TResult>
{
   int lastMove = 0;
   if (game._moves.Count > 0)
   {
      lastMove = game._moves.Last().MoveNumber;
   }

   // 코드 목록을 기반으로 샘플 값을 반환하는 더미 구현일 뿐입니다.
   TResult result = game switch
   {
      { GameType: Game6x4 } => TResult.Parse("2:0", default),
      { GameType: Game8x5 } => TResult.Parse("1:2", default),
      { GameType: Game5x5x4 } => TResult.Parse("1:1:0", default),
      { GameType : Game6x4Simple } => TResult.Parse("0:1:1:2", default),
      _ => default,
   } ?? throw new InvalidOperationException();

   // 이동 생성
   Move<TField, TResult> move = new(game.GameId, Guid.NewGuid(), ++lastMove)
   {
      Fields = fields.ToArray(),
      Results = result
   };

   game._moves.Add(move);
   return result;
}
```

## 기타 구현

기존 유형에 대해 `ISpanParsable`이 어떻게 구현되는지 궁금할 수 있습니다. 다음 코드 스니펫은 `IPAddress` 유형에 대한 구현을 보여줍니다. `TryParse` 메서드는 스팬을 `IPAddress` 인스턴스로 구문 분석하는 데 사용됩니다. 구현은 `IPAddressParser.Parse` 메서드를 사용합니다. 이 메서드의 구현은 스팬에 a `:`가 있는지 확인하고, 있다면 스팬을 IPv6 주소로 구문 분석합니다. 그렇지 않으면 스팬이 IPv4 주소로 구문 분석됩니다. 구문 분석에 실패하면 구현은 `null`이 반환되는 경우 Try 접두사가 있는 메서드가 사용되는지 확인하고, 그렇지 않으면 `FormatException`을 던집니다. IP 주소를 나타내는 긴 타입을 생성하여 IP 주소를 인스턴스화하는 데 사용되는 `IP4StringToAddress` 메서드의 구현은 다음 코드 스니펫에 나와 있습니다.

```csharp
private static unsafe bool TryParseIpv4(ReadOnlySpan<char> ipSpan, out long address)
{
   int end = ipSpan.Length;
   long tempAddr;

   fixed (char* ipStringPtr = &MemoryMarshal.GetReference(ipSpan))
   {
      tmpAddr = Ipv4AddressHelper.ParseNonCanonical(ipStringPtr, 0, ref end, notImplicitFile: true);
   }

   if (tempAddr != IPv4AddressHelper.Invalid && end == ipSpan.Length)
   {
      // IPv4AddressHelper.ParseNonCanonical은 호스트 순서대로 바이트를 반환합니다.
      // 네트워크 순서로 변환하고 성공을 반환합니다.
      address = (uint)IPAddress.HostToNetworkOrder(unchecked((int)tmpAddr));
      return true;
   }

   // 주소 파싱 실패 시
   address = 0;
   return false;
}
```

---

IPAddress 구문 분석 구현에서 볼 수 있듯이 C# 및 Span&lt;T&gt; 유형에 안전하지 않은 코드를 사용하면 유용하게 사용할 수 있습니다. 값이 다른 바이트에 기록되는 `ColorResult` 유형을 구현하면 안전하지 않은 코드를 사용할 필요가 없습니다. 스팬을 정수로 변환하는 `IPAddress` 유형과 같은 경우, 애플리케이션에서 많은 인스턴스를 변환할 때 성능을 향상시킬 수 있어 큰 이점이 될 수 있습니다.

---

## 정리

인스턴스를 생성하기 위한 제네릭 구현을 생성하는 것은 이제 `IParsable<T>` 및 `ISpanParsable<T>` 인터페이스를 사용하여 쉽게 수행할 수 있습니다. 반대로 개체를 문자열이나 스팬으로 변환하려면 `IFormattable` 및 `ISpanFormattable` 인터페이스를 사용하면 됩니다. `ISpanFormattable` 인터페이스는 .NET 6부터 사용할 수 있으며, `IParsable` 및 `ISpanParsable` 인터페이스는 .NET 7부터 사용할 수 있습니다.

학습과 프로그래밍을 즐기세요!

Christian

이 글이 마음에 드셨다면 커피 한 잔으로 응원해 주세요. 고마워요!

[![Buy Me A Coffee](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png align="left")](https://www.buymeacoffee.com/christiannagel)

---

[https://csharp.christiannagel.com/2023/04/14/iparsable/](https://csharp.christiannagel.com/2023/04/14/iparsable/)