---
title: "Rust #3: 코드 따라하기(2/2) - 3장 일반적인 프로그래밍 개념"
datePublished: Thu Jul 15 2021 07:44:58 GMT+0000 (Coordinated Universal Time)
cuid: ckr4lxd8b0694s1s1b30gaaf2
slug: rust-3-22-3
tags: rust

---

## 개요
앞에서 [2장 추측 게임 프로그램의 코드](https://doc.rust-lang.org/nightly/book/ch02-00-guessing-game-tutorial.html)를 따라 가면서 Rust 코드에 익숙해졌는데요, 오늘은 [3장 일반적인 프로그래밍 개념](https://doc.rust-lang.org/nightly/book/ch03-00-common-programming-concepts.html)의 코드를 따라가면서 Rust에 대해서 파악해보도록 합시다.

## 시작
이 장에서는 거의 모든 프로그래밍 언어에서 나타나는 개념과 이 개념이 Rust에서 어떻게 작동하는지를 다룹니다. 프로그래밍 언어의 핵심은 많은 공통점이 있으므로, 이 공통점을 이용해 Rust 언어를 효율적으로 파악하는데 목적이 있습니다.

## 변수와 가변성
첫번째 코드를 봅시다.

```rust
fn main() {
    let x = 5;
    println!("The value of x is: {}", x);
    x = 6;
    println!("The value of x is: {}", x);
}
```

이 코드를 실행하기 위해 `cargo`를 이용해 패키지를 생성합니다.

```shell
$ cargo new chapter3
cd chapter3
code .
```

다음 위의 코드를 타이핑 합니다. 번거롭지만, 눈으로 바로 그 의미를 알 수 있지만 그래도 타이핑 합니다. 프로그래밍은 머리만 쓰는게 아니라 손도 써야 하니까요. 익숙해져야 합니다.

아래처럼 `x = 6`에 빨간 줄이 가져 있는데요, x 변수는 불변성이라 그렇습니다. Rust는 기본 변수를 불변(immutable)으로 할당하기 때문인데요, Rust의 주요 특징이라 할 수 있겠습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1626319050096/uCYdtEYPP.png)

좋아요. 일단 오류를 잡아볼까요? 다음의 코드를 타이핑 합시다.

```rust
fn main() {
    let mut x = 5;
    println!("The value of x is: {}", x);
    x = 6;
    println!("The value of x is: {}", x);
}
```

`let`다음에 `mut`키워드를 추가함으로써 불변이 변할 수 있게 되었습니다. 단, 지금은 오류만 잡은 것이지 Rust의 기본 변수 할당이 불변한 이유는 차차 알아가보도록 해보죠.

### 변수와 상수의 차이점

다음의 코드를 보시죠.

```rust
const MAX_POINTS: u32 = 100_000;
```

직접 타이핑 하는게 중요합니다. `MAX_POINTS`라는 상수를 선언 했는데요, 상수와 변수와의 중요한 차이점은 비할당, 할당이라고 할 수 있겠습니다. 상수는 메모리에 할당되는게 아니에요. 상수는 보통 대문자로 표현하고 언더바로 단어와 단어를 구분합니다. Rust의 상수의 특이점은 형이 있다는것인데요, 이것은 컴파일타임 때 형검사에 이용될 것으로 보입니다. 그리고 마지막으로 자릿수를 표시하는 언더바(`_`)인데요, 최근에 C#에도 도입을 했습니다. 눈이 좀 더 편해집니다.

### 쉐도잉

다음의 코드를 보시죠.

```rust
fn main() {
    let x = 5;

    let x = x + 1;

    let x = x * 2;

    println!("The value of x is: {}", x);
}
```

그리고 직접 타이핑 해봅시다. 아... C#에서는 오류가 날 것이 Rust에는 오류가 안나는군요! 이것을 쉐도잉 이라고 합니다. 이거 메모리를 재사용 하는게 아니라요, 동일한 변수명을 재사용한다고 이해하는게 맞고, 코드 블럭에 영향을 받습니다. (컴파일 내부적으로 아마 쉐도잉 관련 최적화를 통해 동일한 메모리를 사용할 수도 있겠습니다만, 그건 컴파일러의 영역인 것 같습니다.)

```rust
let spaces = "   ";
let spaces = spaces.len();
```

