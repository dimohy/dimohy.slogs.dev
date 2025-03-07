---
title: "Rust #10 : 10장 제네릭 타입, 트레잇, 수명"
datePublished: Thu Sep 02 2021 02:49:38 GMT+0000 (Coordinated Universal Time)
cuid: ckt2byazu098l3js1byfv1mpo
slug: rust-10-10
tags: rust

---

## 개요
중복된 로직을 줄이기 위한 도구로 제네릭을 설명합니다. 제네릭을 이용하면 유형에 상관없이 동일한 로직을 적용할 수 있습니다.

트레잇을 이용하면 다른 유형임에도 불구하고 추상적인 방식으로 동작을 일반화 하여 정의할 수 있습니다.

마지막으로 수명에 대해 살펴봅니다.

## 일반 데이터 유형
제네릭을 이용해서 함수 서명이나 구조체와 같은 항목에 대한 정의를 생성할 수있습니다.

### 함수 정의에서

다음의 코드를 봅시다.

```rust
fn largest_i32(list: &[i32]) -> i32 {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn largest_char(list: &[char]) -> char {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest_i32(&number_list);
    println!("The largest number is {}", result);
    assert_eq!(result, 100);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest_char(&char_list);
    println!("The largest char is {}", result);
    assert_eq!(result, 'y');
}
```

큰 수를 반환하는 동일한 로직의 다른 함수를 사용하고 있습니다. 함수 인자의 배열 형이 다르기 때문인데요, 제네릭을 이용해 다음처럼 표현할 수 있습니다.

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

제네릭 인자를 사용해서 `i32` 인자와 `char` 인자를 처리할 수 있게 되었는데요, 물론 위의 코드는 컴파일 시 오류가 납니다. 왜냐하면 제네릭 T 인자가 비교연산이 가능한지를 Rust 컴파일러에게 어떠한 힌트도 주지 못했기 때문입니다.

```shell
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0369]: binary operation `>` cannot be applied to type `T`
 --> src/main.rs:5:17
  |
5 |         if item > largest {
  |            ---- ^ ------- T
  |            |
  |            T
  |
help: consider restricting type parameter `T`
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> T {
  |             ^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0369`.
error: could not compile `chapter10`

To learn more, run the command again with --verbose.
```

이 문제를 해결하는 방법은 다음장인 트레잇을 다룰 때 해결됩니다.

### 구조체 정의에서

다음의 코드를 봅시다.

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
}
```

각각 `i32`형과 `f64`형의 Pointer 인스턴스를 생성했습니다. 하지만 다음의 코드는 컴파일 오류가 발생합니다.

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let wont_work = Point { x: 5, y: 4.0 };
}
```

이유는 x인자 값 '5'와 y인자 '4.0'은 다른 데이터형이므로 제네릭 T 인자로 표현되지 않기 때문입니다. 이것을 해결하려면, 두개의 제네릭 인자를 사용해야 합니다.

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

fn main() {
    let both_integer = Point { x: 5, y: 10 };
    let both_float = Point { x: 1.0, y: 4.0 };
    let integer_and_float = Point { x: 5, y: 4.0 };
}
```

### 열거형 정의

열거형에도 제네릭을 사용할 수 있습니다. 대표적인 사용이 `Option<T>` 열거형일 텐데요,

```rust
enum Option<T> {
    Some(T),
    None,
}
```

다음 처럼 제네릭 인자를 두개 사용할 수 있고 열겨 유형에 개별로 적용 가능합니다.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

### 메서드 정의에서

메서드의 인자라던가 반환 값에도 제네릭을 사용할 수 있습니다.

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let p = Point { x: 5, y: 10 };

    println!("p.x = {}", p.x());
}
```

만약 특정 데이터형에 종속적인 구현이 있다면 다음처럼 제네릭 인자에 데이터형을 명시해야 합니다.

```rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

`powi()`메소드는 `i32` 등의 정수형에는 존재하지 않기 때문입니다.

