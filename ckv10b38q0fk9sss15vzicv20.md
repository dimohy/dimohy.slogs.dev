---
title: "Rust #17: 17장 Rust의 객체지향 프로그래밍 기능"
datePublished: Thu Oct 21 2021 13:55:17 GMT+0000 (Coordinated Universal Time)
cuid: ckv10b38q0fk9sss15vzicv20
slug: rust-17-17-rust
tags: rust

---

## 개요
Rust는 객체 지향 프로그래밍의 일부 정의에서는 객체 지향으로 분류하지만 다른 정의는 그렇지 않습니다. 이 장에서는 객체 지향으로 간주하는 특정 특성과 이것을 이용해서 관용적 Rust로 변환되는 방식을 알아볼 것입니다. 그런 다음 Rust에서 객체 지향 디자인 패턴을 구현하는 방법에 대해 보여주고 Rust의 장점을 이용해서 구현하는 방식의 장단점에 대해 논의하겠습니다.

## 객체지향 언어의 특징

### 객체에는 데이터와 동작이 포함됩니다.

객체는 객체의 목적에 부합하는 데이터(상태, 속성) 즉 필드와, 행위(동작) 즉 메소드를 포함합니다. Rust는 디자인 패턴 `Gang of Four` 책에 의하면 객체 지향 프로그래밍 언어로 분류할 수 있습니다.

### 구현 세부 사항을 숨기는 캡슐화

Rust는 또한 `pub` 키워드를 통해 외부로 노출하거나 노출하지 않도록 필드 또는 메소드를 조정할 수 있습니다. 그런 의미에서 Rust는 객체 지향 언어 입니다.

```rust
pub struct AveragedCollection {
    list: Vec<i32>,
    average: f64,
}
```

`AveragedCollection`은 구조체 내의 필드는 비공개입니다.

```rust
impl AveragedCollection {
    pub fn add(&mut self, value: i32) {
        self.list.push(value);
        self.update_average();
    }

    pub fn remove(&mut self) -> Option<i32> {
        let result = self.list.pop();
        match result {
            Some(value) => {
                self.update_average();
                Some(value)
            }
            None => None,
        }
    }

    pub fn average(&self) -> f64 {
        self.average
    }

    fn update_average(&mut self) {
        let total: i32 = self.list.iter().sum();
        self.average = total as f64 / self.list.len() as f64;
    }
}
```

하지만 비공개 필드를 이용한 메소드는 공개로 외부에 노출됩니다.

### 유형 시스템 및 코드 공유로서의 상속

상속은 재정의 할 필요 없이 상위 객체의 데이터와 동작을 얻을 수 있는 메커니즘입니다. Rust는 상속을 지원하지 않으므로 상속의 관점에서는 객체 지향 프로그래밍 언어가 아닙니다. 상속을 이용하는 이유는 두 가지 인데 코드 재사용과 유형 시스템을 이용하기 위함 입니다. 유형 시스템은 다른 말로 다형성 이라고 하며 상위 객체 타입으로 접근하고자 할 때 유용한 로직 구조가 있는데 상속을 통해 이를 달성할 수 있습니다.
Rust는 상속을 지원하지 않는 대신 상속을 통해 얻을 수 있는 두 가지 이점을 트레잇을 통해 모두 제공합니다. 

### 다른 유형의 값을 허용하는 트레잇 개체 사용

Rust의 컬렉션인 벡터(Vec)는 한가지 유형만을 지원하는 한계가 있습니다. 이는 컴파일 시점에서 벡터를 최적화 하기 위함입니다. 하지만 때때로 특정 상황에서 벡터와 같은 유형 세트를 확장 했을 때 유용한 경우가 있습니다. 가령 UI의 그리기 행위가 그렇습니다. 그리기 위한 도형의 행위는 모두 다르지만 유형 세트에서는 동일하게 바라봐야 유용합니다.

### 일반적인 행동에 대한 특성 정의

Rust에서도 `dyn` 키워드를 이용해 동적 다형성을 구현할 수 있습니다.

다음처럼 `Draw` 트레잇을 정의하고,

```rust
pub trait Draw {
    fn draw(&self);
}
```

그런 후 그리기 트레잇을 벡터로 보관하는 `Screen`을 정의하고,

```rust
pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}
```

다음처럼 동적 다형성을 이용해 Screen에 포함되어 있는 Draw 트레잇 개체들을 순회하면서 그릴 수 있습니다.

```rust
impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

그리고 다음의 코드처럼 정적 다형성과 다르다는 것을 알 수 있습니다.

```rust
pub struct Screen<T: Draw> {
    pub components: Vec<T>,
}

