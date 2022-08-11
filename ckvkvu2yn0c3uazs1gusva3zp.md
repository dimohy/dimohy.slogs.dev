## Rust #19: 19장 고급 기능

## 개요
이번 장은 다음의 Rust 고급 기술에 대해 설명합니다.

- 안전하지 않은 Rust: Rust의 보증의 일부를 해제해서 이를 수동으로 유지하는 책임을 지는 방법
- 고급 트레잇: 연관유형, 기본 유형 매개변수, 정규화된 구문, 상위 트레잇 및 트레잇과 관련된 newtype 패턴
- 고급 유형: newtype 패턴, 유형 별칭, naver 유형 및 동적 사이즈 유형에 대한 더 많은 정보
- 고급 기능 및 클로저: 함수 포인터와 클로저 반환
- 매크로: 컴파일 시간에 더 많은 코드를 정의하는 방법


## 안전하지 않은 Rust

Rust는 두 가지 이유 때문에 메모리 안전 보장을 적용하지 않은 Rust 모드가 있습니다. 이를 안전하지 않은(unsafe) Rust 라고 합니다.

- 컴파일 시점에서 정보가 충분하지 않다면 컴파일 오류가 발생하도록 하는 것이 Rust가 지향하는 방법입니다. 하지만 특정한 경우에 이를 허용 하면 더 효율적이기도 합니다.
- 컴퓨터 하드웨어가 본질적으로 안전하지 않으므로 저 수준 시스템 프로그래밍 시 이를 허용할 필요가 있습니다.

### 안전하지 않은 초능력

안전하지 않은 Rust에서는 안전한 Rust에서는 사용할 수 없는 5가지 작업을 수행할 수 있습니다.

- 원시 포인터 역참조
- 안전하지 않은 함수 또는 메소드 호출
- 변경 가능한 정적 변수에 대한 접근 또는 수정
- 안전하지 않은 트레잇 구현
- union S의 필드 접근

안전하지 않은 Rust에서 위의 5가지 해제 기능을 제외한 나머지 안전 기능을 그대로 사용할 수 있습니다. 안전하지 않은 Rust는 unsafe 블럭과 unsafe 함수를 통해 표현됩니다. 

### 원시 포인터 역참조

안전하지 않은 Rust에서는 참조와 유사한 `*const T`와 `*mut T`의 두가지 원시 포인터 유형이 있습니다. 여기서 별포는 역참조 연산자가 아니라 유형 표현의 일부입니다.

- 변경 가능/불가능 포인터 또는 동일한 위치에 여러 개의 변경 가능한 포인터를 이용해서 차용 규칙을 무시할 수 있습니다.
- 유효한 메모리를 가리키도록 보장하지 않음
- null이 허용됨
- 자동 정리를 구현해서는 안됩니다.

원시 포인터 역참조를 통해 보장이 적용되지 않는 다른 언어나 하드웨어와 인터페이스 할 수 있게 되고 안전을 포기하는 대신 더 나은 성능을 가질 수 있습니다.

다음은 원시 포인터를 생성하는 방법입니다.

```rust
    let mut num = 5;

    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;
```

unsafe 키워드를 사용하지 않았는데도 컴파일이 정상적으로 됩니다. 메모리 주소를 얻는 것 자체는 unsafe 키워드가 필요하지 않는 안전한 코드입니다.

```rust
let address = 0x012345usize;
let r = address as *const i32;
```

특정 메모리 주소에 대한 역참조도 안전한 코드입니다.  하지만 이를 접근했을 때 컴파일 오류가 발생합니다.

```rust
    let address = 0x012345usize;
    let r = address as *const i32;

    print!("{}", *r); // 컴파일 오류
```

이를 해결하려면 다음 처럼 unsafe 블럭을 사용해야 합니다.

```rust
    let address = 0x012345usize;
    let r = address as *const i32;

    unsafe {
        print!("{}", *r);
    }
```

### 안전하지 않은 함수 또는 메서드 호출

안전하지 않은 함수를 호출하기 위해선 unsafe 블럭이 필요합니다.

```rust
    unsafe fn dangerous() {}

    unsafe {
        dangerous();
    }
```

만약 unsafe 블럭을 사용하지 않고 안전하지 않은 함수를 호출하면 다음과 같은 컴파일 오류가 발생합니다.

