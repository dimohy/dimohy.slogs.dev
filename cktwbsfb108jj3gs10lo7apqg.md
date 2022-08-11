## Rust #13: 13장 함수형 언어의 특징: 반복자와 클로저

## 개요
본 장에서는 Rust가 가지고 있는 함수형 언어의 기능인 반복자와 클로저에 대해 알아볼 것입니다. 본 장은 다음의 순서로 진행합니다.

- 클로저(Closures): 변수에 저장할 수 있는 함수와 유사한 구조
- 반복자(Iterators): 일련의 항목을 처리하는 방법
- 12장에서 I/O 프로젝트를 개선하기 위해 두 가지 기능을 사용하는 방법
- 이 두 기능의 성능

스포일러로 미리 말씀 드리지만, 클로저와 반복자를 사용하더라도 Rust는 성능의 어떠한 성능 하락이 없습니다.

## 클로저: 환경을 캡쳐할 수 있는 익명 함수
Rust도 함수를 변수에 저장 할 수 있습니다. 이를 클로저라고 하는데요, 함수명이 없는 익명 함수 형태로 변수에 저장할 수 있습니다.

클로저를 사용하는 방법에 대해 차차 알아보도록 합시다.

### 클로저로 동작의 추상화 만들기
맞춤형 운동 계획을 생성하는 앱을 만든다고 칩시다. 여기서 중요한 것은 실제 알고리즘이 아니라 이 알고리즘이 얼마나 걸리는가, 그리고 그것을 한번만 호출하도록 해서 사용자가 필요 이상으로 기다리게 하지 않게 하는 것입니다.

```rust
use std::thread;
use std::time::Duration;

fn simulated_expensive_calculation(intensity: u32) -> u32 {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    intensity
}
``` 

`simulated_expensive_calculation()`함수는 알고리즘을 시뮬레이션 하는 가짜 함수입니다.

필수 입력은 다음과 같습니다.

- 저강도 운동 또는 고강도 운동을 원하는지 여부를 나타내는 사용자의 강도 번호
- 운동 계획에 의해 생성되는 임의의 숫자

```rust
fn main() {
    let simulated_user_specified_value = 10;
    let simulated_random_number = 7;

    generate_workout(simulated_user_specified_value, simulated_random_number);
}
```

이제 `generate_workout()`함수를 구현해 봅시다.

```rust
fn generate_workout(intensity: u32, random_number: u32) {
    if intensity < 25 {
        println!(
            "Today, do {} pushups!",
            simulated_expensive_calculation(intensity)
        );
        println!(
            "Next, do {} situps!",
            simulated_expensive_calculation(intensity)
        );
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                simulated_expensive_calculation(intensity)
            );
        }
    }
}
```

조건에 의해서 `simulated_expensive_calculation()` 함수가 여러번 호출될 수 있다는 것을 가정해 봅시다. `intensity`가 25보다 작을 경우가 그런 경우가 됩니다. 함수가 여러번 호출되는 것을 개선하기 위해 리팩토링 해봅시다.

#### 함수를 사용한 리팩토링
우리는 함수의 결과 값을 미리 받고, 그 것을 적용하는 것으로 리펙토링 할 수 있습니다.

```rust
fn generate_workout(intensity: u32, random_number: u32) {
    let expensive_result = simulated_expensive_calculation(intensity);

    if intensity < 25 {
        println!("Today, do {} pushups!", expensive_result);
        println!("Next, do {} situps!", expensive_result);
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!("Today, run for {} minutes!", expensive_result);
        }
    }
}
```

하지만 이 코드도 문제가 있습니다. 어떤 조건과 상관없이 `simulated_expensive_calculation()`함수가 호출됨으로써 알고리즘이 처리되는 시간만큼 기다려야 한다는 것입니다.

#### 클로저의 저장소 코드로 리팩토링
함수의 결과를 저장하는 대신 클로저를 이용해서 익명 함수를 변수로 저장 한 후 필요할 때 사용할 수 있습니다.

```rust
    let expensive_closure = |num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };
```

이제 클로저를 적용한 코드는 다음과 같습니다.

```rust
fn generate_workout(intensity: u32, random_number: u32) {
    let expensive_closure = |num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };

    if intensity < 25 {
        println!("Today, do {} pushups!", expensive_closure(intensity));
        println!("Next, do {} situps!", expensive_closure(intensity));
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                expensive_closure(intensity)
            );
        }
    }
}
```

하지만 다시 첫 번째   문제가 발생합니다. 이것이 함수 호출과 다른 점이 무엇인가요? 지금까지는 결과적으로 완전히 동일 합니다.