이번 코드는 더 신박합니다. 오류 안납니다! 심지어 동일한 변수명으로 이전 변수명을 참조해 값을 취하고요, 심지어 변수형도 다릅니다. 그래도 오류 안나고 잘 동작합니다. 이게 코딩에서 어떤 강점이 있을까요? 일단, 의미적인 구분을 위해 변수를 다양하게 할당하는 습관이 다른 언어에서는 발생할 수 있는데 컴파일러 입장에서는 최적화의 걸림돌입니다. 그런데 이렇게 동일한 변수를 의미에 맞게 쉐도잉 하면, 컴파일러가 되려 메모리 최적화를 할 수 있는 여지가 생기겠죠. 아직은 저의 뇌피셜입니다만, 근거있는 추측입니다.

이제 아래 `let`을 빼봅시다.
```rust
let spaces = "   ";
spaces = spaces.len();
```

불변변수를 수정했고 더군다나 변수형까지 일치하지 않으니 컴파일러에서 오류를 반환합니다.

## 데이터 유형

데이터형은 `: u32` 처럼 변수 오른쪽에 표현할 수 있습니다.

```rust
let guess: u32 = "42".parse().expect("Not a number!");
```

Rust는 정적으로 유형이 지정되는 언어입니다. 즉, 컴파일 시점에서 모든 데이터의 유형이 결정됩니다. Rust는 데이터형을 유추할 수 있는 경우 생략이 가능한데요, 생략이 불가능 한 경우에는 데이터형을 지정해줘야 합니다. 위의 코드에서 `: u32`를 뺴면 컴파일러가 데이터형을 유추할 수 없다는 오류가 발생합니다.

데이터형은 크게 스칼라 유형과 복합 유형이있는데, 스칼라유형은 정수, 부동소수점, 부울형, 문자형이 있고요 복합유형은 튜플과 배열이 있습니다.

스칼라 유형은 정적 언어의 그것과 유사한데 `i8`, `u32`등 간결하게 표기하는게 좀 특이합니다. 마찬가지로 10진수, 16진수, 8진수, 2진수 및 바이트를 다음과 같이 표현할 수 있습니다.
- 10진수 : 98_222
- 16진수 : 0xff
- 8진수 : 0o77
- 2진수 : 0b1111_0000
- 바이트 : b'A'

값 사이의 언더바(`_`)는 값의 가독성을 올리는데 유용한데요, 10진수의 1000자리 표시와 2진수의 4비트 또는 8비트를 나눠서 표현하는데 유용합니다.

여타 언어처럼 정수 오버플로가 발생하는데 Rust는 이를 디버그 모드로 컴파일 했을 떄만 오류로 감지하고 릴리즈 모드에는 오류를 발생하지 않는데, 이런 선택은 성능 관련 선택사항으로 보입니다.

### 스칼라 유형

#### 부동 소수점 유형
```rust
fn main() {
    let x = 2.0; // f64

    let y: f32 = 3.0; // f32
}
```

#### 수치 연산
```
fn main() {
    // addition
    let sum = 5 + 10;

    // subtraction
    let difference = 95.5 - 4.3;

    // multiplication
    let product = 4 * 30;

    // division
    let quotient = 56.7 / 32.2;

    // remainder
    let remainder = 43 % 5;
}
```

#### 부울 유형
```rust
fn main() {
    let t = true;

    let f: bool = false; // with explicit type annotation
}
```

#### 문자 유형
```rust
fn main() {
    let c = 'z';
    let z = 'ℤ';
    let heart_eyed_cat = '😻';
}
```

### 복합유형
#### 튜플
튜플은 관련된 정보의 묶음입니다. 튜플로 묶이게 되면 값을 변경할 수 없고, 일단 선언되면 크기를 늘리거나 줄일수 없습니다.

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
}
```

그리고 다음처럼 튜플로 만들거나 다시 개별 변수로 대입할 수 있습니다.

```rust
fn main() {
    let tup = (500, 6.4, 1);

    let (x, y, z) = tup;

    println!("The value of y is: {}", y);
}
```

`.0`, `.1`, `.2`등으로 튜플 요소에 접근할 수 있습니다.

```rust
fn main() {
    let x: (i32, f64, u8) = (500, 6.4, 1);

    let five_hundred = x.0;

    let six_point_four = x.1;

    let one = x.2;
}
```

#### 배열
Rust의 배열은 튜플과 다르게 동일 유형이여야 합니다. 그리고 선언이 되면 길이는 고정이 됩니다.

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];
}
```

배열은 크기가 고정되어 있고 변하지 않을 경우 스택에 배치되기 때문에 성능상의 이점이 있습니다. 크기가 유동적으로 변해야 할 경우에는 맞지 않는데요, 이때에는 vector를 쓰는게 좋습니다.

```rust
let months = ["January", "February", "March", "April", "May", "June", "July",
              "August", "September", "October", "November", "December"];
```

일년의 달은 정해져 있으므로 이런 경우 유요하지요.

