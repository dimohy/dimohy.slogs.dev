---
title: "Rust #16: 16장 두려움 없는 동시성"
datePublished: Wed Oct 13 2021 01:36:40 GMT+0000 (Coordinated Universal Time)
cuid: ckuoueenk13akebs1bi8u8rt4
slug: rust-16-16
tags: rust

---

## 개요

Rust는 동시 프로그래밍을 안전하고 효율적으로 수행하는 것은 Rust의 또 다른 주요 목표라고 합니다. 동시 프로그램은 프로그램의 다른 부분이 독립적으로 실행되는 것을 의미하고 병렬 프로그램은 다른 부분이 동시에 실행되는 것을 의미합니다. 이는 대중적으로 사용하는 컴퓨터조차 다중 코어를 지원하고 그 활용이 활발해 짐에 따라 중요합니다.

역사적으로 동시성 환경에서 프로그래밍 하는 것은 어렵고 오류가 쉽게 발생할 수 있습니다.

Rust팀은 처음에는 메모리 안전을 보장하는 것과 동시성 문제가 별개의 과제라고 생각했습니다. 그러나 시간이 지남에 따라 소유권 및 유형 시스템을 통해 런타임 오류가 아닌 컴파일 타임에 문제를 해결할 수 있음이 밝혀졌습니다. Rust에서는 다른 언어가 런타임에 겪는 동시성에서 발생할 수 있는 심각한 오류를 컴파일 때 오류로 잡을 수 있습니다.

이 장은 다음 주제를 다룹니다.

- 여러 코드를 동시에 실행하는 스레드를 만드는 방버
- 채널을 통해 스레드 간 메시지를 보내는 메시지 전달 동시성
- 여러 스레드가 동일 데이터를 접근할 수 있는 공유 상태 동시성
- Rust의 동시성 보장을 사용자 정의 유형과 표준 라이브러리에서 제공하는 유형으로 확장하는 Sync 및 Send 특성

## 스레드를 사용하요 코드를 동시에 실행

동시에 코드를 실행할 때 발생하는 문제는 다음과 같습니다.

- 스레드가 일관성 없는 순서로 데이터 또는 리소스에 접근하는 "레이스 컨디션(Race conditions)"
- "데드락(Deadlocks)" 즉 교착 상태로, 두 스레드가 서로를 기다리기 위해 멈춰 있는 상태
- 병렬 처리로 인해 특정 상황에서만 발생하여 재현 및 수정하기 어려운 버그

전통적으로 많은 언어들은 이 3가지의 문제점을 세심하게 고려하여 동시성(동시/병렬) 프로그래밍을 해야만 했었습니다.

Rust는 이를 어떻게 접근하는지 같이 보시죠.

### spawn을 사용하여 새 스레드 만들기

Rust는 `thread::spawn()`을 이용해 스레드를 만들 수 있습니다.

```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

예제는 메인스레드와 `spawn()`으로 만들어진 스레드로 코드를 실행하고 있습니다. 그런데 메인스레드의 코드가 종료되자 `spawn()`으로 만든 스레드가 중지하는군요!

```shell
hi number 1 from the main thread!
hi number 1 from the spawned thread!
hi number 2 from the main thread!
hi number 2 from the spawned thread!
hi number 3 from the main thread!
hi number 3 from the spawned thread!
hi number 4 from the main thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
```

메인 스레드가 종료되면 다른 스레드도 같이 중지되는 것이 기본 동작입니다. 

### join 핸들을 사용하여 모든 스레드가 완료될 때까지 기다리기

다음의 코드를 보시죠.

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap();
}
```

메인 스레드에서 `spawn()`으로 만든 스레드를 핸들을 이용해 `join()`으로 대기하고 있습니다. 이제 동작은 다음과 같습니다.

```shell
hi number 1 from the main thread!
hi number 2 from the main thread!
hi number 1 from the spawned thread!
hi number 3 from the main thread!
hi number 2 from the spawned thread!
hi number 4 from the main thread!
hi number 3 from the spawned thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
hi number 6 from the spawned thread!
hi number 7 from the spawned thread!
hi number 8 from the spawned thread!
hi number 9 from the spawned thread!
```

만약 생성 스레드의 동작이 끝날 때까지 메인 스레드에서 대기한 후 동작하게 하려면 다음처럼 할 수 있습니다.

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    handle.join().unwrap();

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

