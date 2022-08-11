## Rust #4: 4장 소유권의 이해

## 개요
Rust는 참조형 인스턴스의 소유권이라는 독특한 기능을 가지고 있습니다. 이것을 통해 Rust는 가비지 수집기 없이도 안전하게 메모리에 인스턴스를 할당하거나 해제할 수 있게 됩니다. 이글은 [4장. 소유권 이해](https://doc.rust-lang.org/nightly/book/ch04-00-understanding-ownership.html)의 소스코드를 따라 하면서 Rust의 `소유권`을 이해하고 손에 익는데 목적이 있습니다.

## 소유권이란 무엇인가요?
소유권은 Rust의 핵심 기능입니다. 모든 프로그램은 프로그램이 실행되는 동안 메모리를 사용하는 방식을 관리해야 합니다. Java나 C# 등의 가비지 수집기를 가진 언어는 프로그래머가 할당만 제대로 해주면 수집은 가비지수집기에서 알아서 해주기 때문에 메모리 관리가 상대적으로 쉽습니다. C나 C++의 경우 프로그래머가 명시적으로 메모리를 할당하고 해제해야 하며, 메모리를 중복 할당하거나 중복 해제 또는 해제하지 않았을 때 발생하는 다양한 버그가 생길 여지가 있습니다. Rust에서는 소유권이라는 메모리 할당 및 해제를 런타임이 아니라 컴파일 타임에 `소유권`시스템을 통해 추적 관리하고 이를 통해 메모리 할당 및 해제를 안전하게 처리합니다.
소유권이란 개체를 생성한 후 그 개체를 접근하거나(Read) 변경하는(Write) 권한을 가진 것을 소유권이라 하며 오직 하나의 변수만 그 소유권을 가집니다.

### 소유권 규칙
Rust는 다음의 소유권 규칙을 가집니다.
- Rust의 각 값에는 소유자라는 변수가 있습니다.
- 한 번에 하나의 소유자만 있을 수 있습니다.
- 소유자가 범위(중괄호 블럭)를 벗어나면 값이 삭제 됩니다.

### 가변 범위

변수가 할당된 범위를 벗어나면 그 변수를 사용할 수 없습니다. 이 원칙은 다른 프로그래밍 언어와 유사 합니다.

```rust
fn main() {
    {
        let s = "hello";
    }
    println!("{}", s);
}
```

변수 s는 s가 할당된 범위에서 벗어났으므로 `println!()`에서 사용할 수 없습니다.

### String 유형
문자열 리터럴은 변경할 수 없습니다. 변경 가능한 문자열을 처리하기 위해 String이 필요한데요, String은 값을 변경할 수 있으므로 소유권이 적용됩니다. 문자열 리터럴에서 다음과 같이 String을 할당할 수 있습니다.

```rust
fn main() {
    let s = String::from("hello");
    println!("{}", s);
}
```

변수 `s`를 `mut`키워드를 써서 변경가능한 변수로 할당할 수 있습니다.

```rust
fn main() {
    let mut s = String::from("hello");
    s.push_str(", world!");
    println!("{}", s);
}
```

### 메모리 및 할당
문자열 리터럴의 경우 컴파일 타임에 내용을 알고 있으므로 문자열이 최종 실행 파일에 하드코딩 됩니다. 문자열이 변경될 일이 없다면 이때문에 문자열 리터럴이 빠르고 효율적인 이유인데요, 문자열이 런타임시 변경될 수 있는 경우 이것을 사용할 수 없습니다.
String 유형을 사용하면 런타임 때 문자열을 변경하기 때문에 힙 메모리를 사용해야 합니다. 이것은 다음을 의미하는데요,

- 런타임 시 메모리 할당자에게 메모리를 요청해야 합니다.
- 할당자에게 다시 메모리를 반환할 방법이 필요합니다.

첫번째는 일반적으로 프로그래머가 합니다. 바로 `String:from()` 메소드를 통해서인데요, 이것은 프로그래밍 언어에서 거의 보편적인 방법입니다.

하지만 두번째는 프로그래밍 언어에 따라 다양한 전략이 있습니다. 프로그래머가 직접 할당자에게 메모리를 해제하라고 그때그때 지시하거나 더이상 할당한 메모리를 참조하지 않으면 가비지 콜렉터에 의해 이것을 처리하거나 Rust처럼 소유권 규칙을 통해 메모리에 대한 소유자가 더이상 존재하지 않을때 메모리를 해제 합니다.

```rust
fn main() {
    {
        let mut s = String::from("hello");
        s.push_str(", world!");
    }
    println!("{}", s);
}```

동일한 가변 범위의 규칙을 따라, `Hello, World!` String 문자열이 범위를 벗어났으므로 소유자가 더이상 없게 되고 해당 메모리를 해제 됩니다. 

#### 변수와 데이터가 상호 작동하는 방식 : 이동

다음의 코드를 봅시다.
```rust
fn main() {
    let x = 5;
    let y = x;

    println!("{}", y);
}
```
스칼라 형과 배열, 튜플 모두 스택에 할당되고 `복사`가 됩니다. 여기서는 소유권에 대한 규칙이 적용되지 않는데요, String은 힙 메모리를 할당하고 '이동'이 됩니다. 다음의 코드를 보시죠.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;

    println!("{}", s1);
}
```
다른 프로그래밍 언어에서는 오류가 나지 않아야 합니다. `s2`는 `s1`을 단순히 참조할 뿐인데요, Rust에는 다릅니다. `let s2 = s1`에 의해서 String 인스턴스에 대한 소유권이 `s1`에서 `s2`로 이동했습니다. 즉, `println!("{}", s1)에서 `s1`는 소유권이 없으므로 에러입니다. 심지어 컴파일 에러가 발생합니다!