### 클로저 유형 추론 및 주석
클로저는 유형 추론 및 반환 값의 주석을 달지 않아도 어떻게 사용 되는지 확인해 Rust 컴파일러가 유추해 적용합니다.

위의 코드에 의해 클로저는 다음과 같이 해석됩니다.

```rust
    let expensive_closure = |num: u32| -> u32 {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };
```

유형 주석이 추가되면 클로저 구문은 함수 구문과 유사해 집니다. 동일한 코드가 어떻게 유형 추론이 되는지를 아래의 코드로 살펴볼 수 있습니다.

```rust
fn  add_one_v1   (x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```

하지만 유형 추론이 모든 형을 받을 수 있다는 것을 의미하지는 않습니다. 다음의 코드는 컴파일 오류가 발생합니다.

```rust
    let example_closure = |x| x;

    let s = example_closure(String::from("hello"));
    let n = example_closure(5);
```

rust는 컴파일 시점에서 유형이 결정되는 언어라는 것을 잊지 마세요.

### 제네릭 매개변수와 Fn 트레잇을 이용해 클로저 저장하기
다시 운동 생성 앱으로 돌아가 봅시다. 여전히 클로저는 함수와 동일한 문제가 있습니다. 이를 클로저의 고유 기능을 이용해 해결하는 방법을 살펴볼 텐데요, 다음은 클로저가 선택적 결과 값을 보유하는 방법에 대한 코드입니다.

```rust
struct Cacher<T>
where
    T: Fn(u32) -> u32,
{
    calculation: T,
    value: Option<u32>,
}
```

구조체가 클로저를 가질 수 있게 되었습니다. 여기서 `T 매개변수`는 `where`절을 통해 하나의 u32형의 인자를 받아 u32형의 값을 반환하는 클로저임을 표현하고 있습니다. `Cacher<T>`를 최종 구현해 봅시다.

```rust
impl<T> Cacher<T>
where
    T: Fn(u32) -> u32,
{
    fn new(calculation: T) -> Cacher<T> {
        Cacher {
            calculation,
            value: None,
        }
    }

    fn value(&mut self, arg: u32) -> u32 {
        match self.value {
            Some(v) => v,
            None => {
                let v = (self.calculation)(arg);
                self.value = Some(v);
                v
            }
        }
    }
}
```

이제 클로저를 이용해 값을 캐시해서 사용할 수 있는 구조체를 만들었습니다. 이를 기존 코드에 적용해 봅시다.

```rust
fn generate_workout(intensity: u32, random_number: u32) {
    let mut expensive_result = Cacher::new(|num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    });

    if intensity < 25 {
        println!("Today, do {} pushups!", expensive_result.value(intensity));
        println!("Next, do {} situps!", expensive_result.value(intensity));
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                expensive_result.value(intensity)
            );
        }
    }
}
```

클로저를 통해 알고리즘을 `expensive_result ` 생성 시점에서 적용했고, 이후 `expensive_result`를 사용하는 코드는 `value()`를 호출해서 알고리즘을 사용하고 있습니다. 값은 `Cacher<T>`에 의해서 캐시 되므로 이제 불필요하게 알고리즘이 중복 사용되는 문제점이 해결되었습니다.

### Cacher 구현의 한계
물론` Cacher<T>`는 현업에서 사용할 수준은 되지 않습니다. 다른 값이 입력되었을 경우는 감지하지 못합니다. 다음의 단위 테스트는 그래서 실패하게 됩니다.

```rust
    #[test]
    fn call_with_different_values() {
        let mut c = Cacher::new(|a| a);

        let v1 = c.value(1);
        let v2 = c.value(2);

        assert_eq!(v2, 2);
    }
```

`Cacher<T>`는 클로저의 유용함을 이해하기 위한 목적 정도로 생각하면 좋겠습니다.

### 클로저로 환경 캡쳐하기
클로저의 또다른 중요한 기능으로 환경을 캡쳐하는 기능에 대해 다음의 코드를 통해 알아보도록 합시다.

```rust
fn main() {
    let x = 4;

    let equal_to_x = |z| z == x;

    let y = 4;

    assert!(equal_to_x(y));
}
```

우와! 대단하지 않나요? 클로저는 함수와 다르게 클로저 외부 변수를 캡쳐할 수 있습니다. 기존의 함수는 외부 변수를 캡쳐할 수 없습니다.

```rust
fn main() {
    let x = 4;

    fn equal_to_x(z: i32) -> bool {
        z == x
    }

    let y = 4;

    assert!(equal_to_x(y));
}
```

이 코드는 컴파일 오류가 발생합니다.
클로저의 환경 캡쳐 기능은 다양한 구현에서 유용하게 쓰이게 됩니다.

