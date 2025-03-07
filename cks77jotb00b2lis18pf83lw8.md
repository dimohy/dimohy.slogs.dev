---
title: "Rust #7: 7장 패키지, 크레이트 및 모듈로 커지는 프로젝트 관리"
datePublished: Wed Aug 11 2021 08:05:26 GMT+0000 (Coordinated Universal Time)
cuid: cks77jotb00b2lis18pf83lw8
slug: rust-7-7
tags: rust

---

## 개요
작성중인 코드 크기가 커지게 되면 작업 위치를 찾기가 어려워집니다. 그렇기 때문에 코드를 적절하게 모듈화 해야 하는데요,  이번 장은 Rust에서 필요한 기능을 크레이트를 통해 이용하고, 패키지를 모듈화 하는 방법을 알아보도록 합시다.

## 정의
먼저  Rust에서 사용하는 용어에 대한 간단한 정의부터 알아봅시다.

- 패키지(Package): 기능 집합을 제공하는 하나 이상의 크레이트. Cargo로 생성하며 빌드하고 테스트 할 수 있음
- 크레이트(Crate): 라이브러리 또는 실행 파일 단위. 모듈 트리로 구성됨
- 모듈(Module): 경로 집합, 범위 및 경로 접근 권한으로 구성된 단위
- 경로(Path): 구조체, 함수 또는 모듈과 같은 항목의 이름을 지정하는 방법

패키지는 Cargo에 의해 생성되어 유지되는 단위입니다. 크레이트는 실행파일을 포함한 라이브러리를 의미하며, 크레이트는 모듈 트리로 구성됩니다. 모듈은 관련된 것들로 묶여 있는 범위 및 경로라고 할 수 있습니다. 경로는 구조체 및 함수 또는 모듈과 같은 항목의 전체 이름을 의미합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1628669898253/P7MKdseoT.png)

## 패키지 및 크레이트
패키지는 Cargo에 의해서 생성을 할 수 있으며 실행 패키지 또는 라이브러리 패키지일 수 있습니다. 패키지는 한개 이상의 크레이트를 가질 수 있습니다. 크레이트에 대한 설정은 `Cargo.toml`을 통해 관리할 수 있습니다.

cargo를 통해 생성되는 패키지 예시를 살펴봅시다.

```shell
$ cargo new my-project
        Created binary  (application)  `my-project`  package
$ ls my-project
Cargo.toml
$ ls my-project/src
main.rs
```
기본적으로 `cargo new`로 프로젝트를 생성하게 되면 실행 패키지를 만들 수 있습니다. 이때, 기본 설정은 `Cargo.toml`파일에 저장이 되고 "Hello World!"를 출력하는 `main.rs`파일이 생성이 됩니다.

`Cargo.toml`과 `main.rs`를 살펴봅시다.

Cargo.toml
```toml
[package]
name = "my-project"
version = "0.1.0"
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```

`name`, `version`, `edition` 속성의 패키지 기본 설정 값을 볼 수 있습니다. 자동 생성되는 패키지에는 외부 패키지 참조가 없지만, 사용이 필요할 때 `[dependencies]` 영역에 추가하면 Cargo에 의해 크레이트가 자동으로 포함되어 사용할 수 있게 됩니다.

src/main.rs
```rust
fn main() {
    println!("Hello, world!");
}
```

기본 생성되는 파일입니다. 실행파일을 실행했을 때 최초 실행되는 `main()` 함수를 볼 수 있고, `println!()` 매크로를 통해 "Hello, world!"를 출력하는 러스트 파일 입니다.