다음처럼 요소의 유형 및 길이를 지정할 수 도 있습니다. 
```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
```

동일한 값으로 배열을 초기화 할 수 도 있습니다.
```rust
let a = [3; 5];
```

다음처럼 배열의 요소에 접근할 수 있습니다.
```rust
fn main() {
    let a = [1, 2, 3, 4, 5];

    let first = a[0];
    let second = a[1];
}
```

배열의 요소 접근 인덱스의 범위가 벗어나면 Rust는 오류를 발생합니다. 잘못된 메모리 접근을 하지 않습니다.

```rust
use std::io;

fn main() {
    let a = [1, 2, 3, 4, 5];

    println!("Please enter an array index.");

    let mut index = String::new();

    io::stdin()
        .read_line(&mut index)
        .expect("Failed to read line");

    let index: usize = index
        .trim()
        .parse()
        .expect("Index entered was not a number");

    let element = a[index];

    println!(
        "The value of the element at index {} is: {}",
        index, element
    );
}
```

```shell
thread 'main' panicked at 'index out of bounds: the len is 5 but the index is 10', src/main.rs:19:19
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

## 함수
함수는 대부분의 언어에서 코드의 동작을 정의합니다. Rust도 마찬가지로 `fn`이라는 Function의 축약된 표현으로 이 함수를 표현한는데요, 기본적인 구조는 다음과 같습니다. 이제는 알것 같아도 저와 함께 그대로 타이핑 하시죠. 우리는 아는게 중요한게 아니라 코드에 익숙해져야 합니다.

```rust
fn main() {
    println!("Hello, world!");

    another_function();
}

fn another_function() {
    println!("Another function.");
}
```

좋습니다. 좋아요. VS Code의 훌륭한 환경 덕분에 컴파일 전에 바로바로 오타도 확인하고 자동 코드 정리도 되고 좋네요. 실행하면 다음처럼 출력됩니다.

```shell
$ cargo run
Hello, world!
Another function.
```

다른 언어에 숙달된 분들은 다들 이런 전개 익숙하시죠? 계속 해봅시다.

### 함수 인자
다음의 코드는 함수 인자를 사용하는 방법을 보여줍니다.

```rust
fn main() {
    another_function(5);
}

fn another_function(x: i32) {
    println!("The value of x is: {}", x);
}
```
특이한점은 앞에서의 변수 선언과 유사하게 변수형이 변수명의 우측이 있습니다. 요즘 언어들의 트랜드 같은데요, 아직까지는 다른 언어와 매우 유사한 모습입니다. 그래도 열심히 따라 쳐봅시다.

```rust
fn main() {
    another_function(5, 6);
}

fn another_function(x: i32, y: i32) {
    println!("The value of x is: {}", x);
    println!("The value of y is: {}", y);
}
```

복수개 함수 인자는 컴마로 구분해서 넘겨줄 수 있습니다.

다음으로 명령문(statement)를 봅시다.

```rust
fn main() {
    let y = 6;
}
```

`let y = 6`은 반환값이 없는 명령문입니다. 그러므로 다음과 같이 사용할 수 없습니다.

```rust
fn main() {
    let x = (let y = 6);
}
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1626333021096/6-vBcxO5o.png)

다른 언어들이 `x = y = 6`처럼 쓸 수 있는데 반에 Rust는 그럴수 없습니다. 대신 다음의 특이한 형태를 제공하는데요,

```rust
fn main() {
    let x = 5;

    let y = {
        let x = 3;
        x + 1
    };

    println!("The value of y is: {}", y);
}
```
오! 컴파일이 됩니다. 여기서 블럭의 마지막 값이 `y`에 대입이 되는 형태인데요, 이런 동일한 블럭이 바로 뒤에 나오는 `반환 값이 있는 함수` 표현에 동일하게 표현됩니다.

### 반환 값이 있는 함수

```rust
fn five() -> i32 {
    5
}

fn main() {
    let x = five();

    println!("The value of x is: {}", x);
}
```

위 코드를 보면 `five()` 함수는  세미콜론 없는 마지막 값을 반환합니다. 즉, `5`가 반환이 되는데요, 반환 유형을 `-> i32` 로 지정하는게 흥미롭습니다.

다른 코드를 보겠습니다.

```rust
fn main() {
    let x = plus_one(5);

    println!("The value of x is: {}", x);
}

fn plus_one(x: i32) -> i32 {
    x + 1
}
```

x에는 x +1의 값인 6이 대입됩니다. 이런 표현은 `plus_one()`함수를 `x + 1`로 치환했을 때 `let x = x + 1`이므로 매우 자연스럽게 보입니다. 단지 어색할 뿐이죠.

