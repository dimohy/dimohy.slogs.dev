## Rust #9: 9장 오류처리

## 개요
Rust는 다른 프로그래밍 언어와 다르게 `복구 가능한 오류`와 `복구 불가능한 오류`를 구분해서 처리합니다. 이시간을 통해 이 두가지 유형에 대해 알아보는 시간을 가져 봅시다.

## 복구할 수 없는 오류 panic! 매크로
복구할 수 없는 오류란 프로그램의 오동작을 최소화 하기 위해 즉시 프로그램을 종료해야 할 때 사용됩니다. 이때 panic! 매크로를 사용하게 됩니다.

> ### 패닉 시 Rust의 기본 동작
> Rust는 패닉이 발생하면 스택을 백업하고 만나는 함수마다 데이터를 정리하는 행위를 합니다. 이 작업은 상당한 작업이 되기 때문에 컴파일한 실행파일을 작게 만드려면 `Cargo.toml`에 다음의 설정을 할 수 있습니다.
> ```toml
[profile.release]
panic = 'abort'
```

panic! 매크로를 간단히 살펴보겠습니다.

```rust
fn main() {
    panic!("crash and burn");
}
```

위의 코드를 컴파일 하여 실행하면 익숙한 런타임 오류 화면을 볼 수 있습니다.

```shell
$ cargo run
   Compiling panic v0.1.0 (file:///projects/panic)
    Finished dev [unoptimized + debuginfo] target(s) in 0.25s
     Running `target/debug/panic`
thread 'main' panicked at 'crash and burn', src/main.rs:2:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

여기서 panic! 매크로 인자의 메시지가 출력됨을 볼 수 있습니다.

### panic! 역추적 사용

다음의 코드를 통해,

```rust
fn main() {
    let v = vec![1, 2, 3];

    v[99];
}
```

배열의 배열 인덱싱이 잘못되었을 때 출력되는 런타임 오류 메시지가 panic! 매크로로 발생한 메시지와 유사함을 알 수 있습니다.

```shell
$ cargo run
   Compiling panic v0.1.0 (file:///projects/panic)
    Finished dev [unoptimized + debuginfo] target(s) in 0.27s
     Running `target/debug/panic`
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', src/main.rs:4:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

이때 추석 기능을 사용하려면 환경변수 `RUST_BACKTRACE`를 0이 아닌 다른 값으로 설정 하면,

```shell
$ RUST_BACKTRACE=1 cargo run
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', src/main.rs:4:5
stack backtrace:
   0: rust_begin_unwind
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/std/src/panicking.rs:483
   1: core::panicking::panic_fmt
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/panicking.rs:85
   2: core::panicking::panic_bounds_check
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/panicking.rs:62
   3: <usize as core::slice::index::SliceIndex<[T]>>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/slice/index.rs:255
   4: core::slice::index::<impl core::ops::index::Index<I> for [T]>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/slice/index.rs:15
   5: <alloc::vec::Vec<T> as core::ops::index::Index<I>>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/alloc/src/vec.rs:1982
   6: panic::main
             at ./src/main.rs:4
   7: core::ops::function::FnOnce::call_once
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/ops/function.rs:227
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

추적 정보를 확인할 수 있습니다.

## 복구 가능한 오류 Result

가령, 파일을 열려고 했을 때 파일이 없어 실패한 경우 프로그램을 종료하는 대신 새로운 파일을 생성하는 것이 로직에 부합할 수 있습니다. 이 때 사용하는 것이 Result<T, E> 열거형인데요, Option<T> 열거형과 함께 Rust 표준 라이브러리에서 많이 사용하는 기능입니다.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Result 열거형은 다음처럼 사용할 수 있습니다.

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => panic!("Problem opening the file: {:?}", error),
};
```

앞전 시간에 배웠던 `match` 문을 통해 정상 동작일 때 file 핸들을 반환하고, 오류가 발생했을 때 패닉으로 처리하는 코드입니다.

### 다른 오류에 대한 일치

위의 코드는 오류가 발생했을 때 패닉으로 프로그램으로 종료하게 되는데요, 우리가 원하는 것은 오류의 다양한 원인에 대한 처리라고 한다면 다음의 코드처럼 작성할 수 있습니다.

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the fie: {:?}", e),
            },
            other_error => {
                panic!("Problem opening the file: {:?}", other_error)
            }
        },
    };
}
```

위의 코드는 파일을 열려고 할 때 파일이 없으면 파일을 생성하고, 파일을 생성해서 파일을 핸들을 반환하거나, 파일을 생성할 수 없으면 패닉을 발생하고, 그 이외의 오류 역시 패닉을 발생하는 코드입니다.

이 코드는 `match` 키워드를 이용한 정상 코드이지만, 몇 개의 처리로 인해 조금 복잡해 보입니다. 다음의 코드처럼 개선할 수 있습니다.

````rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt").unwrap_or_else(|error| {
        if error.kind() == ErrorKind::NotFound {
            File::create("hello.txt").unwrap_or_else(|error| {
                panic!("Problem creating the file: {:?}", error);
            })
        } else {
            panic!("Problem opening the file: {:?}", error);
        }
    });
}
````

위의 코드는 `unwrap_or_else()` 메소드를 이용해 정상 반환의 경의 핸들을 반환하고 오류일 경우 클로저로 처리를 넘기는 것으로 코드를 좀 더 단순화 하였습니다. 13장에서 클로저에 대해 배울 예정입니다.

## Panic on Error 바로가기: unwrap과 expect

`match`를 사용하면 오류 처리에 대해 잘 처리할 수 있지만 장황하고 의도를 잘 전달하지 못할 수 도 있습니다. 다음의 코드를 보시죠.

```rust
use std::fs::File;