그렇다면 값을 복사하고 싶을때는 어떻게 해야 할까요? 다음의 코드를 보시죠.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();

    println!("s1 = {}, s2 = {}", s1, s2);
}
```

`clone()` 함수에 의해 String 복사 함수를 통해 새로운 힙 메모리를 할당하여 값을 복사했습니다. 즉, `s1`과 `s2`는 동일한 문자열을 가지고 있으나 다른 메모리 위치를 가지게 되어 각각 소유자를 가질 수 있게 되었는데요, 두 코드의 차이를 바로 이해하지 못하는 분도 있을 수 있는데 이를 개체 인스턴스로 바라보면 쉽게 이해됩니다. 첫번째 코드는 개체 인스턴스가 하나고요, 아래 코드는 개체 인스턴스가 두개입니다. 즉, 각각 소유자가 하나, 두개가 될 수 있습니다.

#### 스택 전용 데이터 : 복사
스택은 빠르게 접근할 수 있고 임시 메모리이기 떄문에 `복사`가 유리합니다. 힙은 해제하지 않는 한 프로그램이 종료될 때까지 유지될 수 있으므로 `이동`이 유리합니다. 그러므로 스택에 할당이 되는 스칼라 형이나 튜플, 메모리의 경우 `복사`가 됩니다. String처럼 힙에 할당되는 것은 `이동`이 됩니다. 소유권 규칙은 `이동`에만 해당하는 규칙입니다.

### 소유권과 함수
String을 함수 인자로 전달하면 어떻게 될까요? 동일한 `이동`규칙이 적용됩니다. 즉, 인자로 받은 함수가 소유권을 가져가게 되는데요, 다음의 코드를 통해 확인해봅시다.

```rust
fn main() {
    let s = String::from("hello");

    takes_ownership(s);

    let x = 5;

    makes_copy(x);
}

fn takes_ownership(some_string: String) {
    println!("{}", &some_string);
}

fn makes_copy(some_integer: i32) {
    println!("{}", some_integer);
}
```

`s`는 힙 할당이므로 이동이되어 함수가 소유권을 가져가고, `x`는 스택 할당이므로 복사가 되어 함수에 값이 복사되어 전달됩니다.
다음으로 동작의 차이를 비교하여 확인할 수 있습니다.

```rust
fn main() {
    let s = String::from("hello");

    takes_ownership(s);

    // takes_ownership()가 소유권을 가져갔으므로 오류!
    println!("{}", s);

    let x = 5;

    makes_copy(x);

    // makes_copy()가 소유권을 가져가 않았으므로(값은 소유권이 없으므로) 정상!
    println!("{}", x);
}

fn takes_ownership(some_string: String) {
    println!("{}", &some_string);
}

fn makes_copy(some_integer: i32) {
    println!("{}", some_integer);
}
```

### 반환 값 및 범위
그렇다면 함수 처리 후 다시 소유권을 가져가려면 어떻게 해야 할까요? 다음의 코드를 통해 확인해봅시다.

```rust
fn main() {
    let s1 = gives_ownership();
    let s2 = String::from("hello");
    let s3 = takes_and_gives_back(s2);
}

fn gives_ownership() -> String {
    let some_string = String::from("hello");

    some_string
}