```shell
hi number 1 from the spawned thread!
hi number 2 from the spawned thread!
hi number 3 from the spawned thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
hi number 6 from the spawned thread!
hi number 7 from the spawned thread!
hi number 8 from the spawned thread!
hi number 9 from the spawned thread!
hi number 1 from the main thread!
hi number 2 from the main thread!
hi number 3 from the main thread!
hi number 4 from the main thread!
```

### 스레드와 함께 move 클로저 사용하기

클로저를 사용하면 외부 변수에 대한 접근이 가능해 코드를 좀 더 간단하게 작성할 수 있습니다. 하지만 Rust는 스레드가 언제 실행하는지, 언제까지 실행하는지 특정할 수 없으므로 다음의 코드는 컴파일 오류를 발생합니다.

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("Here's a vector: {:?}", v);
    });

    handle.join().unwrap();
}
```

```shell
$ cargo run
   Compiling threads v0.1.0 (file:///projects/threads)
error[E0373]: closure may outlive the current function, but it borrows `v`, which is owned by the current function
 --> src/main.rs:6:32
  |
6 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `v`
7 |         println!("Here's a vector: {:?}", v);
  |                                           - `v` is borrowed here
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:6:18
  |
6 |       let handle = thread::spawn(|| {
  |  __________________^
7 | |         println!("Here's a vector: {:?}", v);
8 | |     });
  | |______^
help: to force the closure to take ownership of `v` (and any other referenced variables), use the `move` keyword
  |
6 |     let handle = thread::spawn(move || {
  |                                ^^^^^^^

For more information about this error, try `rustc --explain E0373`.
error: could not compile `threads` due to previous error
```

다음은 그런 상황의 예시가 됩니다.

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("Here's a vector: {:?}", v);
    });

    drop(v); // oh no!

    handle.join().unwrap();
}
```

이를 해결하려면 스레드로 외부 변수의 소유권을 이전하는 것이 맞는데요, `move` 키워드를 통해 그것을 표현할 수 있습니다.

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("Here's a vector: {:?}", v);
    });

    handle.join().unwrap();
}
```

이제 `spawn()`의 크로저 내부에서 사용하는 외부 변수 v는 move 이후로 스레드가 소유권을 가지게 됩니다.

만약 `move`한 후 메인함수에서 v를 `drop`한다면 어떻게 될까요? 스레드가 소유권을 가져갔으므로 drop할 수 없다는 컴파일 오류가 발생합니다.

```shell
$ cargo run
   Compiling threads v0.1.0 (file:///projects/threads)
error[E0382]: use of moved value: `v`
  --> src/main.rs:10:10
   |
4  |     let v = vec![1, 2, 3];
   |         - move occurs because `v` has type `Vec<i32>`, which does not implement the `Copy` trait
5  | 
6  |     let handle = thread::spawn(move || {
   |                                ------- value moved into closure here
7  |         println!("Here's a vector: {:?}", v);
   |                                           - variable moved due to use in closure
...
10 |     drop(v); // oh no!
   |          ^ value used here after move

For more information about this error, try `rustc --explain E0382`.
error: could not compile `threads` due to previous error
```

## 메시지 전달을 사용하여 스레드간 데이터 전송

안전한 동시성을 보장하기 위해 사용하는 방법이 메시지 전달 방법입니다. Rust에서는 이를 어떻게 수행하는지 살펴보시죠.

```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
}
```

이 코드는 아직 동작하는 코드는 아닙니다. 제네릭 `T`가 주어지지 않았기 때문인데요, Rust의 편리한 점은 tx또는 rx에서 `T`가 결정되었을 때 위의 코드가 정상 코드가 된다는 점입니다.

`mpsc::channel()`는 메시지를 전달할 수 있는 `tx`와 메시지를 수신할 수 있는 `rx`를 반환합니다. 이것을 통해 어떻게 스레드간 메시지를 주고 받는지 다음의 코드로 살펴봅시다.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });
}
```

먼저 `spawn()`으로 생성하는 스레드에게 `tx`를 전달합니다.  `move`를 사용했기 때문에 이제 스레드에서 소유하게 되었습니다.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

오! 두 스레드가 이제 메시지를 주고 받을 수 있게 되었습니다.

```shell
Got: hi
```

### 채널 및 소유권 이전

Rust의 메시지는 동시성 문제를 해결하기 위해 전송된 메시지를 수신한 쪽으로 소유권을 이전합니다. 다음의 코드를 통해 확인해 봅시다.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
        println!("val is {}", val);
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

`val`은 메시지로 전달되었기 때문에 더이상 `spawn()` 스레드에서 사용할 수 없습니다!

```shell
$ cargo run
   Compiling message-passing v0.1.0 (file:///projects/message-passing)
error[E0382]: borrow of moved value: `val`
  --> src/main.rs:10:31
   |
8  |         let val = String::from("hi");
   |             --- move occurs because `val` has type `String`, which does not implement the `Copy` trait
9  |         tx.send(val).unwrap();
   |                 --- value moved here
10 |         println!("val is {}", val);
   |                               ^^^ value borrowed here after move

For more information about this error, try `rustc --explain E0382`.
```

### 여러 값을 보내고 받는 사람이 기다리고 있는 것을 보기

Rust 메시지는 반복자를 지원합니다! 다음의 코드를 통해 확인해보시죠.

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```

```shell
Got: hi
Got: from
Got: the
Got: thread
```

### 송신기를 복제하여 여러 생산자 생성

지금까지는 1:1 메시지 송수신을 확인했습니다. 이제 송신기를 복제해서 여러 생성자를 통해 단일 수신 처리를 살펴보겠습니다.

```rust
// --snip--

    let (tx, rx) = mpsc::channel();

    let tx1 = tx.clone();
    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx1.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    thread::spawn(move || {
        let vals = vec![
            String::from("more"),
            String::from("messages"),
            String::from("for"),
            String::from("you"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {
        println!("Got: {}", received);
    }

    // --snip--
```

```shell
Got: hi
Got: more
Got: from
Got: messages
Got: for
Got: the
Got: thread
Got: you
```

## 공유 상태 동시성

Rust는 여러 스레드가 자원을 공유하기 위한 `공유 상태 동시성`도 지원합니다. 이 부분은 병렬 세계에서 다양한 버그가 발생할 수 있는 어려운 분야인데요, Rust는 이것을 어떻게 해결했는지 코드를 통해 같이 살펴봅시다.

### 뮤텍스를 사용하여 한번에 한 스레드의 데이터 액세스 허용

뮤텍스를 이용하면 오직 하나의 스레드만 자원에 접근할 수 있도록 합니다.

뮤텍스는 다음의 두 가지 이유 때문에 사용하기 어렵다는 평판이 있습니다.

- 데이터를 사용하기 전에 잠금 획득을 시도해야 합니다.
- 뮤텍스가 보호하는 데이터로 작업을 마치면 다른 스레드가 잠금을 획득할 수 있도록 데이터 잠금을 올바르게 해제해야 합니다.

이 두가지가 제대로 되지 않으면 무한정 잠금 해제를 기다리게 되는 버그가 발생하게 됩니다.

#### Mutex<T> API

Rust는 코드 영역을 Mutex로 처리하는게 아니라 공유할 값을 Mutex로 관리합니다. 이를 통해 자연스럽게 잠금 및 해제 처리를 처리 합니다.

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    {
        let mut num = m.lock().unwrap();
        *num = 6;
    }

    println!("m = {:?}", m);
}
```

#### Mutex<T>를 여러 스레드 간에 공유

다음의 코드를 통해 Mutex<T>를 여러 스레드에서 공유하는 방법을 알 수 있습니다.

```rust
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);
    let mut handles = vec![];

    for _ in 0..10 {
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

하지만 이 코드는 다중 소유권의 문제로 컴파일 오류가 발생합니다.

```shell
$ cargo run
   Compiling shared-state v0.1.0 (file:///projects/shared-state)
error[E0382]: use of moved value: `counter`
  --> src/main.rs:9:36
   |
5  |     let counter = Mutex::new(0);
   |         ------- move occurs because `counter` has type `Mutex<i32>`, which does not implement the `Copy` trait
...
9  |         let handle = thread::spawn(move || {
   |                                    ^^^^^^^ value moved into closure here, in previous iteration of loop
10 |             let mut num = counter.lock().unwrap();
   |                           ------- use occurs due to use in closure

For more information about this error, try `rustc --explain E0382`.
error: could not compile `shared-state` due to previous error
```

#### 다중 스레드가 있는 다중 소유권

15장에서 우리는 이미 Rc<T>를 배웠으므로 적용해보도록 합시다.

```rust
use std::rc::Rc;
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Rc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Rc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

컴파일을 하면,

```shell
$ cargo run
   Compiling shared-state v0.1.0 (file:///projects/shared-state)
error[E0277]: `Rc<Mutex<i32>>` cannot be sent between threads safely
   --> src/main.rs:11:22
    |
11  |           let handle = thread::spawn(move || {
    |  ______________________^^^^^^^^^^^^^_-
    | |                      |
    | |                      `Rc<Mutex<i32>>` cannot be sent between threads safely
12  | |             let mut num = counter.lock().unwrap();
13  | |
14  | |             *num += 1;
15  | |         });
    | |_________- within this `[closure@src/main.rs:11:36: 15:10]`
    |
    = help: within `[closure@src/main.rs:11:36: 15:10]`, the trait `Send` is not implemented for `Rc<Mutex<i32>>`
    = note: required because it appears within the type `[closure@src/main.rs:11:36: 15:10]`

For more information about this error, try `rustc --explain E0277`.
error: could not compile `shared-state` due to previous error
```

아니 이런, 오류가 발생하군요. 이유는 `Rc<T>`가 스레드 안정성을 보장하지 않기 때문입니다. 스레드 안정성이란 여러 스레드에서 동시에 사용되었을 때 그 개체가 정상 동작을 하는가를 의미합니다. 이는 성능과 밀접한 관련이 있기 때문에 `Rc<T>`는 성능을 위해 스레드 안정성을 보장하지 않습니다.

그렇다면 어떤 것을 사용해야 할까요?

####  Arc<T>로 원자적 참조 카운팅

`Rc<T>` 대신 `Arc<T>`를 사용하면 됩니다!

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

이제 컴파일 되고 다음의 정상적인 결과를 확인할 수 있습니다.

```shell
Result: 10
```

### RefCell<T>/Rc<T>와 Mutex<T>/Arc<T>와의 유사점
카운터는 변경 불가능하지만 내부의 변경 가능한 참조를 얻을 수 있습니다. Mutex<T>가 Cell계열과 마찬가지로 내부 가변을 제공하는 것을 의미합니다. 15장에서 RefCell<T>를 이용해서 Rc<T> 내부의 내용 변경하는 것 처럼 Mutex<T>를 사용해서 Arc<T> 내부의 내용을 변경할 수 있습니다.

## Sync 및 Send 특성을 통한 확장 가능한 동시성

최신 언어가 동시성을 처리하기 위해 언어 차원에서 동시성 기능을 제공하는데 반해 Rust 언어에는 동시성 기능이 거의 없다는게 흥미롭습니다. Rust는 동시성을 처리하기 위한 기반을 언어에서 제공하는 소유권 등을 통해 달성했고, 동시성 자체의 기능은 표준 라이브러리를 통해 구현한 샘입니다.
하지만 Rust 언어에 포함되어 있는 `std::maker` 트레잇인 `Sync` 및 `Send`가 있습니다.

### Send를 통해 스레드간 소유권 이전 허용

스레드 간 소유권을 이전할 때 이전이 가능하거나 불가능한지를 Rust에서 인식해야 합니다. 그것을 `Send` 트레잇으로 할 수 있습니다. 가령, `Rc<T>`의 경우 성능 최적화의 이유로 단일 스레드에서만 동작하도록 만들어졌습니다. 즉, Rc<T>는 다른 스레드로 전달해서는 안됩니다.
따라서 Rust의 유형 시스템과 트레잇 경계는 실수로 Rc<T>를 스레드 간에 전달되지 못하도록 합니다.

### Sync를 통해 여러 스레드에서 액세스 허용

마찬가지로 여러 스레드에서 엑세스 가능한지를 표현하기 위해 Sync 트레잇을 구현할 수 있습니다.

### Send와 Sync 의 수동 구현은 안전하지 않은 구현입니다.

Send 및 Sync는 마커 속성입니다. 즉, 직접 트레잇을 구현할 필요가 없습니다. 만약, 수동으로 구현하려면 안전하지 않음(Unsafe)으로 작성해야 합니다.

## 정리
Rust에서 동시성 처리를 하기 위한 방법을 알아봤습니다. 이제 스레드를 생성하고, 스레드에 값을 전달하며, 공유 동시성을 구현하는 방법을 코드로 알아봤습니다.