## 모듈의 범위 및 접근권한 정의
모듈은 연관된 기능의 묶음입니다. 모듈은 `mod` 키워드를 통해 정의할 수 있습니다.

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```

위의 모듈 정의에 의해 `add_to_waitlist()`함수의 경로는 절대 경로로 `crate::front_of_house::hosting::add_to_waitlist()`가 됩니다. 상대 경로로는 호출하는 위치의 모듈 절대 경로와 호출 대상 함수의 모듈 절대 경로 중 같은 경로는 생략이 되며, 예를들어 `seat_at_table()`에서 `add_to_waitlist()` 함수를 호출한다고 할 때 모듈이 모두 동일하므로 바로 `add_to_waitlist()`로 해당 함수를 호출할 수 있습니다. `serve_order()` 함수에서 `add_to_waitlist()`함수를 호출해야 한다면, `front_of_house::add_to_waitlist()`로 해당 함수를 호출해야 합니다.

위 코드의 모듈 트리는 다음과 같습니다. 
```
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```

## 모듈 트리에서 항목을 참조하는 경로
`절대 경로`와 `상대 경로`로 항목을 참조할 수 있습니다.
- 절대 경로: 크레이트 루트 부터 시작하는 전체 경로를 이용해 항목을 찾음
- 상대 경로: 현재 모듈 또는 `self`, `super` 키워드로 시작하는 상대 경로를 통해 항목을 찾음

다음의 코드로 절대 경로 및 상대 경로의 사용법을 이해할 수 있습니다.

``` rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // 절대 경로
    crate::front_of_house::hosting::add_to_waitlist();

    // 상대 경로
    front_of_house::hosting::add_to_waitlist();
}
```

### pub 접두사로 경로 노출
위의 코드는 컴파일 했을 때 오류가 발생합니다. 이유는, 모듈 외부에서 모듈 내부를 탐색하기 위해서는 `pub` 접두사를 통해 모듈에 접근 가능하도록 해야 하기 때문인데요,

다음의 코드를 보시죠

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // 절대경로
    crate::front_of_house::hosting::add_to_waitlist();

    // 상대경로
    front_of_house::hosting::add_to_waitlist();
}
```

외부로 노출하고자 하는 함수나 구조체 정의에 `pub`를 주고, 해당 모듈에 `pub` 접두사를 줌으로써 컴파일 오류가 발생하지 않게 되었습니다.

### `super`로 상대경로 시작

```rust
fn serve_order() {}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::serve_order();
    }

    fn cook_order() {}
}
```

### 구조체와 열거형을 공개하기
구조체의 필드 또는 함수를 다른 모듈에서 접근해야 한다면 공개해야 할 각각의 필드 또는 함수에 `pub` 접두사를 붙여야 합니다., 열거형의 항목은 열거형이 `pub` 접두사가 붙었을 경우 항목은 `pub`를 붙일 필요는 없습니다.

```rust
mod back_of_house {
    pub struct Breakfast {
        pub toast: String,
        seasonal_fruit: String,
    }

    impl Breakfast {
        pub fn summer(toast: &str) -> Breakfast {
            Breakfast {
                toast: String::from(toast),
                seasonal_fruit: String::from("peaches"),
            }
        }
    }
}

pub fn eat_at_restaurant() {
    // Order a breakfast in the summer with Rye toast
    let mut meal = back_of_house::Breakfast::summer("Rye");
    // Change our mind about what bread we'd like
    meal.toast = String::from("Wheat");
    println!("I'd like {} toast please", meal.toast);

    // The next line won't compile if we uncomment it; we're not allowed
    // to see or modify the seasonal fruit that comes with the meal
    // meal.seasonal_fruit = String::from("blueberries");
}
```

```rust
mod back_of_house {
    pub enum Appetizer {
        Soup,
        Salad,
    }
}

pub fn eat_at_restaurant() {
    let order1 = back_of_house::Appetizer::Soup;
    let order2 = back_of_house::Appetizer::Salad;
}
```

## use 키워드를 사용하여 범위에 경로 가져오기
매던 절대 경로나 상대 경로를 통해 항목에 접근하는 것은 비효율적입니다. `use` 키워드를 이용해 범위에 항목의 경로를 가져올 수 있습니다.

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

`use crate::front_of_house::hosting`으로 `hosting` 이전까지의 경로를 생략할 수 있습니다. 
다음처럼 `self`로 경로를 가져올 수 도 있습니다.

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use self::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

### 관용적 use 경로 만들기

다음의 코드를 봅시다.

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting::add_to_waitlist;