fn takes_and_gives_back(a_string: String) -> String {
    a_string
}
```

잘 동작합니다. 하지만 이렇게 소유권을 주거니 받거니 하는게 과연 편리한 코딩일까요? 프로그래밍 경험이 아직 없는 학생이라면 `아 그렇구나` 하겠지만 프로그래밍 경험이 이미 있는 프로그래머라면 위의 소스 동작을 확인하면서 아득히 답답함이 밀려올 것이라는 것을 예상할 수 있습니다. 

```rust
fn main() {
    let s1 = String::from("hello");
    let (s1, len) = calculate_length(s1);
    print!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len();

    (s, length)
}
```

변수 쉐도우와 튜플을 이용해서 다시 소유권을 가져왔지만, 번거롭습니다. 그리고 함수 인자가 하나가 아니라 여러개라면 비효율적인 코딩이 될 것 같습니다. 하지만 Rust는 `참조`라는 개념을 통해 이를 해결하는데요. 한번 살펴봅시다.

## 참조 및 차용
Rust에서는 `참조`를 이용해 다음처럼 코드를 개선할 수 있습니다.

```rust
fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1);
    print!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    let length = s.len();

    length
}
```

참조는 소유권을 가져가지 않고 단지 참조한다는 의미로 `&`를 붙여서 사용합니다.  아래의 도식처럼 접근하게 되는데요,

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1626829504636/KrR6gZErt.png)

다음 코드를 통해 참조의 특징을 파악할 수 있습니다.

```rust
fn main() {
    let s = String::from("hello");

    change(&s);
}

fn change(some_string: &String) {
    some_string.push_str(", world!");
}
```

이 코드는 `some_string.push_str()`함수에서 오류가 발생합니다. 즉, 참조를 통해서는 값을 수정할 수 없습니다. Rust 변수가 기본적으로 불변인 것처럼 참조도 마찬가지라는 것을 알 수 있습니다.

### 변경 가능한 참조
변수의 그것 처럼 참조도 `&mut`으로 변경가능한 참조를 사용할 수 있습니다.

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world!");
}
```

위의 코드는 정상적으로 동작하며, `s`변수의 값을 변경합니다. 하지만 `&mut`는 강력한 제한이 있는데요, 동일한 범위에서 하나의 변경 참조만 허용한다는 것입니다. 다음의 코드는 컴파일 오류가 발생합니다.

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;
    // 두번째 mutable이 발생함으로 오류
    let r2 = &mut s;

    println!("{}, {}", r1, r2);
}
```

이런 제한은 Rust 코딩을 어렵게 하지만 하나의 값에 두개 이상의 코드에서 접근할 때 발생하는 데이터 레이스 경쟁을 해소하는 강력한 방법입니다.

- 두개 이상의 포인터가 동시에 동일한 값에 액세스 합니다.
- 포인터 중 하나 이상이 쓰는데 사용됩니다.
- 값에 대한 액세스를 동기화 하는데 사용되는 메커니즘이 없습니다.

Rust는 위의 세가지 행동을 컴파일 시점에서 원천적으로 막고 있다는 것을 알수 있습니다. 익숙해지면 매우 강력한 메모리 안전 코딩이 가능해진다는 것을 느낄 수 있군요!

다음의 코드는 동일한 범위가 아니므로 에러가 발생하지 않는다는 것을 알 수 있습니다.

```rust
fn main() {
    let mut s = String::from("hello");

    {
        let r1 = &mut s;
    }
    let r2 = &mut s;
}
```

다른 규칙도 있습니다. 다음의 코드로 확인해보죠!

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    let r3 = &mut s;

    println!("{}, {}, and {}", r1, r2, r3);
}
```

참조가 하나 이상 있고 변경가능한 참조를 할 수 없다는 규칙인데요, 다음의 코드도 마찬가지 에러가 발생합니다.

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;

    s.push_str(", world!");

    println!("{}, {}", s, r1);
}
```

참조된 값이 변경될 수 없다는 에러입니다. 이 에러를 해결하려면 다음의 코드 흐름을 가져야 합니다.

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;

    println!("{}", r1);

    s.push_str(", world!");

    println!("{}", s);
}
```

### 댕글링 참조
해제된 메모리를 가리키는 포인터를 다른곳에서 사용하는 것을 댕글링 참조라고 하는데요, 빈번하게 발생하는 메모리 관련 버그입니다. Rust에서 댕글링 참조가 일어날 수 있는지 다음의 코드를 통해 확인해봅시다.

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {
    let s = String::from("hello");

    &s
}
```

위의 코드가 모든 댕글링 참조의 모습을 보여주지는 않지만 실제로 Rust는 댕글링 참조를 컴파일 타임에서 오류로 처리하므로 메모리 관련 버그를 사전에 차단하고 있습니다.

올바른 Rust 코드는 다음과 같습니다.

```rust
fn main() {
    let s = no_dangle();
}

