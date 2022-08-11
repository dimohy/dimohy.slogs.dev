## Rust #18: 18장 패턴과 매칭

## 개요

패턴은 통해 복잡하거나 단순한 유형의 구조에 대해 일치 시키는 Rust의 특수 구문입니다. match 표현식 및 기타 구문과 함께 패턴을 사용하면 프로그램의 흐름을 좀 더 잘 제어할 수 있습니다. 패턴은 다음의 조합으로 구성됩니다.

- 리터럴
- 분해한 배열, 열거형, 구조체 또는 튜플
- 변수
- 와일드카드
- 자리표시자


## 모든 장소 패턴을 사용할 수 있습니다.

패턴은 Rust의 여러 요소에서 사용할 수 있습니다. 패턴의 유효한 모든 위치에서 패턴에 대해 설명합니다.


### match

6장에서 논의한 바와 같이 `match` 표현식에서 패턴을 사용할 수 있습니다.

```rust
match VALUE {
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
}
```
`match`은 모든 가능성이 표현되어야 하는 규칙이 있습니다. 그러므로 대부분 마지막엔 변수에 할당되지 않은 `_`을 사용하게 됩니다.


### 조건부 if let 표현식

6장에서 하나의 경우에 대한 일치를 편리하게 사용하기 위한 `if let`을 사용했습니다.

```chsarp
fn main() {
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Some(color) = favorite_color {
        println!("Using your favorite color, {}, as the background", color);
    } else if is_tuesday {
        println!("Tuesday is green day!");
    } else if let Ok(age) = age {
        if age > 30 {
            println!("Using purple as the background color");
        } else {
            println!("Using orange as the background color");
        }
    } else {
        println!("Using blue as the background color");
    }
}
```

### while let 조건부 루프

`if let`과 구조가 유사한 `while let`에서 조건부 반복문에 패턴을 사용할 수 있습니다.

```rust
    let mut stack = Vec::new();

    stack.push(1);
    stack.push(2);
    stack.push(3);

    while let Some(top) = stack.pop() {
        println!("{}", top);
    }
```

### for 루프

`for` 루프에서 가령 튜플을 사용하기 위해서 패턴이 사용됩니다.

```rust
    let v = vec!['a', 'b', 'c'];

    for (index, value) in v.iter().enumerate() {
        println!("{} is at index {}", value, index);
    }
```

```shell
$ cargo run
   Compiling patterns v0.1.0 (file:///projects/patterns)
    Finished dev [unoptimized + debuginfo] target(s) in 0.52s
     Running `target/debug/patterns`
a is at index 0
b is at index 1
c is at index 2
```

### let 문

사실 `let` 문도 패턴을 사용 합니다.

```rust
let PATTERN = EXPRESSION;
```

여기서 일반적으로 사용하는 패턴은 튜플 분해 입니다.

```rust
let (x, y, z) = (1, 2, 3);
```

이때 양쪽의 각 항목의 개수가 동일해야 오류가 발생하지 않습니다.

```rust
    let (x, y) = (1, 2, 3);
```

```shell
$ cargo run
   Compiling patterns v0.1.0 (file:///projects/patterns)
error[E0308]: mismatched types
 --> src/main.rs:2:9
  |
2 |     let (x, y) = (1, 2, 3);
  |         ^^^^^^   --------- this expression has type `({integer}, {integer}, {integer})`
  |         |
  |         expected a tuple with 3 elements, found one with 2 elements
  |
  = note: expected tuple `({integer}, {integer}, {integer})`
             found tuple `(_, _)`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `patterns` due to previous error
```

### 함수 인자

함수 인자도 패턴일 수 있습니다. 다음의 사용을 살펴봅시다.

```rust
fn foo(x: i32) {
    // code goes here
}
```

패턴을 사용하지 않은 것처럼 보이지만 사실은 패턴입니다. x를 다음과 같이 변형해봅시다.