pub fn eat_at_restaurant() {
    add_to_waitlist();
    add_to_waitlist();
    add_to_waitlist();
}
```

`use crate::front_of_house::hosting::add_to_waitlist`를 통해 `hosting`까지의 경로를 생략할 수 도 있습니다. 하지만 해당 함수가 어디에 속해있는지를 알 수 없으므로 지양해야 합니다.

그러나 다음의 코드처럼

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert(1, 2);
}
```

구조체, 열거형 및 기타 항목을 가져올 때는 어떤 항목을 가져왔는지 `use` 키워드를 통해 알 수 있고 사용할 때 `HashMap::new()`등으로 사용할 수 있기 때문에 올바른 관용적 표현이라 할 수 있습니다.

그러나 다음처럼,

```rust
use std::fmt;
use std::io;

fn function1() -> fmt::Result {
    // --snip--
    Ok(())
}

fn function2() -> io::Result<()> {
    // --snip--
    Ok(())
}
```

각기 다른 모듈의 동일한 이름의 구조체일 경우 문제가 될 수 있습니다. 이런 경우 상위 모듈까지만 `use` 키워드로 표현해야 합니다.

### as 키워드로 새 이름 제공

위의 코드에서 `as` 키워드를 통해 다른 이름으로 치환하여 같은 문제를 해결할 수 도 있습니다.

```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
    Ok(())
}

fn function2() -> IoResult<()> {
    // --snip--
    Ok(())
}
```

### pub use를 사용하여 이름 다시 내보내기

`use` 키워드를 통해 외부 경로를 외부로 가져왔어도 가져온 항목이 다른 범위에서는 비공개가 됩니다. 가져온 항목을 사용한 함수가 외부로 공개되려면 `pub use`를 사용해야 합니다.

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

### 외부 패키지 사용
`Cargo.toml`의 `[dependencies]`에 외부 패키지를 추가하는 것으로 외부 패키지를 손쉽게 참조할 수 있습니다. 이는 Cargo에서 패키지 관리를 해주기 때문인데요, 해당 외부 패키지의 의존성까지 확인해서 정상적으로 참조할 수 있도록 해줍니다.

예를 들어,

```toml
rand = "0.8.3"
```
이라고 추가하게 되면 `rand` 패키지와 모든 종속성을 `crates.io`에서 내려받아 프로젝트에서 사용할 수 있게 합니다.

그다음 `use` 키워드를 통해 해당 패키지를 범위로 불러올 수 있습니다.

```rust
use rand::Rng;

fn main() {
    let secret_number = rand::thread_rng().gen_range(1..101);
}
```

표준 라이브러리(std)의 경우 Rust와 함께 제공하기 때문에 `Cargo.toml`에 포함할 필요는 없고 바로 `use` 키워드를 통해 범위로 가져올 수 있습니다.

```rust
use std::collections::HashMap;
```

### 중첩 경로를 사용하여 큰 use 목록 정리

여러개의 `use` 사용을 한줄로 사용할 수 도 있습니다.

```rust
use std::cmp::Ordering;
use std::io;
```

이것을

```rust
use std::{cmp::Ordering, io};
```

이렇게 줄일 수 있습니다.

다음의 형태의 경우,

```rust
use std::io;
use std::io::Write;
```

이것을

```rust
use std::io::{self, Write};
```

이렇게 줄일 수 있습니다.

### 글로브 오퍼레이터
해당 경로의 모든 항목을 범위로 가져오려면 스타(*)를 이용합니다.

```rust
use std::collections::*;
```

## 모듈을 다른 파일로 분리하기
지금까지 한 파일에서 예제 코드를 테스트 해보셨을 텐데요, 실제로는 파일 단위로 나뉘게 됩니다. 

src/lib.rs
```rust
mod front_of_house;

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

src/front_of_house.rs
```rust
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```

`mod front_of_house;` 와 같이 사용할 경우 다른 파일의 동일한 모듈의 항목 또한 가져옴을 의미합니다.

## 요약
Rust를 이용해 프로그래밍을 하게 되면 패키지 안에 여러개의 크레이트로 분할하고, 각각의 크레이트는 또 여러개의 모듈과 하위 모듈로 분할하여 관리할 수 있습니다.