fn no_dangle() -> String {
    let s = String::from("hello");

    s
}
```

소유권 이전에 의해 아무것도 할당되거나 해제되지 않고 `s`변수가 소유권을 가지게 되었습니다.

### 참조 규칙
다음의 참조에 대해 요약해볼 수 있습니다.

- 주어진 시간에 하나의 변경 가능한 참조 또는 여러개의 불변 참조를 할 수 있습니다.
- 참조는 항상 유효해야 합니다.

소유권과 참조의 이런 규칙에 의해 Rust는 매우 강력한 메모리 안정성을 보장하게 됩니다.

다음으로 슬라이스에 대해 살펴봅시다.

## 슬라이스 유형
참조와 같이 소유권이 없는 또다른 유형은 슬라이스(slice) 입니다. 슬라이스를 사용하면 메모리를 추가 할당할 필요 없이 특정 컬렉션의 연속적인 요소 시퀀스를 참조할 수 있습니다. 슬라이스의 유용함을 이해하기 위해 다음의 코드를 살펴봅시다.

```rust
fn first_word(s: &String) -> ?
```

문장의 첫번째 단어를 반환하는 함수가 있다고 가정합니다. 그 함수의 반환형은 무엇이 되어야 할까요? 여러 프로그래밍 언어는 이것을 달성하기 위해 `복사`방식을 사용했습니다. 제가 익숙한 C#언어도 마찬가지인데요, C# 코드를 예로 들어서 다음과 같이 표현할 수 있습니다.

```csharp
string GetFirstWord(string sentence)
```
sentence를 받아 첫번째 단어를 찾아서 string으로 반환합니다. 그런데 여기서 문제가 있는데요, 이런 방식은 반드시 `복사`가 이루어져야 합니다. 힘 메모리의 복사는 상당한 비용이 발생하는데요, 그리고 C#도 연속한 시퀀스를 참조하는 `Span<T>`라는 것으로 이를 해결하고 있습니다.

rust는 슬라이스를 통해 이를 달성할 수 있는데요 코드를 보면서 확인해봅시다.

```rust
fn first_word(s: &String) -> usize {
    let bytes = s.as_bytes();

    for (i, &item) in byte.iter().enumerte() {
        if item == b' ' {
           return i;
        }
    }

s.len()
}
```
슬라이스를 사용하지 않고 복사하지 않는 버젼입니다. 정확히 첫번째 단어의 끝 위치를 반환하지만, 첫번째 단어를 반환하는 완전한 함수는 아닙니다. 

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s);

    s.clear();

    println!("{}", word);
}

fn first_word(s: &String) -> usize {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return i;
        }
    }

    s.len()
}
```

`s.clear()`에 의해 `s` 변수의 문자열은 초기화 되었습니다. 더이상 첫번째 단어의 끝 위치인 `5`라는 숫자는 의미가 없어졌는데요, 좋은 코드는 아닌 것 같습니다.

### 스트링 슬라이스

슬라이스를 이용해서 복사하지 않고 첫번째 단어를 반환하는 함수를 다음처럼 개선할 수 있습니다.

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s);

    println!("{}", word);

    s.clear();
}

fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

슬라이스를 통해 첫번째 단어를 반환하고 `println!()`을 통해 소비되었습니다. 이후 `s.clear()`에 의해 문자열이 초기화 되었지만 문제가 없는 코드이고 깔끔합니다. `println!()` 함수를 `s.clear()` 아래에다가 다시 써볼까요?

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s);

    println!("{}", word);

    // word 변수는 변하지 않는 대상을 참조해야 하는데 `s.clear()`에 의해 변하므로 오류입니다.
    s.clear();

    println!("{}", word);
}```

#### 문자열 리터럴은 슬라이스 입니다.
문자열 리터럴은 바이너리의 특정 지점을 가리키는 슬라이스입니다. 이것이 문자열 리터럴이 변경할 수 없는 이유이기도 합니다.

#### 매개변수로서의 문자열 슬라이스
리터럴과 String값의 조각을 사용할 수 있다는 것을 알면 first_word를 다음과 같이 개선할 수 있습니다.

```rust
fn first_word(s: &str) -> &str {
```

```rust
fn main() {
    let my_string = String::from("hello world");

    let word = first_word(&my_string[0..6]);
    let word = first_word(&my_string[..]);
    let word = first_word(&my_string);
    let my_string_literal = "hello world";

    let word = first_word(&my_string_literal[0..6]);
    let word = first_word(&my_string_literal[..]);

    let word = first_word(my_string_literal);
```

### 기타 슬라이스

배열도 슬라이스가 됩니다.

```rust
let a = [1, 2, 3, 4, 5];

let slice = &a[1..3];

assert_eq!(slice, &[2, 3]);
```

### 요약
소유권, 차용, 슬라이스의 개념은 Rust 프로그램의 메모리 안전성을 보장하기 위해 모두 컴파일 시간에 이루어집니다.