```rust
$ cargo run
   Compiling unsafe-example v0.1.0 (file:///projects/unsafe-example)
error[E0133]: call to unsafe function is unsafe and requires unsafe function or block
 --> src/main.rs:4:5
  |
4 |     dangerous();
  |     ^^^^^^^^^^^ call to unsafe function
  |
  = note: consult the function's documentation for information on how to avoid undefined behavior

For more information about this error, try `rustc --explain E0133`.
error: could not compile `unsafe-example` due to previous error
```

unsafe 블럭은 블럭안에서 안전하지 않은 함수를 호출할 수 있게 해주고 블럭 안의 코드가 안전하지 않은 코드라는 것을  코드로 인식할 수 있게 해줍니다. 

#### 안전하지 않은 코드에 대한 안전한 추상화 만들기

함수에 안전하지 않은 코드가 있다고 해서 전체 함수가 안전하지 않은 것은 아닙니다. 사실 안전하지 않은 코드를 안전한 함수로 래핑하는 것은 일반적인 추상화 입니다. 그 사례를 살펴봅시다.

다음의 예시 코드를 봅시다.

```rust
    let mut v = vec![1, 2, 3, 4, 5, 6];

    let r = &mut v[..];

    let (a, b) = r.split_at_mut(3);

    assert_eq!(a, &mut [1, 2, 3]);
    assert_eq!(b, &mut [4, 5, 6]);
```

`split_at_mut()`함수는 주어진 인자 기준으로 배열을 분할 합니다. `let (a, b) = r.split_at_mut(3)`은 `a`에 `[1, 2, 3]`, `b`에 `[4, 5, 6]`로 분할합니다. 하지만 이 기능은 안전한 Rust 코드로 작성할 수 없습니다.

```rust
fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();

    assert!(mid <= len);

    (&mut slice[..mid], &mut slice[mid..])
}
```

이는 다음과 같은 컴파일 오류가 발생합니다.

```rust
$ cargo run
   Compiling unsafe-example v0.1.0 (file:///projects/unsafe-example)
error[E0499]: cannot borrow `*slice` as mutable more than once at a time
 --> src/main.rs:6:30
  |
1 | fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
  |                        - let's call the lifetime of this reference `'1`
...
6 |     (&mut slice[..mid], &mut slice[mid..])
  |     -------------------------^^^^^--------
  |     |     |                  |
  |     |     |                  second mutable borrow occurs here
  |     |     first mutable borrow occurs here
  |     returning this value requires that `*slice` is borrowed for `'1`

For more information about this error, try `rustc --explain E0499`.
error: could not compile `unsafe-example` due to previous error
```

두 개로 나눠서 참조하는 것은 참조 위치가 겹칠 수 있어서 Rust는 컴파일 오류를 발생합니다. 코드가 괜찮다는 것을 알지만 경계 값이 동적으로 결정되기 때문에 Rust 컴파일러가 그렇게 까지 컴파일 오류로 잡아주지는 않습니다. 이럴 때 unsafe를 사용할 수 있습니다.

```rust
use std::slice;

fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();

    assert!(mid <= len);

    unsafe {
        (
            slice::from_raw_parts_mut(ptr, mid),
            slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
```

`slice::from_raw_parts_mut()`함수는 안전하지 않은 함수이므로 unsafe 블럭을 통해 감싸주면 `split_at_mut()`함수는 안전하게 호출할 수 있게 됩니다. 다만, 안전하지 않은 코드는 컴파일 시 메모리 안전을 보장하지 않으므로 충분히 테스트를 통해서 검증을 한 뒤 사용되어야 합니다.

#### extern함수를 사용하여 외부 코드 호출

때로는 Rust가 다른 언어로 만들어진 함수를 호출할 필요가 있습니다.  이때 `extern` 키워드를 사용할 수 있습니다.

```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```

또한 `extern`은 다른 언어에서 Rust로 만든 함수를 호출할 수 있도록 도 해줍니다.

```rust
#[no_mangle]
pub extern "C" fn call_from_c() {
    println!("Just called a Rust function from C!");
}
```

### 가변 정적 변수 접근 또는 수정

Rust는 전역 변수를 지원합니다. 이를 정적 변수라고 합니다.

```rust
static HELLO_WORLD: &str = "Hello, world!";

fn main() {
    println!("name is: {}", HELLO_WORLD);
}
```

정적 변수를 수정하는 것은 안전하지 않습니다. 이를 안전하지 않은 코드를 통해 가능하게 할 수 있습니다.

```rust
static mut COUNTER: u32 = 0;

fn add_to_count(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    add_to_count(3);

    unsafe {
        println!("COUNTER: {}", COUNTER);
    }
}
```

### 안전하지 않은 트레잇 구현

트레잇에서 제공하는 하나의 메서드에 컴파일러가 확인 할 수 없는 불변량이 있을 경우 안전하지 않은 트레잇을 사용할 수 있습니다.

```rust
unsafe trait Foo {
    // methods go here
}

unsafe impl Foo for i32 {
    // method implementations go here
}

fn main() {}
```

### Union 필드 액세스

Union 필드는 C의 union을 지원하기 우해 Rust에 존재합니다. 일반적으로 동일한 메모리를 다른 유형으로 접근하는 것은 안전하지 않습니다. 그렇기 때문에 Rust는 이를 안전하지 않은 코드로 보고 안전하지 안은 Rust에서만 지원합니다.

### 안전하지 않은 코드를 사용해야 하는 경우

Rust는 메모리 안전을 유지하기 위해 노력합니다. 그럼에도 불구하고 이를 해제했을 때 유용한 경우가 존재합니다. 이럴 때 주의 깊게 안전하지 않은 코드를 사용할 수 있습니다. Rust는 이런 코드를 명시적으로 unsafe 블럭을 사용해야만 쓸 수 있도록 해서 문제가 발생했을 때 원인을 더 쉽게 추적할 수 있도록 합니다.

## 고급 트레잇

### 트레잇 정의에 연결된 형식과 자리 표시자 유형 지정

Rust는 연관된 유형을 통해 자리표시자 유형을 특정 메서드 정의에서 사용할 수 있도록 해서  트레잇의 특정 유형을 구현 시점에서 결정할 수 있도록 합니다.

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

실제로 구현하는 다음의 Counter에서 Item의 유형을 결정할 수 있습니다.

```rust
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        // --snip--
```

그런데 이것은 이미 제네릭 인자를 통해 달성할 수 있습니다.

```rust
pub trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```

자리 표시자를 통해 연관된 유형을 사용하는 이유는 기존 제네릭의 인자를 통해 이를 달성하면 트레잇의 구현 단계에서 제네릭 인자 유형이 결정이 되고, 이 유형이 복수 유형이 될 수 있습니다. 그렇게 되면 `Iterator` 등 하나의 유형이어야 하는 곳에서 적합하지 않습니다.

### 기본 제네릭 인자 및 연산자 오버로딩

기본 제네릭 인자의 기본 유형을 지정할 수 있습니다.

```rust
use std::ops::Add;

#[derive(Debug, Copy, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    assert_eq!(
        Point { x: 1, y: 0 } + Point { x: 2, y: 3 },
        Point { x: 3, y: 3 }
    );
}
```

트레잇은 다음과 같습니다.

```rust
trait Add<Rhs=Self> {
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}
```

`add()`메소드의 두번째 인자가 Rhs라는 유형이고 이 유형은 `<Rhs=Self>`에 의해서 자기 자신이 됩니다.

다음은 서로 다른 단위의 값을 더하는 코드 입니다.

```rust
use std::ops::Add;

struct Millimeters(u32);
struct Meters(u32);

impl Add<Meters> for Millimeters {
    type Output = Millimeters;

    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + (other.0 * 1000))
    }
}
```

`Rhs` 유형을 지정해서 `add 트레잇`을 변경하지 않고도 사용할 수 있었습니다.

이는 다음의 두가지의 이유로 사용됩니다.

- 기존 코드를 변경하지 않고 유형을 확장
- 필요하지 않은 특정의 경우 사용자 정의를 허용

### 명확성을 위한 정규화된 구문: 동일한 이름의 메서드 호출

여러 트레잇을 사용하게 되었을 때 트레잇 메소드가 중복될 수 있습니다.

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}

impl Human {
    fn fly(&self) {
        println!("*waving arms furiously*");
    }
}
```

여기서는 Pilot의 `fly()`와 Wizard의 `fly()`와 Human의 `fly()`가 모두 이름이 같습니다. Rust는 기본적으로 Human의 `fly()`를 호출하게 됩니다.

```rust
fn main() {
    let person = Human;
    person.fly();
}
```