```rust
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({}, {})", x, y);
}

fn main() {
    let point = (3, 5);
    print_coordinates(&point);
}
```


## 반박 가능성: 패턴이 일치하지 않을 수 있는지 여부

패턴은 반박 가능한 것과 불가능 한 것으로 구분할 수 있습니다.

```rust
let Some(x) = some_option_value;
```

`let`문은 반박 불가만 허용하기 때문에 오류 입니다.

```shell
$ cargo run
   Compiling patterns v0.1.0 (file:///projects/patterns)
error[E0005]: refutable pattern in local binding: `None` not covered
   --> src/main.rs:3:9
    |
3   |     let Some(x) = some_option_value;
    |         ^^^^^^^ pattern `None` not covered
    |
    = note: `let` bindings require an "irrefutable pattern", like a `struct` or an `enum` with only one variant
    = note: for more information, visit https://doc.rust-lang.org/book/ch18-02-refutability.html
    = note: the matched value is of type `Option<i32>`
help: you might want to use `if let` to ignore the variant that isn't matched
    |
3   |     if let Some(x) = some_option_value { /* */ }
    |

For more information about this error, try `rustc --explain E0005`.
error: could not compile `patterns` due to previous error
```

하지만 다음의 `if let`문의 `Some(x)`는 반박 가능하기 때문에 정상 컴파일 됩니다.

```rust
    if let Some(x) = some_option_value {
        println!("{}", x);
    }
```

하지만 다음은 `x`는 반박 불가능 하기 때문에 컴파일 오류입니다.

```rust
    if let x = 5 {
        println!("{}", x);
    };
```

```shell
$ cargo run
   Compiling patterns v0.1.0 (file:///projects/patterns)
warning: irrefutable `if let` pattern
 --> src/main.rs:2:5
  |
2 | /     if let x = 5 {
3 | |         println!("{}", x);
4 | |     };
  | |_____^
  |
  = note: `#[warn(irrefutable_let_patterns)]` on by default
  = note: this pattern will always match, so the `if let` is useless
  = help: consider replacing the `if let` with a `let`

warning: `patterns` (bin "patterns") generated 1 warning
    Finished dev [unoptimized + debuginfo] target(s) in 0.39s
     Running `target/debug/patterns`
5
```

## 패턴 구문
이 섹션에서 패턴의 유효한 구문을 살펴보도록 합시다

### 일치하는 리터럴

다음은 리터럴 패턴입니다.

```rust
    let x = 1;

    match x {
        1 => println!("one"),
        2 => println!("two"),
        3 => println!("three"),
        _ => println!("anything"),
    }
```

### 명명된 변수 일치

다음은 

```rust
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(y) => println!("Matched, y = {:?}", y),
        _ => println!("Default case, x = {:?}", x),
    }

    println!("at the end: x = {:?}, y = {:?}", x, y);
```

여기서 문제는 `match` 내부에서 y를 사용하기 때문에 밖의 `y` 값을 가립니다. 이후 블럭에서 빠져나오면 기존의 y가 출력되게 됩니다. 이것을 해결하려면 추가 조건을 사용해야 합니다.

### 다중 패턴

`1 | 2`로 `1또는 2의 의미`로 다중 패턴을 사용할 수 있습니다.

```rust
    let x = 1;

    match x {
        1 | 2 => println!("one or two"),
        3 => println!("three"),
        _ => println!("anything"),
    }
```

### 값 범위 일치 ..=
`..=`를 이용해 값의 범위를 표현할 수 도 있습니다.

```rust
    let x = 5;

    match x {
        1..=5 => println!("one through five"),
        _ => println!("something else"),
    }
```

다음은 char 값 범위를 표현 합니다.

```rust
    let x = 'c';

    match x {
        'a'..='j' => println!("early ASCII letter"),
        'k'..='z' => println!("late ASCII letter"),
        _ => println!("something else"),
    }
