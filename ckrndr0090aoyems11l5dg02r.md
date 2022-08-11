## Rust #5: 5장 구조체를 사용하여 관련 데이터 구조화

## 개요
의미있는 값을 그룹으로 묶어 관리하면 좀 더 효율적으로 프로그래밍 할 수 있습니다. Rust에도 마찬가지로 여러 관련된 값을 하나의 그룹으로 패키징 하여 사용자 정의 데이터 유형을 만들 수 있으며 이를 구조체(struct)라고 합니다.

## 구조체 정의 및 인스턴스화
구조체는 튜플과 유사한 값의 집합입니다. 차이점은 값의 의미를 명확하게 전달할 수 있도록 필드 이름을 지정할 수 있습니다. 또한 변경 가능한 값을 가질 수도 있는데요, 지금 부터 살펴봅시다.

다음은 일반적인 Rust의 구조체를 보여줍니다.
```rust
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}
```
User라는 구조체 형을 정의하며, User는 각각 username과 email, sign_in_count와 active라는 필드명을 가진 형을 담게 됩니다. 이 필드들의 공통점은 사용자(User)의 속성이라는 것인데요, 이런 식으로 관련된 값을 하나의 구조체 형태로 만들어 사용할 수 있습니다.

다음은 구조체로 만든 사용자 데이터 형을 다음과 같이 선언하여 인스턴스화 하는 것을 보여줍니다.

```rust
    let user1 = User {
        email: String::from("someone@example.com"),
        username: String::from("someusername123"),
        active: true,
        sign_in_count: 1,
    };
```

위와 같이 구조체를 인스턴스화 하여 user1에 할당할 수 있습니다. 구조체는 Rust의 변수의 선언과 마찬가지로 불변하므로 생성한 후 필드를 수정할 수 없는데요, 다음과 같이 `mut` 키워드를 더해 수정 가능한 구조체를 선언할 수 있습니다.

```rust
    let mut user1 = User {
        email: String::from("someone@example.com"),
        username: String::from("someusername123"),
        active: true,
        sign_in_count: 1,
    };

    user1.email = String::from("anotheremail@example.com");
```

`mut` 키워드를 주면 특정 필드만 변경 가능한 것으로 표시하는 것은 허용하지 않습니다.  다른 표현식과 마찬가지로 구조체 또한 함수 본문의 마지막 표현식으로 구성하여 새 인스턴스를 암시적으로 반환할 수 있습니다.

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email: email,
        username: username,
        active: true,
        sign_in_count: 1,
    }
}
```

### 변수와 필드의 이름이 같을 경우 필드명 생략 가능

구조체 필드와 매개변수 이름이 동일할 경우 필드명을 생략할 수 있습니다.

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email,
        username,
        active: true,
        sign_in_count: 1,
    }
}
```

### 구조체 업데이트 구문을 이용하여 다른 인스턴에서 인스턴스 생성

다른 인스턴스의 필드 값을 그대로 가져야 사용하는 경우가 자주 있습니다. Rust는 그럴 경우를 다음처럼 축약하여 사용할 수 있습니다.

```rust
    let user2 = User {
        email: String::from("another@example.com"),
        username: String:: from("anotherusername567"),
        active: user1.active,
        sign_in_count: user1.sign_in_count,
    };
```
이 코드를

```rust
    let user2 = User {
        email: String::from("another@example.com"),
        username: String:: from("anotherusername567"),
        ..user1
    };
```
로 `active`와 `sign_in_count` 필드 값을 그대로 가져와 사용할 수 있습니다.

극단적으로 다음과 같이 쓸수도 있습니다.

```rust
    let user2 = User { ..user1 };
```

### 명명된 필드가 없는 튜플 구조체를 사용하여 다른 유형 만들기

필드명이 주어지지 않아도 그 의미가 유추가 되는 구조체의 경우 필드를 생략하고 정의를 단순화 할 수 있습니다.

```rust
    struct Color(i32, i32, i32);
    struct Point(i32, i32, i32);

    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);
```
Color 구조체는 R, G, B로 이루어졌다는 것을 유추할 수 있으며 그러한 맥락으로 필드명을 생략할 수 있습니다. Point도 마찬가지로 X, Y, Z로 이루어졌다는 것을 유추할 수 있으며 마찬가지로 필드명을 생략할 수 있는 대상이 됩니다.

### 필드가 없는 단위 유사 구조체
구조체를 필드 없이 정의할 수 도 있습니다. 이에 대해서는 10장에서 논의될 예정입니다.

### 구조체의 소유권
구조체의 소유권도 변수의 소유권과 동일합니다.

```rust
    let user3 = user2;

    // 소유권이 user3으로 넘어갔으므로 `user2.email`을 쓸 수 없음
    println!("{}", user2.email);
```

필드의 소유권도 마찬가지입니다.

```rust
    let user2 = User { ..user1 };

    println!("{}", user2.email);
    // email이 String 형이고 `user2.email`로 소유권이 넘어갔으므로 `user1.email`을 쓸 수 없음
    println!("{}", user1.email);
```

## 구조체를 사용한 예제 프로그램

