## Rust #15: 15장 스마트 포인터

## 개요
Rust는 스마트 포인터를 이용해 다양한 기능을 제공합니다. Rust 문서에서는 String과 Vec<T>도 일종의 스마트 포인터라고 정의하며, Rust의 컴파일 시점 소유권에 더해서 런타임 시 소유권을 관리하는 방법을 알려줍니다.
Rust에서 스마트 포인터는 `Deref`와 `Drop` 트레잇을 구현했는가로 정의합니다. 이 장에서는 이 두 트레잇을 다루고 스마트 포인터에서 왜 중요한지를 설명합니다.

이 장에서는 Rust 표준 라이브러리에서 제공하는 다음의 스마트 포인터를 다룹니다.

- 값을 힙에 할당하기 위한 Box<T>
- 복수개의 소유권을 가능하게 하는 참조 카운터 유형인 Rc<T>
- 빌림 규칙을 컴파일 타임이 아니라 런타임에 강제하는 유형인 RefCell<T>를 통해 접근 가능한 Ref<T>와 RefMut<T>

그리고 불변 유형의 내부 값을 변경하기 위한 내부 가변성(interior mutability) 패턴에 대해 다룹니다. 그리고 *참조 순환 (reference cycles)*이 어떤 이유로 메모리 누수가 발생하는지, 그리고 이것을 어떻게 방지하는지 설명합니다.

## Box<T>를 사용하여 힙의 데이터 가리키기

Box<T>는 데이터를 스택이 아니라 힙에 저장할 수 있도록 합니다. 사용하는 스택 정보는 힙 주소를 가리키는 포인터입니다.

Box<T>를 이용하는 것은 데이터를 스택이 아닌 힙에 저장한다는 점 빼고는 성능 오버헤드가 없습니다. 다음의 이유로 Rust에서 자주 사용하게 됩니다.

- 컴파일 타임시 크기를 알 수 없는 유형이 있고 정확한 크기가 필요한 컨텍스트에서 해당 유형의 값을 사용하려는 경우 (포인터 사이즈가 크기가 됨)
- 많은 양의 데이터가 있고 소유권을 이전하고 싶지만 그렇게 할 때 데이터가 복사되지 않아야 하는 경우
- 값을 소유하고 이 값의 특정 유형이 아니라 특정 트레잇을 구현하는 유형으로만 접근하고자 할 때

### Box<T>를 사용하여 힙에 데이터 저장하기

다음의 코드는 힙에 i32형의 값을 저장합니다.

```rust
fn main() {
    let b = Box::new(5);
    println!("b = {}", b);
}
```

b는 `Box::new()`의해 숫자 5를 스택이 아닌 힙에 저장합니다. 마치 스택에 있는 것과 유사한 방식으로 박스 내의 데이터에 접근할 수 있습니다.

### 박스는 재귀적 유형을 가능하게 합니다

컴파일러는 컴파일 시점에서 유형이 얼마만큼의 공간을 차지하는지를 알아야 합니다. 그런데 자기신을 자기가 참조하는 재귀적 유형일 경우 유형의 크기를 컴파일러가 알 수가 없습니다. 이를 확인하기 위해 cons list로 탐험할 것입니다. cons list 유형은 재귀적 구조를 제외하면 직관적입니다.

### Cons List에 대한 더 많은 정보

cons list는 Lisp 프로그래밍 언어 및 파생 언어에서 유래된 데이터 구조입니다. construct function의 줄임말인 cons 함수는 두개의 인자를 받아 새로운 한 쌍을 만드는데, 이는 단일 값과 또다른 쌍입니다. 이런 방법으로 리스트를 구성할 수 있습니다.

다음은 cons list를 표현합니다.

```rust
enum List {
    Cons(i32, List),
    Nil,
}
```

`List` 는 i32와 그 다음 List로 구성된 `Cons` 이거나 아무것도 아닌 `Nil`로 구성됩니다. 위의 List를 통해 다음의 코드를 전개할 수 있습니다.

```rust
use List::{Cons, Nil};

fn main() {
    let list = Cons(1, Cons(2, Cons(3, Nil)));
}
```