| 출력 결과
```shell
*waving arms furiously*
```

이를 각 트레잇의 관점으로 호출하려면 다음과 같이 할 수 있습니다.

```rust
fn main() {
    let person = Human;
    Pilot::fly(&person);
    Wizard::fly(&person);
    person.fly();
}
```

| 출력 결과
```shell
$ cargo run
   Compiling traits-example v0.1.0 (file:///projects/traits-example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.46s
     Running `target/debug/traits-example`
This is your captain speaking.
Up!
*waving arms furiously*
```

그런데 특성의 일부인 관련 함수의 경우 (self 인자가 없는) 이런 방식으로 호출할 수가 없습니다.

```rust
trait Animal {
    fn baby_name() -> String;
}

struct Dog;

impl Dog {
    fn baby_name() -> String {
        String::from("Spot")
    }
}

impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}

fn main() {
    println!("A baby dog is called a {}", Dog::baby_name());
}
```

| 출력 결과
```shell
$ cargo run
   Compiling traits-example v0.1.0 (file:///projects/traits-example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.54s
     Running `target/debug/traits-example`
A baby dog is called a Spot
```

이를 위의 방법 처럼,

```rust
fn main() {
    println!("A baby dog is called a {}", Animal::baby_name());
}
```

호출하게 되면 인스턴스화 된 대상을 호출한 것이 아니므로 컴파일 오류가 발생합니다.

```rust
$ cargo run
   Compiling traits-example v0.1.0 (file:///projects/traits-example)
error[E0283]: type annotations needed
  --> src/main.rs:20:43
   |
20 |     println!("A baby dog is called a {}", Animal::baby_name());
   |                                           ^^^^^^^^^^^^^^^^^ cannot infer type
   |
   = note: cannot satisfy `_: Animal`
note: required by `Animal::baby_name`
  --> src/main.rs:2:5
   |
2  |     fn baby_name() -> String;
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^

For more information about this error, try `rustc --explain E0283`.
error: could not compile `traits-example` due to previous error
```

이런 경우 다음처럼 인스턴스를 타입 케스팅 해서 호출할 수 있습니다.

```rust
fn main() {
    println!("A baby dog is called a {}", <Dog as Animal>::baby_name());
}
```

| 출력 결과
```shell
$ cargo run
   Compiling traits-example v0.1.0 (file:///projects/traits-example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.48s
     Running `target/debug/traits-example`
A baby dog is called a puppy
```

일반적으로 정규화된 구문은 다음과 같이 정의 합니다.

```rust
<Type as Trait>::function(receiver_if_method, next_arg, ...);
```

### Supertraits를 이용해서 다른 트레잇 내에서 특정 기능을 요구하기

다음을 출력하기 위해서

```
**********
*        *
* (1, 3) *
*        *
**********
```

다음의 트레잇을 만들 수 있습니다.

```rust
use std::fmt;

trait OutlinePrint: fmt::Display {
    fn outline_print(&self) {
        let output = self.to_string();
        let len = output.len();
        println!("{}", "*".repeat(len + 4));
        println!("*{}*", " ".repeat(len + 2));
        println!("* {} *", output);
        println!("*{}*", " ".repeat(len + 2));
        println!("{}", "*".repeat(len + 4));
    }
}
```

Display 구현이 없으므로 컴파일 오류가 발생합니다.

```shell
$ cargo run
   Compiling traits-example v0.1.0 (file:///projects/traits-example)
error[E0277]: `Point` doesn't implement `std::fmt::Display`
  --> src/main.rs:20:6
   |
3  | trait OutlinePrint: fmt::Display {
   |                     ------------ required by this bound in `OutlinePrint`
...
20 | impl OutlinePrint for Point {}
   |      ^^^^^^^^^^^^ `Point` cannot be formatted with the default formatter
   |
   = help: the trait `std::fmt::Display` is not implemented for `Point`
   = note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead

For more information about this error, try `rustc --explain E0277`.
error: could not compile `traits-example` due to previous error
```

다음처럼 Display를 구현한 후

```rust
use std::fmt;

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```

`outputline_print()` 트레잇을 사용할 수 있게 됩니다.

## 새로운 유형 패턴을 이용해서 외부 유형에 대한 외부 트레잇 구현

새로운 구조체를 통해 새로운 출력 속성을 부여할 수 있습니다.