그런데 외부 변수가 클로저의 선언 시점에서 소유권이 클로저 함수로 넘어가는 것은 아니므로 필요에 따라 이를 명시할 필요가 있습니다. 이때 `move` 키워드를 명시해서 소유권을 클로저로 넘길 수 있습니다.

```rust
fn main() {
    let x = vec![1, 2, 3];

    let equal_to_x = move |z| z == x;

    println!("can't use x here: {:?}", x);

    let y = vec![1, 2, 3];

    assert!(equal_to_x(y));
}
```

위의 코드는 `move` 키워드에 의해 의도한 대로 x에 대한 소유권이 클로저 함수로 넘어가 컴파일 오류가 발생합니다.

## 반복자를 사용하여 일련의 항목을 처리
Rust는 여타 현대식 언어와 마찬가지로 반복자를 제공합니다. 반복자는 대상보다 항목에 집중할 수 있도록 합니다.

```rust
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();
```

그리고 다음처럼 `for`문을 통해 순회할 수 있습니다.

```rust
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();

    for val in v1_iter {
        println!("Got: {}", val);
    }
``

### Iterator 트레잇과 next 메소드
모든 반복자 Iterator는 표준 라이브러리에 의해 정의된 트레잇입니다. 트레잇의 정의는 다음과 같습니다.

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // methods with default implementations elided
}
```

Rust 컴파일러는 이 트레잇을 구현한 대상을 항목으로 보고 `Iterator`의 `next()` 메소드를 호출해서 항목을 순회할 수 있게 합니다.

```rust
    #[test]
    fn iterator_demonstration() {
        let v1 = vec![1, 2, 3];

        let mut v1_iter = v1.iter();

        assert_eq!(v1_iter.next(), Some(&1));
        assert_eq!(v1_iter.next(), Some(&2));
        assert_eq!(v1_iter.next(), Some(&3));
        assert_eq!(v1_iter.next(), None);
    }
```

### 반복자를 사용하는 메서드
이제 `Iterator` 트레잇을 통해 Rust에서 다양한 반복자를 사용하는 메서드를 쓸 수 있게 됩니다.

```rust
    #[test]
    fn iterator_sum() {
        let v1 = vec![1, 2, 3];

        let v1_iter = v1.iter();

        let total: i32 = v1_iter.sum();

        assert_eq!(total, 6);
    }
```

### 다른 반복자를 생성하는 메서드
`Iterator` 트레잇을 구현했으면 이를 통해 표준 라이브러리에서 이미 구현된 다른 반복자로 변환할 수 있습니다.

```rust
    let v1: Vec<i32> = vec![1, 2, 3];

    v1.iter().map(|x| x + 1);
```

그리고 결과를 목록으로 변환할 수도 있죠.

```rust
    let v1: Vec<i32> = vec![1, 2, 3];

    let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();

    assert_eq!(v2, vec![2, 3, 4]);
```

### 환경을 캡처하는 클로저 사용
이제 환경을 캡처하는 클로저의 기능이 발휘됩니다.

```rust
#[derive(PartialEq, Debug)]
struct Shoe {
    size: u32,
    style: String,
}

fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter().filter(|s| s.size == shoe_size).collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn filters_by_size() {
        let shoes = vec![
            Shoe {
                size: 10,
                style: String::from("sneaker"),
            },
            Shoe {
                size: 13,
                style: String::from("sandal"),
            },
            Shoe {
                size: 10,
                style: String::from("boot"),
            },
        ];

        let in_my_size = shoes_in_size(shoes, 10);

        assert_eq!(
            in_my_size,
            vec![
                Shoe {
                    size: 10,
                    style: String::from("sneaker")
                },
                Shoe {
                    size: 10,
                    style: String::from("boot")
                },
            ]
        );
    }
}
```

여기서 `shoes_in_size()`의 ``filter()` 반복자 함수에 주목하세요. 클로저를 이용해 외부 변수인 `shoe_size`를 필터에서 사용할 수 있는 것을 알 수 있습니다. 굉장히 유용합니다!

### Iterator 트레잇을 사용하여 자체 반복자 만들기

Iterator 트레잇을 구현하면 우리의 코드도 반복자를 제공할 수 있습니다.

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}
``` 

그리고, 반복자를 구현합니다.

```rust
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```

이제 다음의 단위 테스트가 통과 됩니다.

```rust
    #[test]
    fn calling_next_directly() {
        let mut counter = Counter::new();

        assert_eq!(counter.next(), Some(1));
        assert_eq!(counter.next(), Some(2));
        assert_eq!(counter.next(), Some(3));
        assert_eq!(counter.next(), Some(4));
        assert_eq!(counter.next(), Some(5));
        assert_eq!(counter.next(), None);
    }
```