하지만 재귀적 구조에 의해 Rust 컴파일러에서 `List`의 크기를 알 수 없게 되어 컴파일 오류가 발생합니다.

```shell
error[E0072]: recursive type `List` has infinite size
 --> src/main.rs:1:1
  |
1 | enum List {
  | ^^^^^^^^^ recursive type has infinite size
2 |     Cons(i32, List),
  |               ----- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to
  make `List` representable
```

###  비재귀적 유형의 크기 계산하기

앞전 6장에서 열거형 정의에 대해 이야기 할 때 봤던 열거형입니다.

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

재귀적 구조가 아니므로 `Message`는 Rust 컴파일러에 의해 크기를 알 수 있으므로 정상적으로 컴파일 됩니다. 구조체로 메모리에 값을 할당하기 위해서는 크기가 있어야 하기 때문입니다.

앞의 Cons list는 다음 처럼 그 크기를 알 수 없습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633603865892/j7dftEs3H.png)

이를 해결하기 위해서는 비재귀적 유형으로 변경해야 하는데요, 다음의 코드 처럼 수정할 수 있습니다.

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let list = Cons(1,
        Box::new(Cons(2,
            Box::new(Cons(3,
                Box::new(Nil))))));
}
List
```

이제 `List`는 자기자신을 참조하는게 아닌 Box<List>를 참조하게 되고 이는 힙 메모리를 가리키는 포인터이므로 그 크기를 Rust 컴파일러에서 알 수 있게 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633603945371/mJ-42nc7h.png)

## Deref 트레잇으로 스마트 포인터를 일반 참조자와 같이 취급하기

Deref 트레잇을 통해 역참조 연산자의 동작을 커스터마이징 할 수 있습니다.

```rust
fn main() {
    let x = 5;
    let y = &x;

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

모든 테스트가 통과됩니다. 그런데, `assert_eq!(5, *y)` 대신 `assert_eq!(5, y)`를 했을 경우 컴파일 오류가 발생합니다. y는 x의 참조자이기 때문입니다.

```shell
error[E0277]: the trait bound `{integer}: std::cmp::PartialEq<&{integer}>` is
not satisfied
 --> src/main.rs:6:5
  |
6 |     assert_eq!(5, y);
  |     ^^^^^^^^^^^^^^^^^ can't compare `{integer}` with `&{integer}`
  |
  = help: the trait `std::cmp::PartialEq<&{integer}>` is not implemented for
  `{integer}`
```

### Box<T>를 참조자 처럼 사용하기

다음의 코드를 보시죠

```rust
fn main() {
    let x = 5;
    let y = Box::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

y가 참조자일 때 했던 것과 동일한 방식으로 박스 포인터 앞에 역참조 연산자를 사용할 수 있습니다.

### 우리만의 스마트 포인터 정의하기

우리만의 스마트 포인터를 정의해 나가면서 동작을 통해 스마트 포인터를 이해하는 시간은 가져봅시다. 다음의 코드를 보시죠.

```rust
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}
```

이는 MyBox를 생성해서 어떠한 유형이든 인자로 받아 그것을 보유하고 MyBox 인스턴스를 반환합니다.

Box<T> 처럼 구현한 아래의 코드를 보시죠.
```rust
fn main() {
    let x = 5;
    let y = MyBox::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

역참조 부분에서 컴파일 오류가 발생합니다. 이유는 MyBox를 역참조 했을 때 어떤 값이어야 하는지를 알 수 없기 때문인데요. Deref 트레잇을 통해 그것을 알려줄 수 있게 됩니다.

```rust
use std::ops::Deref;

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0
    }
}
```

이제 Deref 트레잇이 구현되었으므로, `assert_eq!(5, *y)`는 컴파일 오류가 발생하지 않고 MyBox가 보관하고 있는 값을 반환하여 사용할 수 있게 됩니다. 이는 `*y`가 `*(y.deref())`으로 컴파일에 의해 해석되기 때문입니다.

### 함수 및 메서드를 사용한 암시적 역참조 강제

역참조 강제는 Deref를 구현한 유형의 참조자를 원래 유형의 참조자로 변환해 줍니다. 이를 통해 다음의 코드가 정상 컴파일 됩니다.

```rust

fn hello(name: &str) {
    println!("Hello, {}!", name);
}
```

```rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&m);
}
```

hello에 인자로 전달한 m은 &MyBox<String> 이지만 역참조 강제에 의해 &String으로 변환되고 String은 역참조 강제에 의해 str로 변환되 &str로 전달할 수 있게 됩니다.

만약 역참조 강제 기능이 없었다면 다음 처럼 코딩해야 했을 것입니다.

```rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&(*m)[..]);
}
```

### 역참조 강제가 가변성과 상호작용 하는 방법

불변 참조자에 대한 `*`를 오버라이딩 하기 위해 `Deref` 트레잇을 구현하는 방법과 비슷하게, 가변 참조자에 대한 `*`를 오버라이딩 하기 위한 `DerefMut` 트레잇을 제공합니다.

- `T: Deref<Target=U>` 일 때 &T에서 &U로
- `T: DerefMut<Target=U>` 일 때 `&mut T`에서 `&mut T`로
- `T: Deref<Target=U>` 일 때 &mut T에서 `&U`로

Rust는 가변 참조자를 불변 참조자로 역참조 강제할 수 있습니다. 그러나 반대는 빌림 규칙때문에 불가능 합니다.

## Drop 트레잇은 메모리 정리 코드를 실행합니다.

스마트 포인터를 이용하게 되면 구조체 인스턴스가 범위 밖으로 벗어날 때 구조체에서 메모리 해제 작업을 해줘야만 합니다. 가장 대표적인 예가 Box<T>가 범위 밖으로 벗어날 때 힙 메모리를 해제하는 것입니다. 그것을 가능하게 하는 것이 Drop 트레잇 입니다.
Drop 트레잇은 인스턴스가 범위 밖으로 벗어나 해제되어야 할 때 Drop 트레잇을 구현하면 Drop 메소드가 호출이 되어 내부 메모리를 해제할 수 있는 기회를 가질 수 있게 됩니다.

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("my stuff") };
    let d = CustomSmartPointer { data: String::from("other stuff") };
    println!("CustomSmartPointers created.");
}
```

### std::mem::drop을 이용해 값을 일찍 버리기

범위 밖으로 벗어나기 전에 강제로 스마트 포인터를 해제 시킬 수도 있습니다.

```rust
fn main() {
    let c = CustomSmartPointer { data: String::from("some data") };
    println!("CustomSmartPointer created.");
    c.drop();
    println!("CustomSmartPointer dropped before the end of main.");
}
```

그러나 위의 코드는 컴파일 오류가 됩니다. 이유는 Rust는 수동으로 `c.drop()`을 호출할 수 없도록 강제하기 떄문입니다.

```shell
error[E0040]: explicit use of destructor method
  --> src/main.rs:14:7
   |