fn main() {
        let f = File::open("hello.txt").unwrap();
}
```

`unwrap()` 메소드로 값만 취하고 에러는 무시하고 있습니다. 만약 오류가 발생한다면 오류가 발생했을 때 `unwrap()` 메소드에서 대신 `panic!`을 호출해 줍니다.

```shell
hread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Error {
repr: Os { code: 2, message: "No such file or directory" } }',
src/libcore/result.rs:906:4
```

`unwrap()`과 유사하지만 사용자가 메시지를 출력할 수 있는 `expect()` 메소드가 있습니다.

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").expect("Failed to open hello.txt");
}
```

```shell
thread 'main' panicked at 'Failed to open hello.txt: Error { repr: Os { code:
2, message: "No such file or directory" } }', src/libcore/result.rs:906:4
```

### 오류 전파

만약 자신이 처리할 수 없는 오류라면 그것을 호출한 코드로 돌려줄 수 있습니다.

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");

    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
    };

    let mut s = String::new();

    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}
```

위의 코드는 `match`를 이용해 발생 오류를 그대로 반환하는 값인  `Result<T, E>`의 E로 반환합니다. 여기서 살펴볼 필요가 있는 코드는 `Err(e) => return Err(e),`인데요, 오류가 발생했을 경우 메소드 끝까지 도달하지 않고 여기서 오류를 반환하도록 합니다.

### 오류 전파의 지름길: ?연산자 

?연산자를 사용하면 오류를 호출한 메소드로 오류를 전달합니다. ?연산자를 사용하여 아래처럼 위의 코드를 좀 더 간단하게 작성할 수 있습니다.

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

?연산자는 당연히 이어서 쓸 수 도 있습니다.

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();

    File::open("hello.txt")?.read_to_string(&mut s)?;

    Ok(s)
}
```

최종적으로 이것을 다음처럼 간단하게 할 수 있습니다.

```rust
use std::fs;
use std::io;

fn read_username_from_file() -> Result<String, io::Error> {
    fs::read_to_string("hello.txt")
}
```

### 오류 전파를 위한 바로가기: ? 연산자

다음의 코드는 보시죠.

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt")?;
}
```

? 연산자에 의해서 결정될 수 있는 유형은 `Result<T, E>` 또는 `Option<T>` 또는 여기서 다루지 않은 `std::ops::Try` 일 수 있습니다. 컴파일러는 이것을 결정할 수 없기 때문에 컴파일 오류가 발생합니다.

다음의 코드처럼 반환형을 캐스팅 할 수 있습니다.

```rust
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box<dyn Error>> {
    let f = File::open("hello.txt")?;

    Ok(())
}
```

여기서 사용하는  `Box<dyn Error>` 유형은 17장에서 다룰 예정입니다.

## panic! 하거나 하지 않거나

그렇다면 언제 panic!으로 프로그램을 종료해야 하고 하지 않아야 하는 걸까요? 프로그램의 로직에 따라 panic! 대상이거나 아닐 수 있습니다. 메소드를 설계할 때 이것을 알 수 없는 경우가 대부분이므로 반환형으로 `Result<T, E>`를 사용하는게 좋은 기본 선택이 됩니다.

### 예쩨, 프로토타입 코드 및 테스트

프로토타입으로 전개하는 코드는 `unwrap()`또는 `expect()`메소드를 이용해 오류 처리를 특별히 하지 않고 코드를 전개하는게 효율적입니다. 또한 테스트 코드는 메서드 호출이 실패했을 때 테스트 중인 기능이 아니더라도 테스트가 실패하는게 맞습니다. 이때에도 `unwra[()` 또는 `expect()` 메소드는 유용합니다.

다음의 코드처럼 하드코딩된 코드를 테스트할 때,

```rust
    use std::net::IpAddr;

    let home: IpAddr = "127.0.0.1".parse().unwrap();
```

"127.0.0.1"에 대한 파싱으로 오류가 발생하지 않을것이란 기대를 할 수 있습니다. 이럴 때 `unwarp()`를 사용할 수 있습니다.

### 오류 처리 지침
다음의 오류 지침을 따라 봅시다.
- 나쁜 상태는 가끔 발생할 것으로 예상되는 것이 아닙니다.
- 이 시점 이후의 코드는 이 나쁜 상태에 있지 않아야 합니다.
- 사용하는 유형으로 이 정보를 인코딩하는 좋은 방법은 없습니다.

이럴때 panic!을 사용할 수 있습니다.

### 검증을 위한 사용자 정의 유형 생성

다음의 코드를 예로 보시죠.

```rust
loop {
        // --snip--

        let guess: i32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        if guess < 1 || guess > 100 {
            println!("The secret number will be between 1 and 100.");
            continue;
        }

        match guess.cmp(&secret_number) {
            // --snip--
    }
```

이것은 입력된 guess 값의 범위 검사를 로직에서 수행하기 때문에 이상적이지 않습니다. 이것을 다음처럼 사용자 유형에서 처리할 수 있습니다.

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

    pub fn value(&self) -> i32 {
        self.value
    }
}
```

## 정리
Rust의 오류 처리 기능은 보다 강력한 코드를 작성할 수 있도록 설계되었습니다.  Rust는 panic! 매크로 및 `Result<T, E>`를 이용해서 즉각적으로 중지해야 하는 오류 유형과 복구할 수 있는 오류 유형을 처리하도록 합니다.
