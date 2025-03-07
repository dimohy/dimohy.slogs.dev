---
title: "Rust #11: 11장 자동화된 테스트 작성하기"
datePublished: Thu Sep 09 2021 00:46:24 GMT+0000 (Coordinated Universal Time)
cuid: cktc7msnz01mwyhs1ghhr1d8o
slug: rust-11-11
tags: rust

---

## 개요
테스트는 작성된 코드가 의도한 대로 동작하는지 확인하는 작업입니다. Rust는 주석 및 매크로, 테스트 시행을 위해 제공되는 기본 동작 및 옵션, 단위 테스트 및 통합 테스트로 구성하는 방법을 제공합니다.

## 테스트 작성 방법

테스트는 코드가 예상대로 동작 하는지 확인하는 Rust 함수라고 할 수 있습니다.  테스트 함수 내용은 일반적으로 다음 세 가지 작업을 수행합니다.

1. 필요한 데이터 또는 상태를 설정
1. 테스트할 코드를 실행
1. 결과가 기대한 것인지 주장(ASSERT)

### 테스트 함수의 해부

Rust에서 테스트 단위는 test 속성으로 주석이 달린 함수라고 할 수 있습니다. `cargo test`를 통해 테스트 되는 단위이며, 테스트 용 실행 바이너리를 빌드하고 각 테스트 함수의 통과 또는 실패 여부를 보고합니다.

`Cargo`를 이용해 새 라이브러리 프로젝트를 생성하면 기본 테스트 기능이 포함된 테스트 모듈이 자동 생성됩니다.

```shell
$ cargo new adder --lib
     Created library `adder` project
$ cd adder
```

파일명: src/lib.rs
```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```
`it_works()'함수는 `#[test]`에 의해 테스트 함수가 됩니다. `assert_eq!`매크로는 두 값이 같은지 주장(확인)합니다. 같으면 테스트 통과이고 같지 않으면 실패가 됩니다.


```shell
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.57s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`it_works`를 `exploration`으로 변경하면 출력 결과 달라집니다.

```shell
$ cargo test
...
running 1 test
test tests::exploration ... ok
...
```

테스트 함수를 추가해 여려 개의 테스트를 수행할 수 있습니다.

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
        assert_eq!(2 + 2, 4);
    }

    #[test]
    fn another() {
        panic!("Make this test fail");
    }
}
```

```shell
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.72s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 2 tests
test tests::another ... FAILED
test tests::exploration ... ok

failures:

---- tests::another stdout ----
thread 'main' panicked at 'Make this test fail', src/lib.rs:10:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::another

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

### assert! 매크로로 결과 확인하기

`assert!` 매크로는 `true`일 때 테스트 성공, `false`일 때 테스트 실패이며 `panic!`을 호출합니다.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        let larger = Rectangle {
            width: 8,
            height: 7,
        };
        let smaller = Rectangle {
            width: 5,
            height: 1,
        };

        assert!(larger.can_hold(&smaller));
    }
}
```
```shell
$ cargo test
   Compiling rectangle v0.1.0 (file:///projects/rectangle)
    Finished test [unoptimized + debuginfo] target(s) in 0.66s
     Running unittests (target/debug/deps/rectangle-6584c4561e48942e)

running 1 test
test tests::larger_can_hold_smaller ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests rectangle

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

다음은 더 작은 사각형이 더 큰 사각형을 담을 수 없다고 단언하는 또 다른 테스트를 추가 해보겠습니다.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        // --snip--
    }

    #[test]
    fn smaller_cannot_hold_larger() {
        let larger = Rectangle {
            width: 8,
            height: 7,
        };
        let smaller = Rectangle {
            width: 5,
            height: 1,
        };

        assert!(!smaller.can_hold(&larger));
    }
}
```