14 |     c.drop();
   |       ^^^^ explicit destructor calls not allowed
```

이는 `c.drop()`을 호출한다 하더라고 러스트는 범위 밖으로 나갈 때 여전히 `drop()`을 호출할 것이기 때문입니다. 대신 차음 처럼 할 수 있습니다.

```rust
fn main() {
    let c = CustomSmartPointer { data: String::from("some data") };
    println!("CustomSmartPointer created.");
    drop(c);
    println!("CustomSmartPointer dropped before the end of main.");
}
```

이제 `drop(c)`의 시점에서 c는 메모리를 해제하고 더이상 컴파일 시점에서 c를 사용하지 못하도록 해줍니다.

## Rc<T>,  참조 카운트 스마트 포인터

Rust에서 제공하는 컴파일 타임의 소유권 규칙을 이용해 대부분의 로직을 구현할 수 있습니다. 그렇지만 여러개의 소유자가 하나의 값을 가지는 경우도 있습니다. 이럴 떄 `Rc<T>`를 사용할 수 있습니다.

### Rc<T>를 사용하여 데이터 공유하기

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633605539754/ykOxe6GcS.png)

위의 도식을 보면 b가 가리키는 구조가 [5, Box<List>]를, a 또한 동일한 [5, Box<List>]를, c가 가리키는 구조가 동일한 [5, Box<List>]를 가리키는 구조임을 알 수 있습니다.

도식대로 코드를 전개하면 다음과 같습니다.

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let a = Cons(5,
        Box::new(Cons(10,
            Box::new(Nil))));
    let b = Cons(3, Box::new(a));
    let c = Cons(4, Box::new(a));
}
```

