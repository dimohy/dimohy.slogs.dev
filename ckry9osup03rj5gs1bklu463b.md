## Rust #6: 6장 열거형과 패턴 매칭

## 개요
열거형(enum)은 특정 값을 식별할 수 있는 형태로 열거하여 정의하는 것을 말합니다. 가장 대표적인 예시가 `요일`이 될 수 잇겠는데요, 한 주는 '월, 화, 수, 목, 금, 토, 일'이라는 요일로 구성될 수 있습니다.

Rust에서의 열거형에 대해 알아보고 열거형을 이용한 패턴 일치 방법과, 다른 언어와는 다른 Rust만의 열거형 특징을 알아봅시다.

## 열거형의 정의
열거형은 어떤 상태의 값을 의미 있는 단어로 구성된 형태로 표현할 수 있는 유용한 방법입니다. 더 나아가서 Rust 열거형은 열거 값에 따른 별도의 필드 값을 소유할 수 있도록 디자인 되었습니다. 아래의 코드는 IP 주소를 표현할 때 V4와 V6의 표현 방식이 다르다는 것을 이용해 예시를 들고 있습니다.

```rust
enum IpAddrKind {
    V4,
    V6
}
```

일반적인 형태의 열거형 사용법입니다.  `IpAddrKind`는 이제 다른 곳에서 사용할 수 있는 사용자 지정 데이터 유형 입니다.

### 열거형 값

열거형을 이용해 사용자 지정 데이터 유형을 만들었으면 다음과 같이 인스턴스를 생성할 수 있습니다.

```rust
    let four = IpAddrKind::V4;
    let six = IpAddrKind::V6;
```

`four` 변수는 IP 주소 중 `V4`를 의미 하는 인스턴스이며, `six` 변수는 IP 주소 중 `V6`를 의미하는 인스턴스가 됩니다. 각각의 값은 내부적으로 문자열이 아닌 효율적인 스칼라 값으로 치환됩니다.

이 값을 함수에 전달하는 방식도 다른 유형과 별반 다르지 않습니다.

```rust
fn route(ip_kind: IpAddrKind) {}
```

그리고 다음과 같이 함수에 IpAddrKind으 인스턴스를 전달할 수 있습니다.

```rust
    route(IpAddrKind::V4);
    route(IpAddrKind::V6);
```

Rust의 열거형을 사용하면 더 많은 이점을 얻을 수 있습니다. 다른 언어에서 열거형을 사용하는 방식을 Rust의 코드로 전개해보겠습니다.

```rust
    enum IpAddrKind {
        V4,
        V6,
    }

    struct IpAddr {
        kind: IpAddrKind,
        address: String,
    }

    let home = IpAddr {
        kind: IpAddrKind::V4,
        address: String::form("127.0.0.1"),
    };

    let loopback = IpAddr {
        kind: IpAddrKind::V6,
        address: String::from("::1"),
    };
```

특별히 이상할 것 없는 코드입니다. 대부분의 프로그래밍 언어들이 위 방식의 코드 형태를 취하고요, 개념적으로나 코드상으로 문제 없는 코드입니다. 하지만 Rust에서는 열거형의 기능을 확장해서 열거 값 마다 필드 값을 취할 수 있도록 했습니다. 이 기능을 활용한 코드는 다음과 같습니다.

```rust
    enum IpAddr {
        V4(String),
        V6(String),
    }

    let home = IpAddr::V4(String::from("127.0.0.1"));

    let loopback = IpAddr::V6(String::from("::1"));
```

와, IP주소의 종류에 따라 그 구성 값이 달라지므로 이런 상황에서는 위의 코드가 더 깔끔하고 직관적으로 보입니다. 대단합니다! 그런데 여기서 끝나는 것이 아닙니다.

```rust
    enum IpAddr {
        V4(u8, u8, u8, u8),
        V6(String),
    }

    let home = IpAddr::V4(127, 0, 0, 1);

    let loopback = IpAddr::V6(String::from("::1"));
```

와, 그렇습니다. 열거형의 각 열거 값에 개별 필드를 다르게 줄 수 있습니다. 이렇게 되면 Rust의 열거형을 이용해 객체지향의 `다형성`을 열거형을 통해 아름답게 표현할 수 있게 되는 것 같습니다.

다음의 코드를 통해 결국에는 모든 종류의 데이터를 열거형의 값에 넣을 수 있음을 알 수 있습니다.

```rust
struct Ip4Addr {
}

struct Ip6Addr {
}

enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv6Addr)
}
```

다음의 코드는 열거형의 다른 예시를 보여줍니다.

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```
열거형의 필드도 구조체의 그것처럼 튜플처럼 필드명을 생략할 수 도 있고, 필드명을 명시할 수 도 있음을 알 수 있습니다. 그리고 Message의 각각의 값은 다양한 필드 값을 가지고 다 다르지만, 각각 Message의 유형이라는 의미로 객체지향의 다형성을 Rust의 열거형으로 잘 표현하고 있습니다.

물론 이것을 구조체로 표현도 다음처럼 가능합니다.

```rust
struct QuitMessage;
struct MoveMessage {
    x: i32,
    y: i32,
}
struct WriteMessage(String);
struct ChangeColorMessage(i32, i32, i32);
```

하지만 구조체로 Message를 표현하게 되면 이런 종류의 메시지를 취하는 함수를 쉽게 정의할 수 없습니다.

열거형에 메소드를 만들수도 있습니다.

```rust
    impl Message {
        fn call(&self) {
        }
    }

    let m = Message:Write(String::from("hello"));
    m.call();