```

### 구조체를 부분으로 분해

#### 구조체 분해
패턴을 이용해서 구조체를 분해할 수 도 있습니다.

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x: a, y: b } = p;
    assert_eq!(0, a);
    assert_eq!(7, b);
}
```

필드명과 변수명을 일치하면 좀 더 간단하게 사용할 수 있습니다.

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x, y } = p;
    assert_eq!(0, x);
    assert_eq!(7, y);
}
```

이를 `match` 식에서 사용할 수도 있습니다.

```rust
fn main() {
    let p = Point { x: 0, y: 7 };

    match p {
        Point { x, y: 0 } => println!("On the x axis at {}", x),
        Point { x: 0, y } => println!("On the y axis at {}", y),
        Point { x, y } => println!("On neither axis: ({}, {})", x, y),
    }
}
```

#### 열거형 분해

열거형의 인자도 동일하게 분해할 수 있습니다.

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let msg = Message::ChangeColor(0, 160, 255);

    match msg {
        Message::Quit => {
            println!("The Quit variant has no data to destructure.")
        }
        Message::Move { x, y } => {
            println!(
                "Move in the x direction {} and in the y direction {}",
                x, y
            );
        }
        Message::Write(text) => println!("Text message: {}", text),
        Message::ChangeColor(r, g, b) => println!(
            "Change the color to red {}, green {}, and blue {}",
            r, g, b
        ),
    }
}
```

#### 중첩 구조체 및 열거형 분해

중첩 구조체 및 열거형도 동일한 방법으로 패턴을 이용해 분해할 수 있습니다.

```rust
enum Color {
    Rgb(i32, i32, i32),
    Hsv(i32, i32, i32),
}

enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(Color),
}

fn main() {
    let msg = Message::ChangeColor(Color::Hsv(0, 160, 255));

    match msg {
        Message::ChangeColor(Color::Rgb(r, g, b)) => println!(
            "Change the color to red {}, green {}, and blue {}",
            r, g, b
        ),
        Message::ChangeColor(Color::Hsv(h, s, v)) => println!(
            "Change the color to hue {}, saturation {}, and value {}",
            h, s, v
        ),
        _ => (),
    }
}
```

#### 구조체와 튜플 분해

복잡한 방법으로 패턴을 혼합하여 일치 및 중첩을 표현할 수 있습니다.

```rust
let ((feet, inches), Point { x, y }) = ((3, 10), Point { x: 3, y: -10 });
```

### 패턴의 값 무시

`_`을 통해 이하의 패턴을 무시하거나 `..`을 이용해서 무시할 수도 있습니다.

#### `_`를 이용해서 전체 값 무시

```rust
fn foo(_: i32, y: i32) {
    println!("This code only uses the y parameter: {}", y);
}

fn main() {
    foo(3, 4);
}
```

#### `_`을 이용해서 중첩된 일부 값 무시

```rust
    let mut setting_value = Some(5);
    let new_setting_value = Some(10);

    match (setting_value, new_setting_value) {
        (Some(_), Some(_)) => {
            println!("Can't overwrite an existing customized value");
        }
        _ => {
            setting_value = new_setting_value;
        }
    }

    println!("setting is {:?}", setting_value);
```

중간 값을 무시할 수 도 있습니다.

```rust
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (first, _, third, _, fifth) => {
            println!("Some numbers: {}, {}, {}", first, third, fifth)
        }
    }
```

#### '_'로 시작하는 변수명으로 해당 변수 무시

아직 사용하지 않는 변수의 경우 컴파일 시 경고 메시지가 발생합니다. 이 때 `_`를 변수명 앞에 붙이면 경고가 사라지게 됩니다.

```rust
fn main() {
    let _x = 5;
    let y = 10;
}
```

하지만 `_`과 차이점이 있습니다. 경고를 발생하지 않을 뿐이지 여전히 변수로의 소유권 이동이 발생합니다.