```shell
$ cargo test
   Compiling rectangle v0.1.0 (file:///projects/rectangle)
    Finished test [unoptimized + debuginfo] target(s) in 0.66s
     Running unittests (target/debug/deps/rectangle-6584c4561e48942e)

running 2 tests
test tests::larger_can_hold_smaller ... ok
test tests::smaller_cannot_hold_larger ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests rectangle

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

다음에는 일부러 코드에 버그를 넣어서 이 테스트 결과가 어떻게 달라지는지 봅시다.

```rust
// --snip--
impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width < other.width && self.height > other.height
    }
}
```

```shell
$ cargo test
   Compiling rectangle v0.1.0 (file:///projects/rectangle)
    Finished test [unoptimized + debuginfo] target(s) in 0.66s
     Running unittests (target/debug/deps/rectangle-6584c4561e48942e)

running 2 tests
test tests::larger_can_hold_smaller ... FAILED
test tests::smaller_cannot_hold_larger ... ok

failures:

---- tests::larger_can_hold_smaller stdout ----
thread 'main' panicked at 'assertion failed: larger.can_hold(&smaller)', src/lib.rs:28:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::larger_can_hold_smaller

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

버그가 생기면서 정상적으로 동작했던 테스트 함수들의 결과를 통해 버그를 감지할 수 있게 되었습니다!

### assert_eq! 및 assert_ne! 매크로를 사용하여 같음 테스트

`assert!` 매크로 뿐만 아니라 `assert_eq!`및 `assert_ne!` 매크로를 통해 테스트를 진행 할 수 있습니다.

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_adds_two() {
        assert_eq!(4, add_two(2));
    }
}
```

`4`와 `add_two(2)`의 결과값(4) 가 같은지 주장합니다. 같으면 테스트 성공, 같지 않으면 테스트 실패가 됩니다.

```shell
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.58s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`add_two()`함수를 다음과 같이 수정해 버그를 만들어 봅시다.

```rust
pub fn add_two(a: i32) -> i32 {
    a + 3
}
```

```shell
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.61s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::it_adds_two ... FAILED

failures:

---- tests::it_adds_two stdout ----
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `4`,
 right: `5`', src/lib.rs:11:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::it_adds_two

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

기대했던 결과가 아니므로 테스트는 실패하며 버그를 감지할 수 있습니다!

`assert_ne!`매크로는 두 값이 동일하지 않을 경우 테스트 성공, 같을 경우 테스트 실패가 됩니다.

`같거나 같지않음`은 모든 기본 유형과 대부분의 표준 라이브러리 유형에서 구현되어 있지만 사용자 유형의 경우 `PartialEq`를 구현해야 합니다. 또 실패할 경우 출력하는 기능인 `Debug`를 구현해야 합니다. 이는 파생 가능한 특성이기 때문에  `#[derive(PartialEq, Debug)]` 주석을 추가하는 것으로 가능합니다.

### 사용자 정의 실패 메시지 추가

```rust
pub fn greeting(name: &str) -> String {
    format!("Hello {}!", name)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn greeting_contains_name() {
        let result = greeting("Carol");
        assert!(result.contains("Carol"));
    }
}
```

이 코드의 테스트 결과는 성공합니다. 실패하도록 다음처럼 수정해봅시다.

```rust
pub fn greeting(name: &str) -> String {
    String::from("Hello!")
}
```

```shell
$ cargo test
   Compiling greeter v0.1.0 (file:///projects/greeter)
    Finished test [unoptimized + debuginfo] target(s) in 0.91s
     Running unittests (target/debug/deps/greeter-170b942eb5bf5e3a)

running 1 test
test tests::greeting_contains_name ... FAILED

failures:

---- tests::greeting_contains_name stdout ----
thread 'main' panicked at 'assertion failed: result.contains(\"Carol\")', src/lib.rs:12:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::greeting_contains_name

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

Rust는 매크로의 기능으로 똑똑하게 `result.contains("Carol")`이라는 메시지를 출력합니다. 하지만 사용자 메시지로 표시하고자 할 경우, 다음의 코드처럼 할 수 있습니다.

```rust
    #[test]
    fn greeting_contains_name() {
        let result = greeting("Carol");
        assert!(
            result.contains("Carol"),
            "Greeting did not contain name, value was `{}`",
            result
        );
    }