```rust
use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("hello"), String::from("world")]);
    println!("w = {}", w);
}
```

여기서 `Wrapper`는 `Vec<String>` 유형을 필드로 해서 새로운 출력 형태를 구현했습니다.

## 고급 유형

### 타입 안전성과 추상화를 위한 새로운 유형 패턴 사용하기

Rust는 기존에 존재하는 유형을 새로운 타입으로 재정의 할 수 있습니다. 이것은 함수 인자로 값을 넘기게 될 때 자칫 순서가 바뀌어서 잘못된 값이 설정되는 등의 실수를 막아줄 수 있습니다.

### 유형 별칭으로 유형 동의어 만들기

유형 별칭은 동일한 유형이지만 그 의미를 좀 더 명확하게 표현할 수 있습니다.

```rust
    type Kilometers = i32;

    let x: i32 = 5;
    let y: Kilometers = 5;

    println!("x + y = {}", x + y);
```

또다른 활용으로는 긴 표현을 짧게 줄였을 때의 유용함입니다.

다음처럼 조금 긴 유형이 있을 때

```rust
Box<dyn Fn() + Send + 'static>
```

이를 사용하게 되면 코드는 다음과 같이 됩니다.

```rust
    let f: Box<dyn Fn() + Send + 'static> = Box::new(|| println!("hi"));

    fn takes_long_type(f: Box<dyn Fn() + Send + 'static>) {
        // --snip--
    }

    fn returns_long_type() -> Box<dyn Fn() + Send + 'static> {
        // --snip--
    }
```

아무리 봐도 가독성이 떨어지는데요, 이때 유형 별칭을 사용할 수 있습니다.

```rust
type Thunk = Box<dyn Fn() + Send + 'static>;

    let f: Thunk = Box::new(|| println!("hi"));

    fn takes_long_type(f: Thunk) {
        // --snip--
    }

    fn returns_long_type() -> Thunk {
        // --snip--
    }
```

`Reuslt<T, E>`에서도 사용되는데 다음의 코드를 보시죠.

```rust
use std::fmt;
use std::io::Error;

pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize, Error>;
    fn flush(&mut self) -> Result<(), Error>;

    fn write_all(&mut self, buf: &[u8]) -> Result<(), Error>;
    fn write_fmt(&mut self, fmt: fmt::Arguments) -> Result<(), Error>;
}
```

`Result<..., Error>`형태가 자주 반복이 됩니다. 이를 다음을 통해

```rust
type Result<T> = std::result::Result<T, std::io::Error>;
```

축약하면 다음처럼 코드가 간결해 집니다.

```rust
pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;

    fn write_all(&mut self, buf: &[u8]) -> Result<()>;
    fn write_fmt(&mut self, fmt: fmt::Arguments) -> Result<()>;
}
```

### 절대 돌아오지 않는 Never 유형

Rust는 반환하지 않는 빈 유형으로 `!`를 사용합니다. 반환하지 않는 함수가 어떤 의미가 있을까요? 다음의 코드를 봅시다.

```rust
        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };
```

`match` 대상은 모두 동일 유형을 반환해야 합니다. 다음을 살펴봅시다.

```rust
    let guess = match guess.trim().parse() {
        Ok(_) => 5,
        Err(_) => "hello",
    };
```

이는 동일한 유형이 아니므로 컴파일 오류가 발생합니다.  그렇다면 추측한 대로 `continue`는 반환하지 않는 빈 유형인 `!`값을 사용하고 있다는 것을 유추할 수 있습니다. continue는 내부적으로 `guess`의 할당을 무시하고 루프의 맨 위로 이동합니다.

panic!의 예로도 살펴볼 수 있습니다.

```rust
impl<T> Option<T> {
    pub fn unwrap(self) -> T {
        match self {
            Some(val) => val,
            None => panic!("called `Option::unwrap()` on a `None` value"),
        }
    }
}
```

`panic!` 매크로는 결코 반환하지 않으므로 `!` 유형입니다.

유형 `!`의 마지막 예시는 loop 사용입니다.

```rust
    print!("forever ");

    loop {
        print!("and ever ");
    }
```

### 동적 크기 유형 및 Sized 특성

Rust는 컴파일 시점에서 동이한 유형은 동일한 사이즈를 가져야만 합니다. 그러므로 다음의 코드는 컴파일 오류가 발생합니다.

