## Rust #8: 8장 공통 컬렉션

## 개요
이 장은 Rust에서 대표적으로 사용하는 벡터(Vector, Vec<T>), 문자열(String), 해시맵(Hash Map, HashMap<K, V>)을 통해 Rust에서 제공하는 컬렉션에 대해 이해하는 시간을 갖도록 합시다.

## 벡터로 값 목록 저장하기
Rust에서는 변경할 수 있는 연속적인 값을 벡터라 하며 `Vec<T>` 컬렉션 유형입니다. 벡터는 동일한 유형의 값만 저장할 수 있으며 리스트 라고도 불리어 집니다.

### 새 벡터 만들기

다음과 같이 코딩해 봅시다.

```rust
let v: Vec<i32> = Vec::new();
// let v = Vec::<i32>::new();
```
Rust에서는 새로운 할당을 약속처럼 `new()` 메소드를 통해 합니다.

다음의 코드처럼 초기 값을 가지는 벡터를 생성할 수 도 있습니다.

```rust
let mut v = vec![1, 2, 3];
```

유추할 수 있는 값이 주어지면 Rust는 똑똑하게 적절한 유형을 찾아줍니다.
위의 vec 매크로는 컴파일 시 해석 되어서 코드로 변환 되는데요, 대략 다음의 코드처럼 변환됩니다.

```rust
// let mut v = vec![1, 2, 3];
// 실제 vec! 동작과는 동일하지 않는 예시입니다.
let mut v: Vec<i32> = Vec::new();
v.push(1);
v.push(2);
v.push(3);
```

### 벡터의 요소 읽기
다른 프로그래밍 언어와 유사하게 인덱싱을 통한 접근과 메소드를 통한 접근을 제공합니다.

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];

    let third = &v[2];
    println!("{}", &third);

    let some_third = v.get(2);
    match some_third {
        Some(n) => println!("{}", n),
        None => println!("None!"),
    }
}
```

차이점은 `get()`메소드의 경우 `Option` 열거형을 반환한다는 점입니다.

다음의 코드를 통해 `get()` 메소드의 이점을 확인할 수 있습니다.

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];

    let does_not_exist = v.get(100);
    let does_not_exist = &v[100];
}
```

`v.get()`을 통해서는 인덱스 범위를 벗어났지만 오류가 발생하지 않고  `Option.None`을 반환하는 반면, `v[]`로 인덱싱 한 경우 런타임 에러가 발생합니다.

다음의 코드는 변수에 `mut` 키워드를 주었을 때 발생하는 에러 유형을 살펴보겠습니다.

```rust
let mut v = vec![1, 2, 3, 4, 5];

let first = &v[0];

v.push(6);

println!("{}", first);
```

이 코드는 컴파일 오류가 발생합니다. 왜냐하면, `first` 변수가 `v` 벡터의 특정 원소를 참조하고 있는데, `v.push(6)`에 의해서 목록이 변형되기 때문입니다.

### 벡터의 값 반복

참조와 역참조를 이용해 Rust에서는 벡터 순회 시 값의 변형을 효율적으로 처리합니다.

```rust
fn main() {
    let mut v = vec![100, 32, 57];
    for i in &mut v {
        *i += 50;
    }

    println!("{:?}", v);
}
```

출력
```shell
[150, 82, 107]
```

보시는 바와 같이 벡터의 값을 꺼내와 그 값에 `50`을 더하는 간결한 코드입니다.

### 열거형을 사용하여 여러 유형 저장
열거형도 벡터의 요소 유형이 될 수 있습니다.

```rust
fn main() {
    #[derive(Debug)]
    enum SpreadsheetCell {
        Int(i32),
        Float(f64),
        Text(String),
    }

    let row = vec![
        SpreadsheetCell::Int(3),
        SpreadsheetCell::Text(String::from("blue")),
        SpreadsheetCell::Float(10.12),
    ];

    println!("{:?}", row);
}
```

출력
```shell
[Int(3), Text("blue"), Float(10.12)]
```

## 문자열이 있는 UTF-8 인코딩 텍스트 저장

문자열도 컬렉션이라고 할 수 있습니다. 문자열을 문자의 연속된 집합으로 정의 내릴 수 있기 때문인데요, 실제로 문자열은 다음처럼 정의 되어 있습니다.

```rust
pub struct String {
    vec: Vec<u8>,
}
```

### 문자열이란 무엇입니까?
Rust에서 사용하는 문자열 인코딩 방식은 `UTF-8` 입니다. `UTF-8`은 인터넷이 활성화된 꽤 최근에 대중적으로 사용하게 된 인코딩 방식인데, 그런 이유로 오래된 프로그래밍 언어는 문자열 인코딩 방식으로 `UTF-16` 또는 좀 더 오래된 언어의 경우 `ANSI`  인코딩 방식을 사용합니다.
Rust는 다른 프로그래밍 언어에 비해 최근에 등장한 언어에 속하는데 그 때문인지 Rust의 문자는 `UTF-8` 인코딩 방식을 사용합니다.