impl<T> Screen<T>
where
    T: Draw,
{
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

위의 코드는 컴파일 시점에서 제네릭 `T`의 형식이 결정됩니다.

이렇게 Rust는 이것을 통해 정적 다형성(컴파일 시점에서 형이 결정됨)과 동적 다형성(런타임 시점에서 형이 결정됨)을 모두 사용할 수 있음을 알 수 있습니다.

### 트레잇 구현

다음의 코드를 통해 `Button`과 `SelectBox`가 `Draw` 트레잇으로 동적으로 결정되어 호출되는 것을 확인해 봅시다.

```rust
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        // code to actually draw a button
    }
}
```

```rust
use gui::Draw;

struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        // code to actually draw a select box
    }
}
```

두 개의 Draw 트레잇을 구현했고 다음처럼 동적으로 동작을 시킬 수 있습니다.

```rust
use gui::{Button, Screen};

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(SelectBox {
                width: 75,
                height: 10,
                options: vec![
                    String::from("Yes"),
                    String::from("Maybe"),
                    String::from("No"),
                ],
            }),
            Box::new(Button {
                width: 50,
                height: 10,
                label: String::from("OK"),
            }),
        ],
    };

    screen.run();
}
```

하지만 어쨌든 Draw 트레잇을 구현한 대상만 추가할 수 있습니다.

```rust
use gui::Screen;

fn main() {
    let screen = Screen {
        components: vec![Box::new(String::from("Hi"))],
    };

    screen.run();
}
```

```shell
$ cargo run
   Compiling gui v0.1.0 (file:///projects/gui)
error[E0277]: the trait bound `String: Draw` is not satisfied
 --> src/main.rs:5:26
  |
5 |         components: vec![Box::new(String::from("Hi"))],
  |                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Draw` is not implemented for `String`
  |
  = note: required for the cast to the object type `dyn Draw`

For more information about this error, try `rustc --explain E0277`.
error: could not compile `gui` due to previous error
```

### 트레잇 객체는 동적 디스패치를 수행합니다.

하지만 트레잇 객체를 사용하려면 동적으로 대상을 식별해야만 합니다. 이 의미는 정적 디스패치에서는 발생하지 않는 추가적인 런타임 비용이 있다는 것입니다. 꼭 필요한 곳에만 `dyn`을 사용하도록 합시다.

### 트레잇 객체에는 객체 안전이 필요합니다.

- 반환 유형은 Self가 아니어야 합니다.
- 제네릭 인자가 없어야 합니다.

첫번째 이유로 다음의 코드는

```rust
pub trait Clone {
    fn clone(&self) -> Self;
}
```

```rust
pub struct Screen {
    pub components: Vec<Box<dyn Clone>>,
}
```

컴파일 오류가 발생합니다.

```shell
$ cargo build
   Compiling gui v0.1.0 (file:///projects/gui)
error[E0038]: the trait `Clone` cannot be made into an object
 --> src/lib.rs:2:29
  |
2 |     pub components: Vec<Box<dyn Clone>>,
  |                             ^^^^^^^^^ `Clone` cannot be made into an object
  |
  = note: the trait cannot be made into an object because it requires `Self: Sized`
  = note: for a trait to be "object safe" it needs to allow building a vtable to allow the call to be resolvable dynamically; for more information visit <https://doc.rust-lang.org/reference/items/traits.html#object-safety>

For more information about this error, try `rustc --explain E0038`.
error: could not compile `gui` due to previous error
```

## 객체 지향 디자인 패턴 구현하기

블로그 게시물 워크플로를 점진적으로 구현하는 것으로 Rust로 상태 패턴을 구현하는 것을 살펴 보도록 합시다.

1. 블로그 게시물은 빈 초안으로 시작됩니다.
1. 초안이 완료되면 게시물에 대한 검토가 요청됩니다.
1. 게시물이 승인되면 게시됩니다.
1. 게시된 블로그 게시물만 인쇄할 콘텐츠를 반환하므로 승인되지 않은 게시물이 실수로 게시되는 일이 없습니다.

다음의 코드를 봅시다.

```rust
use blog::Post;

fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
    assert_eq!("", post.content());

    post.request_review();
    assert_eq!("", post.content());

    post.approve();
    assert_eq!("I ate a salad for lunch today", post.content());
}
```

게시물을 생성한 후, 리뷰를 요청합니다. 하지만 게시물의 콘텐츠가 외부에 공개되어서는 안됩니다. 이후 게시물을 승인하면 외부에 게시물의 콘텐츠가 공개됩니다.

위의 코드는 아직 실행되지 않습니다.

### Post 정의 및 초안 상태의 새 인스턴스 생성

다음의 코드를 보시죠.

```rust
pub struct Post {
    state: Option<Box<dyn State>>,
    content: String,
}

impl Post {
    pub fn new() -> Post {
        Post {
            state: Some(Box::new(Draft {})),
            content: String::new(),
        }
    }
}

trait State {}

struct Draft {}

impl State for Draft {}
```

Post가 동적인 상태를 가지도록 `dyn` 키워드를 사용하였습니다. 이제 state는 동적 다형성을 가지게 됩니다.

### 게시물 콘텐츠의 텍스트 저장

다음의 코드를 봅시다.

```rust
impl Post {
    // --snip--
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
}
```

콘텐츠에 텍스트를 추가합니다.  그러나 이 동작은 상태에 의존하지 않으므로 상태 패턴을 이용한 것은 아닙니다.

### 초안 게시물의 내용이 비어 있는지 확인

```rust
impl Post {
    // --snip--
    pub fn content(&self) -> &str {
        ""
    }
}
```

초안 상태의 콘텐츠는 항상 비어있어야 하므로 일단 이렇게 전개를 합니다.

### 게시물 검토를 요청하면 상태가 변경됩니다.

다음의 코드를 통해 검토를 요청하면 상태가 변경되도록 합니다.

```rust
impl Post {
    // --snip--
    pub fn request_review(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.request_review())
        }
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<dyn State>;
}