다음처럼 제네릭 인자를 혼합할 수 도 있습니다.

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 10.4 };
    let p2 = Point { x: "Hello", y: 'c' };

    let p3 = p1.mixup(p2);

    println!("p3.x = {}, p3.y = {}", p3.x, p3.y);
}
```

### 제네릭을 사용한 코드의 성능
Rust의 제네릭은 런타임  때 느려지지 않도록 디자인 되었습니다. 다음의 제네릭을 사용한 코드에서 컴파일 시 어떻게 일반 코드가 생성 되는지를 보여줍니다.

```rust
let integer = Some(5);
let float = Some(5.0);
```

컴파일 시 다음의 코드 처럼 변환됩니다.

```rust
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}

fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

### 제네릭 요약
Rust의 제네릭은 성능 저하 없이 다양한 데이터형을 처리할 수 있게 해주는 강력한 기능입니다. 이를 잘 사용하면 중복 코드 없이 코딩이 가능하게 됩니다. 더불어서 트레잇과 결합하여 일반 기능(함수)를 다른 유형과 공유할 수 있게 됩니다.

## 트레잇: 공유 행동 정의
트레잇은 다른 프로그래밍에서 말하는 인터페이스와 유사하고 할 수 있습니다. Rust에서의 트레잇은 특정 유형이 다른 유형과 기능을 공유할 수 있도록 해줍니다.

### 트레잇 정의

다음의 코드를 보시죠.

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

`Summary`라는 트레잇을 정의하고, `summarize()`라는 메소드를 정의했습니다. 구현은 빠져있는데 구현은 `Summary`를 `impl`에서 구현하게 됩니다.

### 유형에 대한 트레잇 구현

다음의 코드는 `NewArticle`과 `Tweet` 구조체에서 `Summary` 트레잇을 구현한 코드입니다.

```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

`Summary` 트레잇을 구현한 `NewsArticle`과 `Tweet`는 완전히 다른 사용자 유형입니다. 이것이 어떻게 트레잇을 통해 하나의 함수에서 처리 되는 지를 있다가 살펴보도록 합시다.

```rust
let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    println!("1 new tweet: {}", tweet.summarize());
```

이 코드는 굿이 트레잇을 사용하지 않더라도 문제없는 정상적인 코드입니다.

### 기본 구현
트레잇을 이용하면 메소드를 기본 구현할 수 있습니다.

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

기본 구현은 `impl`에서 해당 메소드를 구현하지 않으면 기본 구현이 사용됩니다. 만약 해당 메소드를 구현하게 되면 그 구현이 사용되게 됩니다.

또한 다음처럼 기본 구현이 구현 메소드를 호출하게 할 수 도 있습니다.

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}
```

```rust
impl Summary for Tweet {
    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }
}
```

```rust
    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    println!("1 new tweet: {}", tweet.summarize());
```

### 매개변수로서의 트레잇
함수의 매개변수로 트레잇의 구현체를 전달할 수 있습니다.

```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

`Summary`를 구현한 어떠한 유형이던 notify의 item 인자에 넘길 수 있습니다.

#### 특정 바인딩 구문

위의 매개변수로서의 전달 방식 말고 제네릭을 통해서도 동일한 동작을 하는 함수를 만들 수 있습니다.

```rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

함수 인자의 유형에 따라 `&impl`을 써도 되고 다음처럼 반복될 경우 바인딩 구문을 사용하는게 좋습니다.

```rust
pub fn notify(item1: &impl Summary, item2: &impl Summary) {
```

```rust
pub fn notify<T: Summary>(item1: &T, item2: &T) {
```

#### + 구문으로 여러 트레잇 경계 지정

또한 + 구문으로 하나 이상의 트레잇 바인딩을 지정할 수 있습니다.

```rust
pub fn notify(item: &(impl Summary + Display)) {
```

또는,

```rust
pub fn notify<T: Summary + Display>(item: &T) {
```

이것의 의미는 함수 `item` 인자가 `Summary` 트레잇과 `Display` 트레잇을 구현한 구현체임을 의미합니다.

### 절이 있는 명확한 트레잇 경계

트레잇 경계를 `where`절로도 표현할 수 있습니다. 복잡한 트레잇 경계일 경우 `where`절로 표현하는게 명확합니다.

```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{
```

### 트레잇을 구현하느 반환 유형

다음처럼 트레잇 구현을 반환할 수 도 있습니다.

```rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    }
}
```

하지만 다음처럼은 동작하지 않습니다. 왜냐하면 트레잇 역시 제네릭 처럼 컴파일 시점에서 데이터 유형이 결정되기 때문입니다. 다음의 코드는 런타임에 따라 두가지 유형의 인스턴스를 반환하는데요. Rust에서는 허용하지 않습니다. 다음의 코드는 컴파일 오류가 발생합니다.

```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        NewsArticle {
            headline: String::from(
                "Penguins win the Stanley Cup Championship!",
            ),
            location: String::from("Pittsburgh, PA, USA"),
            author: String::from("Iceburgh"),
            content: String::from(
                "The Pittsburgh Penguins once again are the best \
                 hockey team in the NHL.",
            ),
        }
    } else {
        Tweet {
            username: String::from("horse_ebooks"),
            content: String::from(
                "of course, as you probably already know, people",
            ),
            reply: false,
            retweet: false,
        }
    }
}
```

### 특정 경계가 있는 largest 함수 수정

앞의 제네릭에서 `i32`와 `char`형을 받는 제네릭을 코드가 컴파일 오류가 발생하는 것을 살펴봤습니다. 제니릭 `T` 인자가 비교 연산을 지원하는 지를 Rust 컴파일에서 알 수 없기 때문인데요.

```shell
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0369]: binary operation `>` cannot be applied to type `T`
 --> src/main.rs:5:17
  |
5 |         if item > largest {
  |            ---- ^ ------- T
  |            |
  |            T
  |
help: consider restricting type parameter `T`
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> T {
  |             ^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0369`.
error: could not compile `chapter10`

To learn more, run the command again with --verbose.
```

위의 오류 메시지를 보시면, 제니릭 T 인자가 `std::cmp::PartialOrd`를 지원해야 한다고 가이드 하고 있습니다. 네. 비교 연산을 하려면 `std::cmp::PartialOrd` 트레잇을 구현하거나 구현되었음을 표시해야 한다는 것 이였습니다!

```rust
fn largest<T: PartialOrd>(list: &[T]) -> T {
```

그런데 여전히 오류가 발생합니다.

```shell
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0508]: cannot move out of type `[T]`, a non-copy slice
 --> src/main.rs:2:23
  |
2 |     let mut largest = list[0];
  |                       ^^^^^^^
  |                       |
  |                       cannot move out of here
  |                       move occurs because `list[_]` has type `T`, which does not implement the `Copy` trait
  |                       help: consider borrowing here: `&list[0]`

error[E0507]: cannot move out of a shared reference
 --> src/main.rs:4:18
  |
4 |     for &item in list {
  |         -----    ^^^^
  |         ||
  |         |data moved here
  |         |move occurs because `item` has type `T`, which does not implement the `Copy` trait
  |         help: consider removing the `&`: `item`

error: aborting due to 2 previous errors

Some errors have detailed explanations: E0507, E0508.
For more information about an error, try `rustc --explain E0507`.
error: could not compile `chapter10`

To learn more, run the command again with --verbose.
```

Copy 트레잇을 구현해야 한다는 이야기인데요, 최종적으로 다음처럼 수정하면 컴파일이 정상적으로 이루어지고 실행이 되게 됩니다.

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

즉, `i32`형과 `char`형은 이미 `PartialOrd` 트레잇과 `Copy` 트레잇이 구현되어 있음을 알 수 있습니다.

### 특성 경계를 사용하여 메서드를 조건부 구현합니다.

앞전의 제네릭의 경우와 마찬가지로 트레잇도 조건부 구현을 통해 구현할 수 있습니다.

```rust
use std::fmt::Display;

struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}

impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```

또한 모든 형식에 대한 특성을 조건부적으로 구현할 수 있습니다.

```rust
impl<T: Display> ToString for T {
    // --snip--
}
```

이를 통해 다음의 코드가 정상 동작하게 됩니다.

```rust
let s = 3.to_string();
```

### 트레잇 정리
트레잇은 Rust의 강력한 도구입니다. 제네릭이 다양한 유형을 동일한 로직으로 처리할 수 있게 했다면, 트레잇은 동일한 기능을 다양한 유형에서 쓸 수 있도록 합니다. 그러나 다른 프로그램 언어와 달리 트레잇은 컴파일 시점에서 기능이 유형에 따라 결정이 되므로 다른 프로그래밍 언어의 인터페이스가 런타임 때 유형을 인터페이스로 인식으로 처리하는 방식과 다르게 동작합니다.

## 수명 참조 검사

4장 `참조 및 차용`에서 다루지 않은 것은 Rust의 모든 참조에는 그 참조의 유효한 범위가 있다는 것입니다. 대부분의 경우 수명은 암시적이며 유추됩니다. 이 장에서는 유추되지 않는 유형에 대해 알아보고 암시적으로 유추될 수 있는 경우에 대해 알아봅니다.

### 수정으로 댕글링 참조 방지하기
수명의 주요 목표는 댕글링 참조를 방지하는 것입니다. 실제로 댕글링 참조로 인해 많은 메모리 관련 버그가 발생하는데요, Rust는 언어 자체에서 이를 강력하게 막습니다.

```rust
    {
        let r;

        {
            let x = 5;
            r = &x;
        }

        println!("r: {}", r);
    }
```

위의 코드를 보시면, `r`변수가 참조하고 있는 `x`변수는 블록이 빠져나올 때 해제되므로 `println!()`함수에서 r을 출력할 수 없습니다. Rust 언어는 컴파일 때 이를 오류로 감지합니다.


### 차용 검사기
Rust 컴파일러는 차용 검사기를 통해 이를 감지하도록 서설계되었습니다.

```rust
    {
        let r;                // ---------+-- 'a
                              //          |
        {                     //          |
            let x = 5;        // -+-- 'b  |
            r = &x;           //  |       |
        }                     // -+       |
                              //          |
        println!("r: {}", r); //          |
    }                         // ---------+
```

이 코드는 컴파일 오류를 발생합니다. 정상적인 코드는 다음과 같습니다.

```rust
    {
        let x = 5;            // ----------+-- 'b
                              //           |
        let r = &x;           // --+-- 'a  |
                              //   |       |
        println!("r: {}", r); //   |       |
                              // --+       |
    }  
```

### 함수의 일반 수명
함수에 입력된 두 문자열 중 길이가 긴 문자열을 반환한다고 생각해봅시다. 함수는 입력된 두 인자의 길이를 비교하고 그 중 큰 문자열을 반환하게 됩니다. 그런데 함수에서는 입력된 두 문자열의 수명을 알 수 없습니다. `if`문에 의해 수명이 다를 수 도 있는 값을 반환하게 되므로 다음의 코드는 컴파일 오류를 발생합니다.

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
``` 

```shell
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0106]: missing lifetime specifier
 --> src/main.rs:9:33
  |
9 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
9 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ^^^^    ^^^^^^^     ^^^^^^^     ^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0106`.
error: could not compile `chapter10`

To learn more, run the command again with --verbose.
```

### 수명 주석 구문

그렇다면 어떻게 해야 할까요? Rust에서는 수명 주석 구문을 제공합니다. 동일한 수명 주석 구문을 사용했다면 수명이 동일하다는 것을 의미합니다.

```rust
&i32        // a reference
&'a i32     // a reference with an explicit lifetime
&'a mut i32 // a mutable reference with an explicit lifetime
```

### 함수 서명의 수명 주석

수명 주석 구문을 이용해 컴파일 오류를 수정할 수 있습니다.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

이제 함수 인자 `x`와 `y`, 그리고 반환 값의 수명이 수명 주석을 통해 같음을 명시하였습니다. 이제 이 함수는 정상적인 코드가 됩니다. 여기서 `'a'는 `'b' 등으로 대체해서 쓸 수 있습니다. 만약, 함수 인자 마다 수명이 다를 수 있다면, 같은 수명끼리 이렇게 묶어줄 수 있습니다.

이제 다음의 코드는 수명 주기에 의해서 정상적인 코드가 됩니다.

```rust
fn main() {
    let string1 = String::from("long string is long");

    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    }
}
```

그런데 입력된 두 인자의 수명이 다를 경우 컴파일 오류가 발생할 것입니다.

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

```shell
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0597]: `string2` does not live long enough
 --> src/main.rs:6:44
  |