## 코멘트
Rust 세계에서의 코멘트는 //, /\* \*/로 표현하며 다른 언어의 그것과 별 반 다르지 않습니다. Crate에 관련된 다른 표현의 주석도 있다고 한는데 그건 Crate를 다두는 때 살펴보도록 합시다.

## 제어 흐름
### if 표현식
```rust
fn main() {
    let number = 3;

    if number < 5 {
        println!("condition was true");
    } else {
        println!("condition was false");
    }
}
```
부지런히 타이밍 하고 있나요? 타이핑 할 때는 내가 왜 칠까 생각을 멈추고 그냥 타이핑 합시다! 위 코드를 보면 if문의 조건식에 괄호가 없음을 알 수 있습니다.

다음의 코드는 어떨까요?

```rust
fn main() {
    let number = 3;

    if number {
        println!("number was three");
    }
}
```
네, 오류가 납니다. 조건식에는 `bool` 변수형만 가능합니다.

다음은 같지 않음에 대한 표현 입니다.

```rust
fn main() {
    let number = 3;

    if number != 0 {
        println!("number was something other than zero");
    }
}
```

다음으로는 여러 조건에 대한 처리입니다. 다른 언어들 처럼 `else if` 문을 제공합니다.

```rust
fn main() {
    let number = 6;

    if number % 4 == 0 {
        println!("number is divisible by 4");
    } else if number % 3 == 0 {
        println!("number is divisible by 3");
    } else if number % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
}
```
잘 동작 안할 이유가 없습니다. 하지만 세련된 느낌은 없는데요, `else if`가 2개 이상의 경우 `match` 키워드를 이용할 수 있습니다.

### if문에 let 사용
지금부터는 좀 신박합니다.

```rust
fn main() {
    let condition = true;
    let number = if condition { 5 } else { 6 };

    println!("The value of number is: {}", number);
}
```
C 계열 언어의  `? :` 삼항 연산자 처럼 보이는데요,  사실 위에서 이미 소개한 형태를 한줄로 표현한 것 뿐입니다. 더 직관적이고 중괄호에 의해 연산 우선순위에 방해받지 않습니다.

하지만 다음의 코드처럼 변수형이 다르면 오류가 발생합니다. (당연하지요)

```rust
fn main() {
    let condition = true;

    let number = if condition { 5 } else { "six" };

    println!("The value of number is: {}", number);
}
```

### 루프를 사용한 반복
Rust에는 loop, while, for문으로 반복문을 제공합니다.

다음 코드는 loop 문입니다.

```rust
fn main() {
    loop {
        println!("again!");
    }
}
```

C 계열의 언어의 `while (true)`보다 좀더 간결합니다.

### loop에서 값 변환
loop의 반환으로 값을 바로 받을 수 있습니다. Rust 언어 설계자 짱이네요! 왜냐하면 이런 일이 실 프로그래밍에서 비일비재 하거든요.

```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };

    println!("The result is {}", result);
}
```

멋있는 코드입니다. 점점 Rust 코드가 멋있어 보입니다.

### while문으로 조건 루프

네, 당연히 이런 방식의 사용도 가능해야 겠죠.

```rust
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{}!", number);

        number -= 1;
    }

    println!("LIFTOFF!!!");
}
```
네. 예상대로 잘 동작을 합니다. 조건문에 괄호가 없는 듯의 어색한 부분은 컴파일러가 정확히 오류로 잡아주기 때문에 조금씩 적응되어가고 있습니다.

### for문을 이용해 컬렉션을 루핑하기

다음은 index를 이용한 배열 접근 루핑입니다.
```rust
fn main() {
    let a = [10, 20, 30, 40, 50];
    let mut index = 0;

    while index < 5 {
        println!("the value is: {}", a[index]);

        index += 1;
    }
}
```

완벽하게 동작하지만 경계 지정을 해야 하고 별도의 인덱스 변수를 사용해야 합니다. 아름답지 않아요. `for`문을 이용해 다음처럼 표현할 수 있습니다.

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a.iter() {
        println!("the value is: {}", element);
    }
}
```
사실 C# 개발자라면 저런 표현을 `foreach`문으로 하기 때문에 익숙합니다. 

아래의 코드처럼도 가능합니다. 

```rust
fn main() {
    for number in (1..4).rev() {
        println!("{}!", number);
    }
    println!("LIFTOFF!!!");
}
```
잘됩니다! Rust는 동적 언어 수준의 간결한 표현으로 네이티브 코드를 생성하는 멋진 결과물을 만들었군요!