Rust는 하나의 소유자만 값을 소유할 수 있으므로 위의 코드는 컴파일 오류가 발생합니다.

```shell
error[E0382]: use of moved value: `a`
  --> src/main.rs:13:30
   |
12 |     let b = Cons(3, Box::new(a));
   |                              - value moved here
13 |     let c = Cons(4, Box::new(a));
   |                              ^ value used here after move
   |
   = note: move occurs because `a` has type `List`, which does not implement
   the `Copy` trait
```

이럴 때 `Rc<T>`를 `Box<T>` 대신 사용할 수 있습니다.

```rust
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
}
```

여기서 `Rc::clone(&a)`는 값을 복사하는게 아니라 값을 참조하는 횟수를 증가하게 됩니다. 이로써 총 3개의 소유자가 a를 참조할 수 있게 됩니다.

### Rc<T>의 클론 생성은 참조 카운트를 증가 시킵니다.

다음의 코드를 통해 참조 횟수를 확인할 수 있습니다.

```rust
fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    println!("count after creating a = {}", Rc::strong_count(&a));
    let b = Cons(3, Rc::clone(&a));
    println!("count after creating b = {}", Rc::strong_count(&a));
    {
        let c = Cons(4, Rc::clone(&a));
        println!("count after creating c = {}", Rc::strong_count(&a));
    }
    println!("count after c goes out of scope = {}", Rc::strong_count(&a));
}
```

불변 참조자를 통하여 Rc<T>는 읽기 전용으로 여러 부분에서 데이터를 공유할 수 있도록 허용합니다. 그러나 대상이 가변일 경우 빌림 규칙을 위반할지도 모릅니다. 하지만 데이터의 변형을 가능하게 하는 것은 매우 유용한 기능이죠. 다음절에서 내부 가변성 패턴과 이를 가능하게 하는 RefCell<T> 유형에 대해 논의할 것입니다.

## RefCell<T>와 내부 가변성 패턴

어떤 데이터에 대한 불변 참조자라 하더라도 내부 가변성 패턴을 이용해 데이터를 변형할 수 있게 해줍니다. 이를 위해서는 `unsafe`를 사용해야 한는데 19장에서 배울 예정입니다.

내부 가변성 패턴을 따르는 `RefCell<T>` 타입을 살펴보는 것으로 이 개념을 탐구해 봅시다.

### RefCell<T>를 가지고 런타임에 빌림 규칙 시행

Rc<T>는 여러개의 소유권을 가능하게 하지만 RefCell<T>는 오직 단일 소유권을 나타냅니다. 그렇다면 Box<T>랑은 어떤 차이점이 있을까요?

- 주어진 시간(런타임) 시간에 하나의 변경 가능한 참조 또는 임의의 수의 변경 불가능한 참조 중 하나 (둘 모두는 아님)을 가질 수 있게 합니다.
- 참조는 항상 유효(해야)합니다.

참조와 Box<T>를 이용하면 빌림 규칙의 불변성은 컴파일 시점에서 결정됩니다. RefCell<T>를 이용하면 컴파일 시점이 아니라 런타임 시점에서 이를 시행합니다. 참조자를 가지고 이 규칙을 어기면 컴파일 오류가 발생합니다. 이에 반해 RefCell<T>을 이용했을 때 이 규칙을 어기면 런타임 오류(panic!)를 일으키고 프로그램을 종료하게 됩니다.