struct Draft {}

impl State for Draft {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        Box::new(PendingReview {})
    }
}

struct PendingReview {}

impl State for PendingReview {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        self
    }
}
```

### content의 동작을 변경하는 approve 메서드 추가

이제 content의 동작을 변경하는 approve 메소드를 추가해봅시다.

```rust
impl Post {
    // --snip--
    pub fn approve(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.approve())
        }
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<dyn State>;
    fn approve(self: Box<Self>) -> Box<dyn State>;
}

struct Draft {}

impl State for Draft {
    // --snip--
    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }
}

struct PendingReview {}

impl State for PendingReview {
    // --snip--
    fn approve(self: Box<Self>) -> Box<dyn State> {
        Box::new(Published {})
    }
}

struct Published {}

impl State for Published {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        self
    }

    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }
}
```

이제 content를 상태에 따라 반환하는 코드로 변경해야 합니다.

```rust
impl Post {
    // --snip--
    pub fn content(&self) -> &str {
        self.state.as_ref().unwrap().content(self)
    }
    // --snip--
}
```

다음 상태에 따라 content를 반환하는 코드를 추가 합시다.

```rust
trait State {
    // --snip--
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        ""
    }
}

// --snip--
struct Published {}

impl State for Published {
    // --snip--
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        &post.content
    }
}
```

상태가 Published일 때만 포스트의 콘텐츠를 반환하는 것을 알 수 있습니다.

### 상태 패턴의 절충

이렇게 러스트에도 상태 패턴을 구현할 수 있습니다. 하지만 디자인 패턴이 Rust에서 선호하는 구현은 아닙니다. Rust는 상속이 없기 때문에 상속 구조를 통해 문제를 푸는 방식이 Rust에 적용하기에는 중복 코드가 발생하고 구조적으로 복잡해질 수 있기 때문입니다.

다음의 절충 방안으로 코드를 개선해 봅시다.

```rust
fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
    assert_eq!("", post.content());
}
```

첫번째 새로운 포스트를 만들고 콘텐츠를 추가했을 때 포스트의 콘텐츠는 외부에 노출되지 않아야 합니다. 이것을 다음 처럼 구현할 수 있습니다.

```rust
pub struct Post {
    content: String,
}

pub struct DraftPost {
    content: String,
}

impl Post {
    pub fn new() -> DraftPost {
        DraftPost {
            content: String::new(),
        }
    }

    pub fn content(&self) -> &str {
        &self.content
    }
}

impl DraftPost {
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
}
```

`Post::new()`를 통해 `DraftPost`객체를 반환합니다. 그리고 `content`는 비공개이고 `content()`로 접근할 수 있는 것은 오직 `Post`일 뿐입니다.

### 다른 유형으로 전환하는 것으로 전환을 구현

`DraftPost`는 `request_review()`를 구현해서 `PendingReviewPost`를 반환하도록 합니다.

```rust
impl DraftPost {
    // --snip--
    pub fn request_review(self) -> PendingReviewPost {
        PendingReviewPost {
            content: self.content,
        }
    }
}

pub struct PendingReviewPost {
    content: String,
}

impl PendingReviewPost {
    pub fn approve(self) -> Post {
        Post {
            content: self.content,
        }
    }
}
```

`PendingReviewPost`는 `approve()`를 통해 최종적으로 `Post`의 인스턴스를 반환합니다. 이제 `Post`의 `Content()`를 통해 콘텐츠에 접근할 수 있게 됩니다.

```rust
use blog::Post;

fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");

    let post = post.request_review();

    let post = post.approve();

    assert_eq!("I ate a salad for lunch today", post.content());
}
```

이런 전략은 디자인 패턴의 `상태 패턴`과 다른 Rust만의 전략입니다. `상태 패턴`을 사용하지 않더라도 승인되기 전 까지 콘텐츠에 접근할 수 없게 하고 그 값을 유지하도록 했습니다. 최종적으로 `승인`되었을 때 콘텐츠에 접근할 수 있는 `Post`가 되는 샘입니다.

이렇게 Rust는 다른 객체 지향 언어의 풀이와는 다른 Rust만의 풀이 방법으로 해당 문제를 해결할 수 있습니다.

### 정리

Rust가 객체 지향 언어라고 생각하거나 생각하지 않거나 상관없이 트레잇 객체를 통해 Rust에서 객체 지향 기능을 얻을 수 있다는 것을 알 수 있었습니다. 일반적인 객체 지향 언어의 풀이 방법과는 사뭇 다르지만 Rust만의 풀이 방법이 있다는 것을 알 수 있었습니다.