문자열은 문자의 집합입니다. Rust의 문자열은 String 및 문자열 슬라이스를 의미하는 &str로 사용할 수 있습니다. 그리고 문자형은 char인데, char는 `UTF-8` 인코딩에 의해 문자를 표현합니다.

### 새 문자열 만둘기

다음과 같은 방법으로 새 문자열을 만들 수 있습니다.

```rust
let mut s = String::new();
```

다음과 같이 문자열 리터럴에서 String을 생성할 수 있고,

```rust
    let data = "initial contents";
    let s = data.to_string();

    let s = "initial contents".to_string();
```

`String::from()`을 통해 생성할 수도 있습니다.

```rust
let s = String::from("initial contents");
```

Rust의 문자열은 `UTF-8` 인코딩을 사용하므로 다음과 같이 다양한 언어를 표현할 수 있습니다.

```rust
    let hello = String::from("السلام عليكم");
    let hello = String::from("Dobrý den");
    let hello = String::from("Hello");
    let hello = String::from("שָׁלוֹם");
    let hello = String::from("नमस्ते");
    let hello = String::from("こんにちは");
    let hello = String::from("안녕하세요");
    let hello = String::from("你好");
    let hello = String::from("Olá");
    let hello = String::from("Здравствуйте");
    let hello = String::from("Hola");
```

### 문자열 업데이트

#### push_str과 push로 문자열 추가

다음의 코드와 같이 문자열을 추가할 수 있습니다.

```rust
let mut s = String::from("foo");
s.push_str("bar");
```

다음의 코드를 통해 `push_str()`에 전달된 문자열의 소유권이 여전히 유효함을 알 수 있습니다.

```rust
    let mut s1 = String::from("foo");
    let s2 = "bar";
    s1.push_str(s2); // &str 이므로 소유권을 가져가지 않음
    println!("s2 is {}", s2);
```

단일 문자를 추가하는 것도 어렵지 않습니다.

```rust
let mut s = String::from("lo");
s.push('l');
```

s는 "lol"이 됩니다.

#### + 연산자 또는 format! 매크로와의 연결

다음의 코드를 보시죠.

```rust
    let s1 = String::from("Hello, ");
    let s2 = String::from("world!");
    let s3 = s1 + &s2;
```

s1은 소유권을 잃고 더이상 사용할 수 없게 됩니다. s3는 결합 메소드의 형태에 따라 참조형이여야 합니다.

```rust
// 이것은 정확한 표준 라이브러리의 정의는 아닙니다.
fn add(self, s: &str) -> String { ... }
```

세개 이상의 문자열 결합도 가능합니다.

```rust
let s1 = String::from("tic");
let s2 = String::from("tac");
let s3 = String::from("toe");

let s = s1 + "-" + &s2 + "-" + &s3;
```

결합 키워드로 인해 가독성이 조금은 떨어졌습니다. 이것을 다음의 코드로 변경할 수 있습니다.

```rust
let s1 = String::from("tic");
let s2 = String::from("tac");
let s3 = String::from("toe");

let s = format!("{}-{}-{}", s1, s2, s3);
```

조금 더 읽기 쉬운 코드가 되었습니다.

### 문자열로 인덱싱

다음의 코드는 컴파일 오류가 발생합니다.

```rust
let s1 = String::from("hello");
let h = s1[0];
```

Rust의 문자는 `UTF-8` 인코딩을 사용하며 가변입니다. 문자열은 이런 문자의 집합으로 이루어졌기 때문에 인덱스 접근을 허용하면 자칫 치명적인 성능 문제가 발생하는 코드를 작성할 수 있게 됩니다. 이로 인해서 Rust의 문자열에는 인덱스 접근을 지원하지 않습니다.

#### 내부 표현

좀 더 자세히 살펴 봅시다. 다음의 코드를 보면,

```rust
let hello = String::from("Hola");
```
`hello`의 바이트 길이는 4바이트입니다. 다음으로 다음의 코드를 보면,

```rust
let hello = String::from("Здравствуйте");
```

`hello`의 바이크 길이가 총 12바이트일것 같지만, 실제로는 총 24바이트의 길이를 가지게 됩니다.

### 바이트 및 스칼라 값과 그래프 클러스터! 오 저런!

실제로 다양한 나라의 언어에서 UTF-8을 이용해 Rust의 char 유형의 유니코드 스칼라 값이 최종 문자열이 되지 않습니다.

```rust
['न', 'म', 'स', '्', 'त', 'े']
```

이 char 목록은, 최종적으로 다음의 문자들이 됩니다.

```rust
["न", "म", "स्", "ते"]
```

Rust는 이런 특징 때문에 문자열의 인덱스를 제공하지 않는 것으로 결정이 되었습니다.

## 슬라이싱 문자열

그러므로 슬라이싱 문자열을 조심스럽게 사용해야 합니다.

```rust
fn main() {
    let name = "디모이";
    
    //let first = &name[0..1]; // 런타임 오류
    let first = &name[0..3];
    println!("{}", first);
}
```

## 문자열을 반복하는 방법