Box<T>, Rc<T> 및 RefCell<T>를 선택하는 방법은 다음을 따릅니다.
- Rc<T>는 동일 데이터에 대해 복수개의 소유자가 필요할 때 사용합니다. Box<T> 및 RefCell<T>는 단일 소유자만 가능합니다.
- Box<T>는 컴파일 시점에서 검사된 불변 또는 가변 빌림을 허용합니다. Rc<T>는 컴파일 타임에 검사된 불변 빌리만 허용합니다. RefCell<T>는 런타임에 검사된 불변 또는 가변 빌림을 허용합니다.
- RefCell<T>는 런타임에 검사된 가변 빌림을 허용해서, RefCell<T>은 불변이 때라도 RefCell<T> 내부의 값을 변경할 수 있습니다.

불변값 내부의 값을 변경하는 것을 내부 가변성 패턴이라고 합니다.

### 내부 가변성: 불변값에 대한 가변 빌림

다음의 코드를 보시죠.

```rust
fn main() {
    let x = 5;
    let y = &mut x;
}
```

x는 불변이므로 컴파일 오류가 발생합니다.

```shell
error[E0596]: cannot borrow immutable local variable `x` as mutable
 --> src/main.rs:3:18
  |
2 |     let x = 5;
  |         - consider changing this to `mut x`
3 |     let y = &mut x;
  |                  ^ cannot borrow mutably
```

하지만 값이 해당 메서드에 의해 변경 되지만 외부의 코드에서는 변경할 수 없는 것처럼 동작하는게 유용할 때가 있는데, 이때 `RefCell<T>`을 쓸 수 있습니다.

#### 내부 변경 가능성에 대한 사용 사례: mock 객체

테스트 더블은 테스트 하는 동안 다른 타입으로 대신 테스트를 진행하는 개념입니다. 이 때 사용하는 것이 mock 개체입니다. mock 개체는 형태는 동일하지만 어떠한 기능을 하지 않고, 그 형태가 실 개체와 정확히 같아 이후 테스트 코드를 수정하지 않고도 테스트를 진행할 수 있게 합니다.

다음의 코드를 보시죠.

```rust
pub trait Messenger {
    fn send(&self, msg: &str);
}

pub struct LimitTracker<'a, T: 'a + Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T>
    where T: Messenger {
    pub fn new(messenger: &T, max: usize) -> LimitTracker<T> {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }

    pub fn set_value(&mut self, value: usize) {
        self.value = value;

        let percentage_of_max = self.value as f64 / self.max as f64;

        if percentage_of_max >= 0.75 && percentage_of_max < 0.9 {
            self.messenger.send("Warning: You've used up over 75% of your quota!");
        } else if percentage_of_max >= 0.9 && percentage_of_max < 1.0 {
            self.messenger.send("Urgent warning: You've used up over 90% of your quota!");
        } else if percentage_of_max >= 1.0 {
            self.messenger.send("Error: You are over your quota!");
        }
    }
}
```

어떤 값이 최대값에 얼마나 근접하는지를 추적해 특정 수준이 되면 경고를 보내주는 코드입니다.

우리는 Messenger를 구현하기 전에 아무런 동작을 하지 않는 Mock Messenger를 만들고 `LimitTracker`의 `set_value()`를 테스트하고자 합니다.

다음 처럼 Mock Messenger를 구현해 봅시다.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    struct MockMessenger {
        sent_messages: Vec<String>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger { sent_messages: vec![] }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);

        limit_tracker.set_value(80);

        assert_eq!(mock_messenger.sent_messages.len(), 1);
    }
}
```

그런데 문제가 있습니다. `MockMessenger` 메시지를 추적하기 위해 메시지를 보관하는데, 다음처럼 컴파일 오류가 발생합니다.

```shell
error[E0596]: cannot borrow immutable field `self.sent_messages` as mutable
  --> src/lib.rs:52:13
   |
