---
title: "Rust #2: 코드 따라하기(1/2) - 2장 추측 게임 프로그래밍"
datePublished: Wed Jul 14 2021 13:49:58 GMT+0000 (Coordinated Universal Time)
cuid: ckr3jiwty01q6v8s12eh5bs3p
slug: rust-2-12-2
tags: rust

---

## 개요
러스트 홈페이지에서 제공하는 [The Rust Programming Language 온라인 북](https://doc.rust-lang.org/nightly/book/title-page.html)에 나와 있는 코드를 따라 해보면서 Rust의 특징을 파악하고 손에 익히도록 합니다.

## [2장 추측 게임 프로그래밍](https://doc.rust-lang.org/nightly/book/ch02-00-guessing-game-tutorial.html)
여러 프로그래밍 언어에서 실습을 위해 예제로 곧잘 활용하는 숫자 맞추기 게임입니다. 온라인 북은 이를 통해 Cargo 설정파일 및 외부 Crate를 어떻게 포함시키는지 숫자 맞추기 게임을 구현하기 위해 간단한 Rust 코드를 학습 시킵니다.

`cargo`를 이용해 기본 패키지를 생성합니다.
```shell
$ cargo new guessing_game
cd guessing_game
```

기본 `cargo.toml` 구조입니다. 지금은 자동생성된 toml 설정파일의 기본 구조만 살펴봅니다.
```shell
[package]
name = "guessing_game"
version = "0.1.0"
authors = ["Your Name <you@example.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```

기본 생성된 `main.rs` 소스코드 입니다.
```rust
fn main() {
    println!("Hello, world!");
}
```

심지어 이 코드는 `cargo run`을 통해 바로 실행도 잘 됩니다.

```shell
$ cargo run
   Compiling guessing_game v0.1.0 (W:\rust-learning\guessing_game)
    Finished dev [unoptimized + debuginfo] target(s) in 0.59s
     Running `target\debug\guessing_game.exe`
Hello, world!
```

훌륭합니다. 기본 컴파일 속도도 빠르고, 별도의 의존성 없이 곧잘 실행되네요.

`VS Code`로 코딩을 계속 진행합니다.

실행 명령을 통해 Rust를 실행하려면 `launch.json`을 다음과 같이 작성합니다.
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug",
            "program": "${workspaceFolder}/target/debug/guessing_game",
            "args": [],
            "cwd": "${workspaceFolder}"
        }
    ]
}
```
기본 생성되는 설정파일에 `program`의 경로만 설정했습니다. 이제 VS Code로 코딩을 계속할 수 있습니다.

```rust
use std::io;