문자열을 반복하기위한 방법은 없는것일까요? 다행이 `chars()`를 통해 가능합니다.

``rust
fn main() {
    let name = "디모이";

    //let first = &name[0..1]; // 런타임 오류
    for c in name.chars() {
        println!("{}", c);
    }
}
```

바이트로의 접근은 `bytes()`로 가능합니다.

```rust
for b in name.bytes() {
    println!("{}", b);
}
```

## 문자열은 그렇게 간단하지 않습니다.
`UTF-8` 인코딩 방식은 현대에 가장 대표적으로 사용하는 인코딩 방식이지만 간단하지는 않습니다. Rust는 `UTF-8` 인코딩을 사용하기로 결정하게 되면서 가변 바이트 길이의 문자 처리에 대한 어려움이 생겼지만 반대로 `UTF-8`기반의 문자열은 별도의 처리 없이 언어 차원에서 처리할 수 있는 장점이 있습니다.

## 해시맵에 연결된 키로 값 저장

Rust는 키로 값에 접근할 수 있는 `HashMap<K, V>`를 제공합니다. 다른 프로그래밍 언어에서는 `Dictionary`라고 하기도 합니다.

### 새 해시맵 생성

다음의 코드로 새 해시맵을 생성할 수있습니다.

```rust
    use std::collections::HashMap;

    let mut scores = HashMap::new();

    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);
```

다음의 코드는 키목록과 값 목록에서 해시맵을 만드는 코드입니다.

```rust
    use std::collections::HashMap;

    let teams = vec![String::from("Blue"), String::from("Yellow")];
    let initial_scores = vec![10, 50];

    let mut scores: HashMap<_, _> =
        teams.into_iter().zip(initial_scores.into_iter()).collect();
```

### 해시맵 및 소유권

해시맵도 Rust 소유권의 영향을 받습니다.

```rust
    use std::collections::HashMap;

    let field_name = String::from("Favorite color");
    let field_value = String::from("Blue");

    let mut map = HashMap::new();
    map.insert(field_name, field_value);

    println!("{}", &field_name); // 소유권이 map으로 넘어갔으므로 오류!
    println!("{}", &field_value); // 소유권이 map으로 넘어갔으므로 오류!
```

### 해시맵 값 액세스

```rust
    use std::collections::HashMap;

    let mut scores = HashMap::new();

    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    let team_name = String::from("Blue");
    let score = scores.get(&team_name);
```

벡터와 유사하게 `get()` 메소드를 통해 `Option`으로 값을 취할 수 있습니다.

(키, 값)으로 해쉬맵을 순회하는 방법은 다음과 같습니다.

```rust
    use std::collections::HashMap;

    let mut scores = HashMap::new();

    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    for (key, value) in &scores {
        println!("{}: {}", key, value);
    }
```

### 해시맵 업데이트

해시맵은 동일한 키로 `insert()`할 경우 값을 갱신하도록 합니다.

```rust
    use std::collections::HashMap;

    let mut scores = HashMap::new();

    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Blue"), 25);

    println!("{:?}", scores);
```

#### 키에 값이 없는 경우에만 값 삽입

`entry()` 및 `or_insert()` 메소드를 이용해 키에 값이 없는 경우에 값을 삽입할 수 있도록 할 수 있습니다.

```rust
    use std::collections::HashMap;

    let mut scores = HashMap::new();
    scores.insert(String::from("Blue"), 10);

    scores.entry(String::from("Yellow")).or_insert(50);
    scores.entry(String::from("Blue")).or_insert(50);

    println!("{:?}", scores);
```

#### 이전 값을 기반으로 값 업데이트

흥미로운 부분입니다. 다음의 코드를 보시죠.

```rust
    use std::collections::HashMap;

    let text = "hello world wonderful world";

    let mut map = HashMap::new();

    for word in text.split_whitespace() {
        let count = map.entry(word).or_insert(0);
        *count += 1;
    }

    println!("{:?}", map);
```

역참조를 통해 해시맵의 값 참조를 할 수 있고 가공할 수 있습니다. 위의 코드는 단어의 개수를 세는 간단한 코드인데, 매우 짧은 코드로 효율적인 동작을 하는 실행파일을 만들 수있습니다.

## 해싱 함수

해시맵의 키는 문자열 뿐만 아니라 해시값을 제공하는 무엇이든 가능 합니다.
Rust에서 사용하는 해싱 함수는 가장 빠른 해싱 알로기즘은 아니지만 DoS 공격에 저항할 수 있는 해시 기능을 제공합니다. 이 외에 `crates.io`에 일반적인 해싱 알고리즘 관련 라이브러리를 확인할 수 있습니다.

## 정리

벡터, 문자열, 해시맵에 대해 알아보았습니다. 이 세가지 컬렉션은 Rust 언어 뿐만 아니라 대부분의 프로그래밍 언어에서 로직을 구현하기 위한 필수 컬렉션입니다.  본 장을 살펴보면서 Rust의 벡터와 문자열, 해시맵에 대해 익숙해졌기를 바랍니다.