```rust
    let s1: str = "Hello there!";
    let s2: str = "How's it going?";
```

다음의 제네릭 메소드를 봅시다.

```rust
fn generic<T>(t: T) {
    // --snip--
}
```

이것은 다음처럼 처리 됩니다.

```rust
fn generic<T: Sized>(t: T) {
    // --snip--
}
```

그러므로 기본적으로 제네릭 함수는 컴파일 시점에서 크기가 정해진 형식에서만 동작합니다. 하지만 특수 구문을 사용해서 이 제한을 완화할 수 있습니다.

```rust
fn generic<T: ?Sized>(t: &T) {
    // --snip--
}
```

## 고급 함수 및 클로저

### 함수 포인터

함수에 클로저를 전달하는 방법을 배웠습니다. 그런데 크로저 뿐만 아니라 일반 함수도 함수 인자로 전달할 수 있습니다.

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}

fn main() {
    let answer = do_twice(add_one, 5);

    println!("The answer is: {}", answer);
}
```

다음의 예시 코드는 map함수에서 클로저 뿐만 아니라 `ToString::to_string()` 함수를 넣어 사용할 수 있음을 확인할 수 있습니다.

```rust
    let list_of_numbers = vec![1, 2, 3];
    let list_of_strings: Vec<String> =
        list_of_numbers.iter().map(|i| i.to_string()).collect();
```

```rust
    let list_of_numbers = vec![1, 2, 3];
    let list_of_strings: Vec<String> =
        list_of_numbers.iter().map(ToString::to_string).collect();
```

이를 열거형 목록을 초기화 하는데 응용할 수 도 있습니다.

```rust
    enum Status {
        Value(u32),
        Stop,
    }

    let list_of_statuses: Vec<Status> = (0u32..20).map(Status::Value).collect();