fn main() {
    println!("Guess the number!");

    println!("Please input your guess.");

    let mut guess = String::new();

    io::stdin()
        .read_line(&mut guess)
        .expect("Failed to read line");

    println!("You guessed: {}", guess);
}
```

첫번째 그럴싸한 Rust 코드입니다. C# 프로그래머 입장에서는 대략적인 의미를 파악할 수 있는 형식과 무엇인지 알 수 없는 부분이 있습니다. 가령, `use std::io`는 std의 io 모듈을 사용하겠다라는 `영역`지정의 의미이지만, 왜 stdio() 앞에 `io::`가 붙어야 하는지는 아직은 모르겠습니다. 함수명이 중복되는것을 최대한 막기 위해 모듈명을 적어줘야 하는게 기본 동작으로 보입니다. 그리고 fn은 function으로 함수를 의미한다고 유추할 수 있고 main() 함수가 최초 진입점임을 알 수 있습니다. `println!()`형태의 함수는 느낌표가 들어가 특이한데, 매크로라는 의미라고 합니다. 친절하게 `F12`를 누르면 해당 기능의 소스코드를 살펴볼 수 있습니다. 아마도 가변 함수 인자를 컴파일 타임에 처리하기 위함으로 보입니다.
`guess`를 `String:new()`로 할당하는데요, 특이한 점은 `mut` 키워드 입니다. 기본적으로 Rust는 `변경가능한` 변수가 아닌 `변경불가능한` 불변(immutable) 변수를 생성하는게 기본 동작으로 보입니다. `io:stdin().read_line()` 함수를 통해 입력한 값을 문자열로 받기 위함인데요, 반환값이 아닌 참조 인자로 받는 것은 특이한 점입니다. 아마 관련해서 Rust 언어의 특징이 드러날 것으로 보입니다.
`read_line()` 함수에서 오류가 발생할 경우 `expect()`의 인자를 오류처리시 사용하는것으로 보입니다.

최종적으로 `println!()`함수를 통해 입력된 값을 출력하네요. `{}`에 값이 들어가는 것으로 보입니다.

다음으로 난수값을 이용하기 위해 rand Crate를 `Cargo.toml`에 추가합니다.

```toml
[dependencies]
rand = "0.8.3"
```

이게 끝입니다. 다시 `cargo run`을 하면 알아서 해당 Crate 및 의존 Crate들을 설치해줍니다.

```rust
use rand::Rng;
use std::io;

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1..101);

    println!("The secret number is: {}", secret_number);

    println!("Please input your guess.");

    let mut guess = String::new();

    io::stdin()
        .read_line(&mut guess)
        .expect("Failed to read line");

    println!("You guessed: {}", guess);
}
```

`use rand::Rng`를 통해 Rand Crate를 사용함을 알리고, `rand::thread_rng().gen_range()`를 통해 난수값을 취합니다. 해당 함수의 정확한 의미는 아직 모릅니다. `1..101` 형태로 범위 값을 사용할 수 있는데, python 등 동적 언어에서 이전부터 지원했었고, C#도 최근 버젼에서는 지원합니다. 

```rust
use rand::Rng;
use std::cmp::Ordering;
use std::io;

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1..101);

    println!("Please input your guess.");

    let mut guess = String::new();

    io::stdin()
        .read_line(&mut guess)
        .expect("Failed to read line");

    let guess: u32 = guess.trim().parse().expect("Please type a number!");

    println!("You guessed: {}", guess);

    match guess.cmp(&secret_number) {
        Ordering::Less => println!("Too small!"),
        Ordering::Greater => println!("Too big!"),
        Ordering::Equal => println!("You win!"),
    }
}
```

`String.trim()` 및 `parse()`를 통해 문자열을 숫자로 변환합니다. 어떻게 `parse()`함수가 반환값을 추측할 수 있는지는 모르곘습니다. Rust의 특징으로 보이네요. => 반환값을 추측하는게 아니라 `FromStr`형을 반환하는데 이게 숫자형으로 보입니다.

`std:cmp::Ordering` 모듈을 이용해 `guess.cmp()`로 작거나 크거나 같음을 비교할 수 있습니다. `match` 키워드를 통해 상당히 세련되게 표현할 수가 있군요.

다음은 최종적인 코드 입니다.

```rust
use rand::Rng;
use std::cmp::Ordering;
use std::io;

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1..101);

    loop {
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        println!("You guessed: {}", guess);

        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}
```

String의 parse()의 반환형인 Result는 Ok이거나 Err일 수 있는데, 함수형 언어의 영향을 받은 것으로 보입니다. Ok일 경우 반환된 값을 그대로 사용할 수 있고 Err일 경우 에러 관련 처리를 할 수 있습니다.

이렇게 2장을 둘러보면서 Rust언어의 특징을 대략적으로 조금이나마 파악할 수 있었습니다.  Rust 표준 라이브러리에서는 반환값으로 Result 형을 대부분 사용할것 같고요, 이하 기본적인 구문은 C# 프로그래머 입장에서는 예측할 만한 범위에 아직은 있는 것 같습니다.

실행화면
```shell
$ cargo run
Guess the number!
Please input your guess.
5
Guess the number!
Please input your guess.
30
You guessed: 30
Too small!
Please input your guess.
40
You guessed: 40
Too small!
Please input your guess.
50
You guessed: 50
Too small!
Please input your guess.
60
You guessed: 60
Too small!
Please input your guess.
70
You guessed: 70
Too small!
Please input your guess.
80
You guessed: 80
Too small!
Please input your guess.
90
You guessed: 90
Too big!
Please input your guess.
85
You guessed: 85
Too big!
Please input your guess.
80
You guessed: 80
Too small!
Please input your guess.
83
You guessed: 83
Too small!
Please input your guess.
84
You guessed: 84
You win!
```