이 단위 테스트가 통가 되었다는 의미는 `for`문과 반복자 함수를 이용할 수 있게 되었다는 의미입니다.

#### 다른 Iterator 트레잇 방법 사용

다음의 코드를 보시죠.

```rust
    #[test]
    fn using_other_iterator_trait_methods() {
        let sum: u32 = Counter::new()
            .zip(Counter::new().skip(1))
            .map(|(a, b)| a * b)
            .filter(|x| x % 3 == 0)
            .sum();
        assert_eq!(18, sum);
    }
```

Rust는 훌륭한 반복자 관련 함수를 제공합니다. 이를 적극적으로 활용하면 코드의 크기도 줄고 관련된 코드를 직접 짤 떄 발생하는 다양한 실수를 없앨 수 있습니다.

## I/O 프로젝트 개선
이제 앞 장의 I/O 프로젝트를 본 장에서 배운 것으로 개선해 봅시다.

### 반복자를 이용해서 `clone()` 사용 제거
`clone()`은 값을 복사해서 메모리를 사용하게 되므로 성능 위주의 코드에서는 부적절합니다. 

```rust
impl Config {
    pub fn new(args: &[String]) -> Result<Config, &str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config {
            query,
            filename,
            case_sensitive,
        })
    }
}
```

이를 반복자를 이용해 개선해 봅시다.

#### 반환된 반복자를 직접 사용하기

파일이름: src/main.rs
```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // --snip--
}
```

이를 다음으로 변경,

```rust
    let config = Config::new(env::args()).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // --snip--
}
```

다음으로 변경,
파일 이름: src/lib.rs

```rust
impl Config {
    pub fn new(mut args: env::Args) -> Result<Config, &'static str> {
        // --snip--
```

#### 인덱싱 대신 Iterator 트레잇 방법 사용

이제 `Config` 구조체의 `new()` 메소드를 다음과 같이 변경 합니다.

```rust
impl Config {
    pub fn new(mut args: env::Args) -> Result<Config, &'static str> {
        args.next();

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };

        let filename = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a file name"),
        };

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config {
            query,
            filename,
            case_sensitive,
        })
    }
}
```

어떤가요? `clone()` 함수를 사용하지 않으므로 추가적인 메모리를 소비하지 않습니다.

### 반복자 어댑터로 코드를 더 명확하게 만들기
이제 `search()` 함수를 개선해 봅시다.

파일 이름: src/lib.rs
```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }

    results
}
```

을 다음처럼 변경할 수 있습니다.

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents
        .lines()
        .filter(|line| line.contains(query))
        .collect()
}
```

## 성능 비교: 루프 대 반복자
반복자를 사용하면 혹시 성능이 떨어지지 않을까요? 이러한 의구심은 많은 언어들이 반복자가 일반 배열의 인덱싱 방식 보다 느리기 때문에 그렇습니다. 하지만 Rust는 다르네요!

```
test bench_search_for  ... bench:  19,620,300 ns/iter (+/- 915,700)
test bench_search_iter ... bench:  19,234,900 ns/iter (+/- 657,200)
```

되려 반복자를 이용한 것이 좀 더 빠름을 알 수 있습니다!

반복자는 높은 수준의 추상화이지만, 하위 수준 코드를 직접 작성한 것과 거의 동일하게 Rust 컴파일러가 컴파일 합니다. 이는 Rust가 `제로 비용 추상화`를 지향하고 있기 때문입니다. Rust언어는 정말 성능에 목숨을 건 것 같네요.

다음의 예는 높은 수준의 추상화를 적용하더라도 결과적으로 성능에 미치는 영향이 없다는 것을 보여주는 코드로 실제 오디오 디코더에서 가져온 코드입니다.

```rust
let buffer: &mut [i32];
let coefficients: [i64; 12];
let qlp_shift: i16;

for i in 12..buffer.len() {
    let prediction = coefficients.iter()
                                 .zip(&buffer[i - 12..i])
                                 .map(|(&c, &s)| c * s as i64)
                                 .sum::<i64>() >> qlp_shift;
    let delta = buffer[i];
    buffer[i] = prediction as i32 + delta;
}
```

## 정리
Rust의 클로저는 함수형 프로그래밍 언어에서 영감을 받은 Rust의 기능입니다. 이는 높은 수준의 아이디어를 낮은 수준의 결과로 만드는 Rust의 훌륭한 기능 중의 하나 입니다.
반복자 역시 클로저와 결합하여 다양한 문제를 높은 추상화로 풀되 성능은 손해 보지 않는 Rust의 훌륭한 기능입니다.