```

### 클로저 반환

클로저 자체는 반환될 수 없습니다.

```rust
fn returns_closure() -> dyn Fn(i32) -> i32 {
    |x| x + 1
}
```

```shell
$ cargo build
   Compiling functions-example v0.1.0 (file:///projects/functions-example)
error[E0746]: return type cannot have an unboxed trait object
 --> src/lib.rs:1:25
  |
1 | fn returns_closure() -> dyn Fn(i32) -> i32 {
  |                         ^^^^^^^^^^^^^^^^^^ doesn't have a size known at compile-time
  |
  = note: for information on `impl Trait`, see <https://doc.rust-lang.org/book/ch10-02-traits.html#returning-types-that-implement-traits>
help: use `impl Fn(i32) -> i32` as the return type, as all return paths are of type `[closure@src/lib.rs:2:5: 2:14]`, which implements `Fn(i32) -> i32`
  |
1 | fn returns_closure() -> impl Fn(i32) -> i32 {
  |                         ^^^^^^^^^^^^^^^^^^^

For more information about this error, try `rustc --explain E0746`.
error: could not compile `functions-example` due to previous error
```

오류 메시지를 봤을 때 Sized 관련 오류임을 알 수 있습니다.  Rust에서 클로저를 저장하기 위해 얼마 만큼의 공간이 필요한지 모릅니다! 우리는 앞전에 해결 방법을 살폈습니다.

```rust
fn returns_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}
```

이제 클로저를 반환할 수 있게 되었습니다.

## 매크로
우리는 `println!()`등을 통해 이미 매크로를 경험했지만 탐구하지는 못했습니다. 매크로는 크게 다움의 세가지 방식으로 사용됩니다.

- 구조체 및 열거형에 사용되는 속성 `#[derive]`와 함께 추가된 코드를 지정하는 사용자 지정 매크로
- 모든 항목에 사용할 수 있는 사용자 정의 속성을 정의하는 속성 유사 매크로
- 함수 호출처럼 보이지만 인수로 지정된 토큰에서 작동하는 함수 유사 매크로

### 매크로와 함수의 차이점

매크로는 기본적으로 메타프로그래밍으로 알려진 코드를 생성하는 방법입니다. `println!` 및 `vec!` 매크로는 컴파일 시 매크로에 의해 수동으로 작성된 코드보다 더 많은 코드를 생성하도록 확장됩니다.

함수 서명은 함수가 가진 매개변수의 수와 유형이 결정되어 있습니다. 반면에 매크로는 다양한 수의 매개변수를 사용해서 컴파일 시점에서 코드로 변환할 수 있습니다.

그러나 매크로의 단점은 함수로 작성하는 것보다 매크로로 작성하는 것이 더 복잡합니다. 이런 이유는 간접 참조로 인한 것이며 일반적으로 매크로 정의는 일반적으로 함수 정의보다 읽고, 이해하고, 유지 관리하기가 어렵습니다.

### 일반 메타프로그래밍을 위한 선언적 매크로 macro_rules!

`vec!`은 선언적 매크로의 일반적인 예입니다. `macro_rules!`를 이용해 선언적 매크로를 만들 수 있습니다.

```rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

이는 다음의 코드로 변환됩니다.

```rust
{
    let mut temp_vec = Vec::new();
    temp_vec.push(1);
    temp_vec.push(2);
    temp_vec.push(3);
    temp_vec
}
```

### 속성에서 코드를 생성하기 위한 절차적 매크로

속성을 통해 매크로의 동작을 부여할 수 있습니다.

```rust
use proc_macro;

#[some_attribute]
pub fn some_name(input: TokenStream) -> TokenStream {
}
```
`some_name()`의 인자가 소스 코드의 입력이 되고 반환이 매크로가 생성하는 코드가 됩니다.

### 사용자 정의 derive 매크로를 작성하는 방법

먼저 결과 코드를 만들어 봅시다.

```rust
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}
```

이 코드는 `Hello, Macro! My name is Pancakes!`를 인쇄하는 것을 목표로 합니다.

먼저 새로운 라이브러리 크레이트를 생성합니다.

```shell
$ cargo new hello_macro --lib
```

| src/lib.rs
```rust
pub trait HelloMacro {
    fn hello_macro();
}
```

속성을 통해 매크로 생성을 이용하기 않고 일반적으로 구현하는 방법은 다음과 같습니다.

```rust
use hello_macro::HelloMacro;

struct Pancakes;

impl HelloMacro for Pancakes {
    fn hello_macro() {
        println!("Hello, Macro! My name is Pancakes!");
    }
}

fn main() {
    Pancakes::hello_macro();
}
```

하지만 이 방식은 매번 트레잇을 구현해야 한다는 문제점이 있습니다.

추가적으로 라이브러리 크레이트를 생성합니다.

```rust
$ cargo new hello_macro_derive --lib
```

그리고 `proc-macro`를 활성화 합니다.

```toml
proc-macro = true

[dependencies]
syn = "1.0"
quote = "1.0"
```

| hello_macro_derive/src/lib.rs
```rust
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;
use syn;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // Construct a representation of Rust code as a syntax tree
    // that we can manipulate
    let ast = syn::parse(input).unwrap();

    // Build the trait implementation
    impl_hello_macro(&ast)
}
```

`impl_hello_macro()`함수가 아직 구현되지 않았으므로 컴파일 되지 않습니다.

`impl_hello_macro()`에 전달되게 될 구조는 다음과 같습니다.

```yaml
DeriveInput {
    // --snip--

    ident: Ident {
        ident: "Pancakes",
        span: #0 bytes(95..103)
    },
    data: Struct(
        DataStruct {
            struct_token: Struct,
            fields: Unit,
            semi_token: Some(
                Semi
            )
        }
    )
}
```

이제 `impl_hello_macro()`함수를 구현해 봅시다.

```rust
fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident;
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };
    gen.into()
}
```

### 속성과 유사한 매크로

다음의 코드에서 속성은,

```rust
#[route(GET, "/")]
fn index() {
```

다음의 형식으로 처리될 수 있습니다.

```rust
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
```

첫번째 `TokenStream`은 `GET, "/"`을 의미하고 두번째 `TokenStream`은 `fn index() {}`을 해석할 수 있도록 하는 토큰 스트림이 됩니다.

### 기능과 유사한 매크로

`macro_rules!`를 사용해 `println!`와 같은 매크로를 생성할 수 있습니다.

```rust
let sql = sql!(SELECT * FROM posts WHERE id=1);
```

위의 코드는 `sql!` 매크로가 제대로 구현이 된다면 정상적인  Rust 코드가 됩니다. 위의 기능 유사 매크로는 다음의 형식을 이용할 수 있습니다.

```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
```

## 정리
이번 장에서 고급 Rust 기능들을 살펴봤습니다. 