6 |         result = longest(string1.as_str(), string2.as_str());
  |                                            ^^^^^^^ borrowed value does not live long enough
7 |     }
  |     - `string2` dropped here while still borrowed
8 |     println!("The longest string is {}", result);
  |                                          ------ borrow later used here

error: aborting due to previous error

For more information about this error, try `rustc --explain E0597`.
error: could not compile `chapter10`

To learn more, run the command again with --verbose.
```

### 수명 관점에서 생각하기

Rust를 올바르게 사용하기 위해서는 수명 관점에서 코드를 바라봐야 합니다. 아래의 코드를 봅시다.

```rust
fn longest<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

반환하는 값이 `x`값으로 결정되어 있으므로 정상적인 코드입니다.

하지만 다음의 코드는 오류를 발생합니다.

```rust
fn longest<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("really long string");
    result.as_str()
}
```

이유는 함수 내부에서 `result` 변수가 생성이 되었고 함수를 빠져나가자 마자 소멸될 텐데, 그 값을 반환하기 때문입니다.

```shell
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0515]: cannot return value referencing local variable `result`
  --> src/main.rs:11:5
   |
11 |     result.as_str()
   |     ------^^^^^^^^^
   |     |
   |     returns a value referencing data owned by the current function
   |     `result` is borrowed here

error: aborting due to previous error

For more information about this error, try `rustc --explain E0515`.
error: could not compile `chapter10`

To learn more, run the command again with --verbose.
```

### 구조체 정의의 수명 주석

지금까지 우리는 함수에서의 수명 주석에 대해 알아보았습니다. 구조체 역시 수명주기를 사용할 수 있습니다.

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

### 수명 제거

이제 우리는 Rust는 수명 주석에 의해서 수명 매개변수를 지정해야 한다는 것을 알았습니다. 그렇다면, 수명 주석을 사용하지 않더라도 정상적으로 컴파일 되는 코드는 무엇일까요?

아래의 코드는 정상 컴파일 및 실행이 됩니다.

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

이것은 Rust 컴파일러가 인자 및 반환 값의 수명을 유추할 수 있기 때문인데요, 하나의 인자와 그 값을 반환하기 때문에 실제론 다음과 같습니다.

```rust
fn first_word<'a>(s: &'a str) -> &'a str {
```

Rust 컴파일은 암시적으로 유추하여 위의 코드로 처리합니다.

### 메서드 정의의 수명 주석

수명이 있는 구조체에서 메서드를 구현할 때 일반 유형 매개변수의 구문과 동일한 구문을 사용하게 됩니다. 아래의 코드를 보시죠. 

```rust
impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
}
```

`impl` 키워드 뒤와 구조체 이름 뒤에 사용되었음을 볼 수 있습니다. 이는 수명이 구조체 유형의 일부이기 때문입니다.

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

다음의 코드를 보면,

```rust
impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
}
```

첫번째 인자가 있지만 자기 자신이므로 규칙을 생략할 수 있습니다.

다음의 코드를 보시죠.

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

두개의 인자가 있지만 첫번째는 자기 자신이므로 생략되고 두번째 인자에 암묵적으로 적용되게 됩니다.

### 정적 수명

다음은 정적 수명입니다. 정적 수명이란 프로그램이 시작될 때부터 종료할 때까지의 수명을 가진다는 것으로 한마디로 수명 검사를 하지 않는 것과 동일 합니다.

```rust
let s: &'static str = "I have a static lifetime.";
```

이것은 문자열 리터럴 등 명확히 정적 수명을 가진 것에만 조심스럽게 사용해야 합니다. 왜냐하면 정적 수명을 통해 `댕글링 참조`가 가능해지기 때문입니다.


## 제네릭 유형 매개변수, 트레잇 경계와 수명을 함께

하나의 함수에서 제네릭 유형 매개변수와 트레잇 경계 및 수명을 모두 지정하는 구문을 살펴봅시다.

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

## 정리
이제 제네릭 유형 매개변수, 트레잇 및 트레잇 경계, 제네릭 수명 매개변수에 대해 살펴보았습니다!