```

이제 열거형으로 만들어진 표준 라이브러리의 `Option` 형에 대해 알아봅시다.

### Option 열거 및 Null 값 이상의 장점
Null 값은 값이 없음을 의미하는 값인데, 마치 값처럼 취급됩니다. 그래서 Null 값을 이용하는 다수의 프로그래밍 언어는 컴파일 시점에서 이것을 감지하지 못합니다. 결국에는 실행 중에 Null일 경우 관련 예외가 발생해 프로그램이 중지되는 심각한 버그가 발생하게 되는데, Rust는 그래서 Null값이 없는 언어로 디자인 되었습니다.

하지만 모든 결과가 성공할 수는 없으므로 Null 값과 유사하지만 반환값 자체는 아닌 형태가 필요한데 그렇기 때문에 Rust 표준 라이브러리는 열거형을 이용해 이것을 구현했고 이것이 `Option`입니다.

Option의 세부 구현은 좀 더 있겠지만 간단히 표현하면 다음과 같습니다.

```rust
enum Option<T> {
    None,
    Some(T),
}
```

Option은 아무 값도 아닌 `None`과 어떤 값을 의미하는 `Some`으로 표현됩니다. Some은 `T`형을 필드로 가지므로 T 그 자체라는 것을 의미합니다.

```rust
    let some_number = Some(5);
    let some_string = Some("a string");

    let absent_number: Option<i32> = None;
```

`Option`을 이용해 값이 없음을 열거형으로 표현할 수 있게 되었습니다. Option을 이용하면 Option 인스턴스가 T 자체는 아니기 때문에 값이 `None`일 경우 관련 런타임 예외를 미연에 방지할 수 있는 코드를 작성할 수 있습니다.

```rust
    let x: i8 = 5;
    let y: Option<i8> = Some(5);

    let sum = x + y;
```

이 코드는 Some(5)가 5는 아니므로 컴파일 오류가 발생합니다.

## 일치 제어 흐름 연산자
`match`를 이용해 열거형의 값에 따라 분기하는 효율적인 제어 흐름을 코드로 표현할 수 있습니다.

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

이 코드처럼 열거형의 값에 따라 분기하는 제어 흐름을 `match`로 표현할 수 있는데요, Rust의 강력한 표현을 이용해 다음처럼도 가능합니다.

```rust
fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => {
            println!("Lucky penny!");
            1
        }
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```
세미콜론(;)이 없는 마지막 값을 반환 값으로 취하기 때문에 블럭 안에 특정 코드를 실행시키고 마지막에 값을 반환하는 것도 문제가 되지 않습니다.

### 값과 바인딩 되는 패턴

`match`를 이용하면 열겨형의 값 필드를 분기 시 인자로 넘길수도 있습니다.

Quarter에 UsState 형의 필드가 있습니다.

```rust
#[derive(Debug)]
enum UsState {
    Alabama,
    Alaska,
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState),
}
```

이것을 다음처럼 인자로 넘길 수 있습니다.

```rust
fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin:Penny => 1,
        Coin:Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {:?}!", state);
            25
        }
    }
}
```

Quarter의 UsState형의 값이 `match`의 `Coin::Quarter(state)`의 `state`로 바인딩되어 사용할 수 있게 됩니다.

`Option`으로 다시 예를 들어봅시다. 값에 1을 더하는 함수로 다음을 확인합니다.

```rust
    fn plus_one(x: Option<i32>) -> Option<i32> {
        match x {
            None => None,
            Some(i) => Some(i + 1),
        }
    }

    let five = Some(5);
    let six = plus_one(five);
    let none = plus_one(None);
```

### 일치는 철저합니다.
다음의 코드처럼 비교 대상이 누락되었을 경우 Rust는 에러를 발생합니다.

```rust
    fn plus_one(x: Option<i32>) -> Option<i32> {
        match x {
            Some(i) => Some(i + 1),
        }
    }
```

`None` 케이스를 처리하지 않았기 때문인데요, 모든 대상을 매칭 하거나 다음처럼 언더바(`_`)로 이외의 처리를 해야 합니다.

```rust
    let some_u8_value = 0u8;
    match some_u8_value {
        1 => println!("one"),
        3 => println!("three"),
        5 => println!("five"),
        7 => println!("seven"),
        _ => (),
    }
```

## if let을 이용한 간결한 제어 흐름
특정 값일 때 무언가를 하고 싶고 나머지는 아무것도 하지 않는 코드는 `match`를 통해 다음처럼 코딩할 수 있습니다.

```rust
    let some_u8_value = Some(0u8);
    match some_u8_value {
        Some(3) => println!("three"),
        _ => (),
    }
```

하지만 이외의 처리는 아무것도 하지 않으므로 불필요해 보이는데요, 이를 `if let`을 통해 다음처럼 간결하게 표현할 수 있습니다.

```rust
    let some_u8_value = Some(0u8);
    if let Some(3) = some_u8_value {
        println!("three");
    }
```

또한 다음의 `match` 코드와 `if let` 코드로 이외의 처리를 할 수 있습니다.

`match`사용
```rust
let mut count = 0;
match coin {
    Coin::Quarter(state) => println!("State quarter from {:?}!", state),
    _ => count += 1,
}
```

`if let` 사용
```rust
let mut count = 0;
if let Coin::Quarter(state) = coin {
    println!("State quarter from {:?}!", state);
} else {
    count += 1;
}
```

## 정리
이번 장에서 열거형에 대해서 알아보았고 Rust 만의 열거형 특징을 파악했습니다. 표준 라이브러리인 `Option`을 통해 열거형의 값에 개별 필드를 두어서 사용하는 방법과 `match`로 분기하는 코드를 살펴보았습니다. 또한 `if let`을 이용해 해당 값을 추출해서 비교하는 방법도 살펴보았습니다.