```rust
    let s = Some(String::from("Hello!"));

    if let Some(_s) = s {
        println!("found a string");
    }

    println!("{:?}", s);
```

블럭에서 빠져나와 더 이상 `_s`를 사용할 수 없게 되므로 마지막 출력에서 s가 소유권이 없으므로 컴파일 오류가 발생합니다. 이것을 해결하려면 `_s` 대신 `_`를 사용해야 합니다.

```rust
    let s = Some(String::from("Hello!"));

    if let Some(_) = s {
        println!("found a string");
    }

    println!("{:?}", s);
```

#### `..`로 값의 나머지 무시

`..`를 이용하면 나머지 범위를 무시할 수 있습니다.

```rust
    struct Point {
        x: i32,
        y: i32,
        z: i32,
    }

    let origin = Point { x: 0, y: 0, z: 0 };

    match origin {
        Point { x, .. } => println!("x is {}", x),
    }
```

다음처럼 중간에 표현할 수 도 있습니다.

```rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (first, .., last) => {
            println!("Some numbers: {}, {}", first, last);
        }
    }
}
```

하지만 다음처럼 `second`의 위치를 알 수 없게 되면 컴파일 오류가 발생합니다.

```rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (.., second, ..) => {
            println!("Some numbers: {}", second)
        },
    }
}
```

```shell
$ cargo run
   Compiling patterns v0.1.0 (file:///projects/patterns)
error: `..` can only be used once per tuple pattern
 --> src/main.rs:5:22
  |
5 |         (.., second, ..) => {
  |          --          ^^ can only be used once per tuple pattern
  |          |
  |          previously used here

error: could not compile `patterns` due to previous error
```

### 매치 가드가 있는 추가 조건부
매치 가드를 통해 좀 더 복잡한 패턴 표현이 가능합니다.

```rust
    let num = Some(4);

    match num {
        Some(x) if x < 5 => println!("less than five: {}", x),
        Some(x) => println!("{}", x),
        None => (),
    }
```

여기서 `if x < 5`를 통해 x가 5보다 작은 경우를 매칭 처리 합니다.

다음은 앞에서 `y`를 가리게 되었을 때의 문제점을 해결 합니다.

```rust
fn main() {
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(n) if n == y => println!("Matched, n = {}", n),
        _ => println!("Default case, x = {:?}", x),
    }

    println!("at the end: x = {:?}, y = {}", x, y);
}
```

다음의 코드를 통해 `|`연산자를 사용했을 때도 매치 가드를 이용할 수 있습니다.

```rust
    let x = 4;
    let y = false;

    match x {
        4 | 5 | 6 if y => println!("yes"),
        _ => println!("no"),
    }
```

하지만 `4 또는 5 또는 6일 때 y가 참이면`의 의미인지 `6일 때 y가 참인지`의 의미인지 모호하게 보이지만 다음처럼 동작합니다.

```rust
(4 | 5 | 6) if y => ...
```

이것을 `6일 때 y가 참인지`로 표현하려면 다음처럼 해야 합니다.

```rust
4 | 5 | (6 if y) => ...
```

### @ 바인딩
@ 바인딩을 통해 패턴에서 바인딩할 수 없는 표현에서 바인딩을 할 수 있습니다.

```rust
    enum Message {
        Hello { id: i32 },
    }

    let msg = Message::Hello { id: 5 };

    match msg {
        Message::Hello {
            id: id_variable @ 3..=7,
        } => println!("Found an id in range: {}", id_variable),
        Message::Hello { id: 10..=12 } => {
            println!("Found an id in another range")
        }
        Message::Hello { id } => println!("Found some other id: {}", id),
    }
```

Message::Hello.id가 `3..=7`의 범위에 있을 경우 `id_variable`에 값이 바인딩 됩니다.

## 정리
이번 장에서 다양한 Rust의 패턴을 살펴봤습니다. Rust의 패턴 표현은 강력하며 다양한 식 또는 문에서 효과적으로 사용할 수 있습니다.