51 |         fn send(&self, message: &str) {
   |                 ----- use `&mut self` here to make mutable
52 |             self.sent_messages.push(String::from(message));
   |             ^^^^^^^^^^^^^^^^^^ cannot mutably borrow immutable field
```

이는 `send()`의 &self에 의해서 필드의 값을 변경할 수 없기 때문입니다. 이를 RefCell<T>를 이용해 다음과 같이 해결할 수 있습니다.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger { sent_messages: RefCell::new(vec![]) }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.borrow_mut().push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        // --snip--

        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```

이제 MockMessenger는 불변 객체 처럼 취급되지만 `RefCell<T>`에 의해 내부 필드의 값을 변경할 수 있게 되었습니다.

#### RefCell<T>는 런타임에 빌림을 추적합니다.

다음의 코드를 보시죠.

```rust
impl Messenger for MockMessenger {
    fn send(&self, message: &str) {
        let mut one_borrow = self.sent_messages.borrow_mut();
        let mut two_borrow = self.sent_messages.borrow_mut();

        one_borrow.push(String::from(message));
        two_borrow.push(String::from(message));
    }
}
```

동일한 범위에서 두개의 가변 참조자를 만들었으므로 이는 런타임 오류를 발생시킵니다.

```shell
---- tests::it_sends_an_over_75_percent_warning_message stdout ----
    thread 'tests::it_sends_an_over_75_percent_warning_message' panicked at
    'already borrowed: BorrowMutError', src/libcore/result.rs:906:4
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

### Rc<T>와 RefCell<T>를 조합하여 가변 데이터의 복수 소유자 만들기

Rc<T>와 RefCell<T>를 조합하면 가변 데이터의 복수 소유자를 만들 수가 있습니다.

```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(10)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```

출력:
```shell
a after = Cons(RefCell { value: 15 }, Nil)
b after = Cons(RefCell { value: 6 }, Cons(RefCell { value: 15 }, Nil))
c after = Cons(RefCell { value: 10 }, Cons(RefCell { value: 15 }, Nil))
```

## 순환 참조는 메모리 릭을 발생시킬 수 있습니다.
`RefCell<T>`를 이용해 런타임에서 소유권을 다룰 수 있게 되었습니다. 이것의 의미는 프로그래머에 의해서 잘못된 사용이 생길 수 있다는 것인데요, 대표적인게 순환 참조입니다.

### 순환 참조 만들기

순환 참조를 만들어 봄으로써 Rust에서도 메모리 릭이 발생할 수 있음을 확인할 수 있습니다.

먼저 리스트를 다음과 같이 만듭시다.

```rust
use std::rc::Rc;
use std::cell::RefCell;
use List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match *self {
            Cons(_, ref item) => Some(item),
            Nil => None,
        }
    }
}
```

`RefCell<RC<List>>`를 썼으므로 복수개의 소유자가 있고 그것을 수정할 수 있는 구조를 만들었습니다.

아래의 코드는 순환 참조를 일으킵니다.

```rust
fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));

    println!("a initial rc count = {}", Rc::strong_count(&a));
    println!("a next item = {:?}", a.tail());

    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));

    println!("a rc count after b creation = {}", Rc::strong_count(&a));
    println!("b initial rc count = {}", Rc::strong_count(&b));
    println!("b next item = {:?}", b.tail());

    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }

    println!("b rc count after changing a = {}", Rc::strong_count(&b));
    println!("a rc count after changing a = {}", Rc::strong_count(&a));

    // Uncomment the next line to see that we have a cycle;
    // it will overflow the stack
    // println!("a next item = {:?}", a.tail());
}
```

맨 아래 주석을 제거하면 다음처럼 `a.tail()`에 의해 순환 참조 구조로 인해 끝나지 않는 동작을 하게 되고 결국에는 스택 오버플로우가 발생합니다.

```shell
...
RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell { value: Cons(5, RefCell { value: Cons(10, RefCell {
thread 'main' has overflowed its stack
```

다시 주석 처리를 하면 다음 처럼 출력됩니다.

```shell
a initial rc count = 1
a next item = Some(RefCell { value: Nil })
a rc count after b creation = 2
b initial rc count = 1
b next item = Some(RefCell { value: Cons(5, RefCell { value: Nil }) })
b rc count after changing a = 2
a rc count after changing a = 2
```

인위적으로 a와  b의 Rc<List> 인스턴스의 참조 카운드를 증가시켰고, main의 끝에서 정리가 될 텐데, 여전히 a와 b가 각각 RC<List> 인스턴스 내의 카운트가 1일 것이고, 메모리 릭이 발생하게 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633614074876/vlBHZmA17.png)

### 참조 순환 방지하기: Rc<T>를 Weak<T>로 바꾸기
`Weak<T>`는 약한 참조를 합니다. Rc<T>는 강한 참조 카운트를 통해 0일 되었을 때 비로서 메모리를 해제하는 데 반해, `Weak<T>`는 약한 참조로 그 카운트가 0이 아니더라도 메모리를 해제하는 차이가 있습니다.

강한 참조는 Rc<T> 인스턴스의 소유권을 공유할 수 있는 방법입니다. 약한 참조는 소유권 관계를 표현하지 않습니다.  이것은 순환 참조를 만들지 않기 때문에 유용합니다. 다만, `Weak<T>`는 참조하는 값이 이미 해제되었을 수 있으므로 반드시 사용하기 전에 `upgrade` 메소드를 통해 확인이 필요합니다.

#### 트리 데이터 구조 만들기: 자식 노드를 가진 Node

트리 구조를 만들어 봅시다.

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    children: RefCell<Vec<Rc<Node>>>,
}
```

`i32` 유형 값을 가진 노드는 수정 가능한 `Rc<Node>` 목록을 가집니다.

```rust
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        children: RefCell::new(vec![]),
    });

    let branch = Rc::new(Node {
        value: 5,
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });
}
```

위의 코드에서 보다시피 `leaf`을 자식으로 가지는 `brance`를 만들 수 있게 됩니다. 이를 통해 `branch`에서 `children`을 통해 `leaf`에 접근할 수 있습니다. 그러나 반대의 경우인 `leaf`에서 `children`으로 접근할 방법이 없습니다. 이를 더 구현해 봅시다.

### 자식으로부터 부모로 가는 참조자 추가하기

자식 노드가 부모를 알기 위해 `parent` 필드를 추가 할 필요가 있습니다.

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}
```

그런데 parent에서 Rc<Node>를 사용할 수 는 없는데 그렇게 되면 순환 참조가 되기 때문입니다. 이것을 `Weak<Node>`로 해서 순환 참조를 해결할 수 있습니다.

노드는 이제 부모 노를 참조할 수는 있지만 소유하지는 않게 되었습니다. 이제 `main()`을 다시 수정해 봅시다.

```rust
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());

    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });

    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
}
```

첫 `println!()`로 다음과 같이 출력됩니다.

```shell
leaf parent = None
```

그리고 두번째 `println!()`는 다음과 같이 출력됩니다.

```shell
leaf parent = Some(Node { value: 5, parent: RefCell { value: (Weak) },
children: RefCell { value: [Node { value: 3, parent: RefCell { value: (Weak) },
children: RefCell { value: [] } }] } })
```

무한 출력이 되지 않는다는 것은 이 코드가 순환 참조를 하지 않는것을 의미합니다.

#### strong_count와 weak_count의 변화 시각화하기

코드를 통해 strong_count와 weak_count가 어떻게 변화 하는지를 살펴봅시다.

```rust
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf),
    );

    {
        let branch = Rc::new(Node {
            value: 5,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(vec![Rc::clone(&leaf)]),
        });

        *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

        println!(
            "branch strong = {}, weak = {}",
            Rc::strong_count(&branch),
            Rc::weak_count(&branch),
        );

        println!(
            "leaf strong = {}, weak = {}",
            Rc::strong_count(&leaf),
            Rc::weak_count(&leaf),
        );
    }

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf),
    );
}
```

## 정리

이 장에서는 일반적인 참조자를 통한 방식과 다른 방식으로 스마트 포인터를 사용하는 방법을 다루었습니다.
또 Deref 및 Drop 트레잇을 다루었는데 이는 내가 만든 구조체가 스마트 포인터라는 것을 의미합니다. 또한 우리는 메모리 릭을 발생시킬 수 있는 순환 참조를 없애기 위해 `Weak<T>`를 이용해 이를 방지하는 방법도 탐구하였습니다.