```

```shell
$ cargo test
   Compiling greeter v0.1.0 (file:///projects/greeter)
    Finished test [unoptimized + debuginfo] target(s) in 0.93s
     Running unittests (target/debug/deps/greeter-170b942eb5bf5e3a)

running 1 test
test tests::greeting_contains_name ... FAILED

failures:

---- tests::greeting_contains_name stdout ----
thread 'main' panicked at 'Greeting did not contain name, value was `Hello!`', src/lib.rs:12:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::greeting_contains_name

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
````

### 패닉 확인 should_panic
코드가 예상한 대로 동작 하는지 확인하는 것도 중요하지만 코드가 예상대로 오류 조건을 처리하는지 확인하는 것도 중요합니다.

```rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

```shell
$ cargo test
   Compiling guessing_game v0.1.0 (file:///projects/guessing_game)
    Finished test [unoptimized + debuginfo] target(s) in 0.58s
     Running unittests (target/debug/deps/guessing_game-57d70c3acb738f4d)

running 1 test
test tests::greater_than_100 - should panic ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests guessing_game

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`should_panic` 속성으로 해당 함수가 `panic!`되었을 때 테스트 성공으로 처리합니다!
이제 버그를 만들어서 이를 감지 하는지 확인해봅시다.

```rust
// --snip--
impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess { value }
    }
}
```

```shell
$ cargo test
   Compiling guessing_game v0.1.0 (file:///projects/guessing_game)
    Finished test [unoptimized + debuginfo] target(s) in 0.62s
     Running unittests (target/debug/deps/guessing_game-57d70c3acb738f4d)

running 1 test
test tests::greater_than_100 - should panic ... FAILED

failures:

---- tests::greater_than_100 stdout ----
note: test did not panic as expected

failures:
    tests::greater_than_100

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

멋지네요. 그런데 `panic!`이 테스트하고자 하는 원인으로 발생하지 않을 수 있습니다. 이런 경우 테스트 실패가 되어야 하는데 다음 코드처럼 `expected`에 `panic!` 문자열을 주어 이것을 구분할 수 있습니다.

```rust
// --snip--
impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 {
            panic!(
                "Guess value must be greater than or equal to 1, got {}.",
                value
            );
        } else if value > 100 {
            panic!(
                "Guess value must be less than or equal to 100, got {}.",
                value
            );
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "Guess value must be less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

이제 버그를 만들어 봅시다.

```rust
        if value < 1 {
            panic!(
                "Guess value must be less than or equal to 100, got {}.",
                value
            );
        } else if value > 100 {
            panic!(
                "Guess value must be greater than or equal to 1, got {}.",
                value
            );
        }
```

```shell
$ cargo test
   Compiling guessing_game v0.1.0 (file:///projects/guessing_game)
    Finished test [unoptimized + debuginfo] target(s) in 0.66s
     Running unittests (target/debug/deps/guessing_game-57d70c3acb738f4d)

running 1 test
test tests::greater_than_100 - should panic ... FAILED

failures:

---- tests::greater_than_100 stdout ----
thread 'main' panicked at 'Guess value must be greater than or equal to 1, got 200.', src/lib.rs:13:13
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
note: panic did not contain expected string
      panic message: `"Guess value must be greater than or equal to 1, got 200."`,
 expected substring: `"Guess value must be less than or equal to 100"`

failures:
    tests::greater_than_100

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

테스트 실패로 버그를 감지할 수 있습니다.

### Result<T, E> 테스트에서 사용

다음의 코드를 봅시다.

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() -> Result<(), String> {
        if 2 + 2 == 4 {
            Ok(())
        } else {
            Err(String::from("two plus two does not equal four"))
        }
    }
}
```

결과가 `Ok`이면 테스트 성공, `Err`이면 테스트 실패이며, 메시지가 출력 됩니다.
또한 `panic!`이 호출되었을 때 테스트가 실패하므로 물음표 연산자를 통해 위의 코드를 좀 더 단순하게 만들 수도 있습니다.
`Result<T, E>`를 반환하는 테스트 함수에서는 `#[should_panic]`을 사용할 수 없습니다.

## 테스트 실행 방법 제어
`cargo run`이 코드를 컴파일 한 후 바이너리를 실행하는 것 처럼 `cargo test`는 테스트 모드에서 코드를 컴파일하고 테스트 바이너리를 실행합니다. 명령 옵션을 통해 기본 동작을 바꿀 수 있습니다.

### 병렬 또는 연속으로 테스트 실행
테스트를 더 빨리 끝내기 위해 테스트 스레드를 지정해서 병렬로 테스트를 진행할 수 있습니다.

```shell
$ cargo test -- --test-threads=1
```

테스트 환경이 병렬로 진행할 수 있는지 확인한 후 테스트 스레드 개수를 늘려 테스트 시간을 단축할 수 있습니다.

### 함수 출력 표시
특별한 인자를 주지 않을 경우 테스트 중에는 테스트가 성공했을 때는 표준 출력을 캡쳐 하지 않습니다.

```rust
fn prints_and_returns_10(a: i32) -> i32 {
    println!("I got the value {}", a);
    10
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn this_test_will_pass() {
        let value = prints_and_returns_10(4);
        assert_eq!(10, value);
    }

    #[test]
    fn this_test_will_fail() {
        let value = prints_and_returns_10(8);
        assert_eq!(5, value);
    }
}
```

```shell
$ cargo test
   Compiling silly-function v0.1.0 (file:///projects/silly-function)
    Finished test [unoptimized + debuginfo] target(s) in 0.58s
     Running unittests (target/debug/deps/silly_function-160869f38cff9166)

running 2 tests
test tests::this_test_will_fail ... FAILED
test tests::this_test_will_pass ... ok

failures:

---- tests::this_test_will_fail stdout ----
I got the value 8
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `5`,
 right: `10`', src/lib.rs:19:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::this_test_will_fail

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

다음의 인자를 주어 테스트가 성공했을 때도 표준 출력으로 캡쳐할 수 있습니다.

```shell
$ cargo test -- --show-output
```

```shell
$ cargo test -- --show-output
   Compiling silly-function v0.1.0 (file:///projects/silly-function)
    Finished test [unoptimized + debuginfo] target(s) in 0.60s
     Running unittests (target/debug/deps/silly_function-160869f38cff9166)

running 2 tests
test tests::this_test_will_fail ... FAILED
test tests::this_test_will_pass ... ok

successes:

---- tests::this_test_will_pass stdout ----
I got the value 4


successes:
    tests::this_test_will_pass

failures:

---- tests::this_test_will_fail stdout ----
I got the value 8
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `5`,
 right: `10`', src/lib.rs:19:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::this_test_will_fail

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

### 이름으로 테스트 하위 집합 실행
다음의 테스트 코드가 있을 때,

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_two_and_two() {
        assert_eq!(4, add_two(2));
    }

    #[test]
    fn add_three_and_two() {
        assert_eq!(5, add_two(3));
    }

    #[test]
    fn one_hundred() {
        assert_eq!(102, add_two(100));
    }
}
```

테스트를 진행하면 모든 테스트가 수행됩니다.

```shell
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.62s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 3 tests
test tests::add_three_and_two ... ok
test tests::add_two_and_two ... ok
test tests::one_hundred ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

#### 단일 테스트 실행
특정 테스트만 수행하고 싶을 경우 인자로 테스트 함수 이름을 넣을 수 있습니다.

```shell
$ cargo test one_hundred
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.69s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::one_hundred ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out; finished in 0.00s
```

#### 여러 테스트를 실행하기 위한 필터링
특정 이름으로 두 개 이상의 테스트 함수를 실행할 수 있습니다.

```shell
$ cargo test add
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.61s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 2 tests
test tests::add_three_and_two ... ok
test tests::add_two_and_two ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.00s
```

### 특별히 요청하지 않는 한 일부 테스트 무시
`ignore` 속성이 부여되었을 경우 해당 테스트 함수는 전체 테스트에서 제외됩니다. 

```rust
#[test]
fn it_works() {
    assert_eq!(2 + 2, 4);
}

#[test]
#[ignore]
fn expensive_test() {
    // code that takes an hour to run
}
```

```shell
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.60s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 2 tests
test expensive_test ... ignored
test it_works ... ok

test result: ok. 1 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.02s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

하지만 `ignore` 옵션을 주었을 경우 해당 테스트 함수가 실행되게 됩니다.

```shell
$ cargo test -- --ignored
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.61s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test expensive_test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 테스트 구조
테스트는 복잡한 분야이며 사람들마다 다른 용어와 구조를 사용합니다. Rust에서는 단위 테스트와 통합 테스트라는 두 가지 주요 범주에서 테스트를 바라봅니다.

두 종류의 테스트를 작성하는 것은 라이브러리가 자체 또는 이용하는 측면에서 기대하는 대로 동작 하는지 확인하는데 중요합니다.

### 단위 테스트
단위 테스트의 목적은 자체적으로 각 기능이 정상적으로 동작하는지 확인하는데 목적이 있습니다. 라이브러리 내부의 기능이 잘 동작 하는 것을 확인합니다.

#### 테스트 모듈 및 #[cfg(test)]
`#[cfg(test)]`는 `cargo build`에는 포함되지 않고 `cargo test`로 테스트 중에만 해당 모듈이 포함되도록 Rust에 지시 합니다.
그러나 통합 테스트는 다른 디렉토리로 이동하기 때문에 `#[cfg(test)]` 주석이 필요하지 않습니다. 그러나 단위 테스트는 일반 코드와 동일한 코드에 있기 때문에 `#[cfg(test)]` 주석을 주어 빌드 시 바이너리에 포함되지 않도록 합니다.

앞 전에 `addr` 라이브러리 프로젝트를 생성했을 떄 다음과 같이 단위 테스트 코드가 생성된 것을 확인할 수 있었습니다.

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

#### 비공개 기능 테스트
비공개 기능을 직접 테스트해야 하는지에 대한 논쟁이 있지만 Rust는 이를 허용합니다.

```rust
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}

fn internal_adder(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn internal() {
        assert_eq!(4, internal_adder(2, 2));
    }
}
```

### 통합 테스트
통합 테스트는 라이브러리가 정상적으로 동작하는지 대상인 라이브러리의 외부에서 테스트를 진행합니다. 마치 라이브러리를 이용하는 것처럼 동일한 방식으로 공개된 API를 호출합니다. 그러므로 단위 테스트와는 다르게 비공개 기능에는 접근할 수 없습니다. 실제로 사용하는 관점에서 테스트를 수행하므로 통합 테스트 역시 단위 테스트 만큼 중요합니다.

#### `tests` 디렉토리
Rust는 `tests` 디렉토리가 통합 테스트를 위한 디렉토리라는 것을 알고 있습니다.

파일 이름: tests/integration_test.rs
```rust
use adder;

#[test]
fn it_adds_two() {
    assert_eq!(4, adder::add_two(2));
}
```

라이브러리와는 별도의 테스트이므로 `#[cfg(test)]` 주석이 빠진다는 것을 알 수 있습니다.

```shell
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 1.31s
     Running unittests (target/debug/deps/adder-1082c4b063a8fbe6)

running 1 test
test tests::internal ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_test.rs (target/debug/deps/integration_test-1082c4b063a8fbe6)

running 1 test
test it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

파일 단위의 테스트를 수행하기 위해 다음과 같이 인자를 주어 테스트를 진행할 수 있습니다.

```shell
$ cargo test --test integration_test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.64s
     Running tests/integration_test.rs (target/debug/deps/integration_test-82e7799c1bc62298)

running 1 test
test it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

#### 통합 테스트의 하위 모듈
통합 테스트에도 일반 코드 모듈처럼 하위 모듈로 구성해서 테스트를 진행할 수 있습니다. 다음은 테스트 함수에서 초기화 등을 이유로 공통으로 호출해야 하는 함수가 있을 경우 `common.rs`등으로 만들고 호출해서 사용할 수 있습니다.

파일 이름: tests/common.rs
```rust
pub fn setup() {
    // setup code specific to your library's tests would go here
}
```

```shell
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.89s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::internal ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/common.rs (target/debug/deps/common-92948b65e88960b4)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_test.rs (target/debug/deps/integration_test-92948b65e88960b4)

running 1 test
test it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

그런데 테스트 대상에 해당 함수가 포함되었습니다. 우리가 원하는 결과는 아닙니다. 이것을 해결하려면 `tests/common.rs` 대신 `tests/common/mod.rs`의 위치에 만듭니다. 다음은 테스트 함수에서 common모듈을 호출하는 예입니다.

```rust
use adder;

mod common;

#[test]
fn it_adds_two() {
    common::setup();
    assert_eq!(4, adder::add_two(2));
}
```

#### 바이너리 크레이트 통합 테스트
Rust는 라이브러리 크레이트만 통합 테스트를 지원합니다.

## 정리
Rust의 테스트 기능 코드가 변경 되었을 때 코드가 정상 동작함을 확인하는 방법을 제공합니다. 단위 테스트는 라이브러리의 여러 부분을 개별적으로 실행하고 비공개 구현 세부 정보를 테스트할 수 있습니다. 통합 테스트는 종합적으로 라이브러리와 함께 외부 코드에서 사용하는 것과 동일한 방식으로 코드를 테스트 합니다.