구조체를 이해하기 위해 직사각형의 면적을 계산하는 프로그래을 작성해 봅시다.

```rust
fn main() {
    let width1 = 30;
    let height1 = 50;

    println!(
        "The area of the rectangle is {} square pixels.",
        area(width1, height1)
    );
}

fn area(width: u32, height: u32) -> u32 {
    width * height
}
```

잘 작동합니다. 하지만 서로 관련이 있는 `width`와 `height`가 별개의 변수로 사용되고 함수의 인자로 전달되었기 때문에 코드 가독성이 떨어지고 수정시 어렵습니다.

### 튜플을 사용한 리팩토링

`width`와 `height`는 서로 관련이 있으므로 이를 튜플로 표현해 봅시다.

```rust
fn main() {
    let rect1 = (30, 50);

    println!(
        "The area of the rectangle is {} square pixels.",
        area(rect1)
    );
}

fn area(dimensitions: (u32, u32)) -> u32 {
    dimensitions.0 * dimensitions.1
}
```

잘 작동합니다. 두 값이 튜플로 서로 묶였기 때문에 좀 더 의미가 명확해집니다. 하지만 필드명이 없기 때문에 그 의미가 명확하지 않다는 단점이 있습니다.

### 구조체를 사용한 리팩토링: 더 많은 의미 추가

이제 위의 코드를 구조체로 리팩토링 해 봅시다.

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!(
        "The area of the rectangle is {} square pixels.",
        area(&rect1)
    );
}

fn area(rectangle: &Rectangle) -> u32 {
    rectangle.width * rectangle.height
}
```

구조체를 이용해서 직사각형의 속성이 더 명확해졌습니다. 필드명을 이용하기 때문에 값 계산 시 혼돈이 없습니다.

### 파생된 특성으로 유영한 기능 추가하기

어노테이션을 이용하면 다양한 속성을 구조체에 부여할 수 있습니다.

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    // Rectangle은 `std::fmt::Display`를 구현하지 않았으므로 오류가 발생함
    println!("rect1 is {}", rect1);
}
```

구조체에 `#[derive(Debug)]` 어노테이션을 부여하고 포맷에 `{}`대신 `{:?}`으로 변경하면 오류가 제거됩니다. Debug의 구현을 이용해 정상적으로 출력할 수 있게 됩니다.

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    // 오류 발생하지 않음
    println!("rect1 is {:?}", rect1);
}
```

Rectange 구조체는 사각형의 너비와 높이를 갖는 훌륭한 사용자 유형입니다. 하지만 사각형의 면적을 구하는 함수가 Rectangle 구조체 외부에 있습니다. 사각형의 면적은 사각형과 밀접한 관련이 있으므로 내부의 함수로 제공하면 좋을 것 같습니다.

## 메서드 구문

메소드는 함수와 유사하게 fn 키워드와 이름으로 선언하고 매개변수화 반환값을 가집니다. 그러나 메소드는 구조체 내에 정의되고 첫번째 매개변수가 항상 self라는 점에서 다릅니다.

### 메소드 정의

면적을 구하는 함수를 `Rectangle`의 메소드로 리펙토링 해봅시다.

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!(
        "The area of the rectangle is {} square pixels.",
        rect1.area()
    )
}
```

Rectangle의 메소드를 정의하기 위해 `impl` 구현 블록을 이용하여 메소드를 정의할 수 있습니다.

### 더 많은 매개변수가 있는 메서드
메소드의 첫번째 인자가 `&self`라는 것만 제외하고 함수와 동일 합니다.

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };
    let rect2 = Rectangle {
        width: 10,
        height: 40,
    };
    let rect3 = Rectangle {
        width: 60,
        height: 45,
    };

    println!("Can rect1 hold rect2? {}", rect1.can_hold(&rect2));
}
```

### 관련 기능
`&self`를 인자로 사용하지 않아도 됩니다. self를 사용하지는 않지만 구조체에 포함되어 있기 때문에 이를 이런 유형을 `연결된 함수`라고 합니다. 구조체의 필드나 메소드에 접근할 필요가 없는 경우 이렇게도 사용되는데요, 대표적인 사용이 새 인스턴스를 반환하는 생성자에 자주 사용됩니다.

```rust
impl Rectangle {
    fn square(size: u32) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }
}
```

### 다중 impl 블록
다중 impl 블록을 통해 메소드를 정의할 수도 있습니다. 굳이 그럴 필요는 없지만 유용한 경우가 있습니다. 10장에서 다중 블록이 유용한 경우를 다룰 것입니다.

## 요약

구조체를 사용하면 의미있는 사용자 지정 유형을 만들 수 있습니다. 구조체를 사용하면 연결된 데이터 조각을 구조체라는 단위로 유지하고 코드를 명확하게 하기 위해 각 조각에 이름을 지정할 수 있습니다. 메소드를 이용하면 구조체의 인스턴스가 가지는 값에 따라 동작을 지정할 수 있고 연결된 함수를 사용하면 인스턴스를 사용할 수 없는 상태에서 구조체에 특정한 기능을 제공할 수 있습니다.

Rust에서는 구조체와 더불어 다른 언어와 다르게 강력한 열거형 기능을 제공합니다.
