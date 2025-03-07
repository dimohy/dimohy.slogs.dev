---
title: "Rust #12: 12장 I/O 프로젝트: 명령줄 프로그램 구축"
datePublished: Tue Sep 14 2021 13:34:33 GMT+0000 (Coordinated Universal Time)
cuid: cktk49wax02suzds16l1a5bsl
slug: rust-12-12-io
tags: rust

---

## 개요
본 장을 통해 지금까지 배운 기술을 적용하고 몇 가지 표준 라이브러리 기능을 활용 할 것입니다. 진행 과정을 통해 점진적으로 모듈화가 되어가는 것을 같이 체험할 수 있으며, 테스트 주도 개발이 무엇인지도 맛보기로 체험할 수 있습니다.

## 명령 줄 인자 받아들이기
먼저 실행 크레이트를 생성합시다.

```shell
cargo new minigrep
     Created binary (application) `minigrep` project
$ cd minigrep
```

우리는 본 장을 통해 최종적으로 다음과 같이 특정 텍스트 파일에서 찾고자 하는 문자열이 있는 라인 항목을 출력하도록 만들 것입니다.

```shell
$ cargo run searchstring example-filename.txt
```

### 인수 값 읽기
먼저 메인 함수에서 명령 줄 인자를 받는 방법을 살펴봅시다.

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    println!("{:?}", args);
}
```

이제 다음처럼 입력된 명령 줄 인자가 잘 출력 됨을 확인할 수 있습니다.

```shell
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.61s
     Running `target/debug/minigrep`
["target/debug/minigrep"]
```

```shell
$ cargo run needle haystack
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 1.57s
     Running `target/debug/minigrep needle haystack`
["target/debug/minigrep", "needle", "haystack"]
```

### 변수에 인수 값 저장
이제 필요로 하는 인자를 변수에 담아 봅시다.

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();

    let query = &args[1];
    let filename = &args[2];

    println!("Searching for {}", query);
    println!("In file {}", filename);
}
```
```shell
$ cargo run test sample.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep test sample.txt`
Searching for test
In file sample.txt
```

## 파일 읽기
먼저 테스트에 필요한 텍스트 파일을 생성해봅시다.

파일명: poem.txt
```
I'm nobody! Who are you?
Are you nobody, too?
Then there's a pair of us - don't tell!
They'd banish us, you know.

How dreary to be somebody!
How public, like a frog
To tell your name the livelong day
To an admiring bog!
```

그리고 다음과 같이 해당 파일이 잘 읽혀 지는지 확인해봅시다.

```rust
use std::env;
use std::fs;

fn main() {
    // --snip--
    println!("In file {}", filename);

    let contents = fs::read_to_string(filename)
        .expect("Something went wrong reading the file");

    println!("With text:\n{}", contents);
}
```

```shell
$ cargo run the poem.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep the poem.txt`
Searching for the
In file poem.txt
With text:
I'm nobody! Who are you?
Are you nobody, too?
Then there's a pair of us - don't tell!
They'd banish us, you know.

How dreary to be somebody!
How public, like a frog
To tell your name the livelong day
To an admiring bog!
```

`fs::read_to_string()` 함수를 통해 파일을 문자열로 읽어올 수 있었습니다.

## 모듈화 및 오류 처리 개선을 위한 리팩토링
프로그램 코드는 유지/보수를 위해 모듈화 되어 있어야만 합니다. 모듈화 되어 있는 코드는 지역적 수정과 기능 확장이 유리한 구조입니다. 그리고 적절한 오류 처리가 되어야 하는데 오류 처리 역시 모듈화를 해야 합니다. 본 내용을 통해 어떻게 모듈화를 하는지 알아봅시다.

### 바이너리 프로젝트에 대한 관심사 분리
우리는 다음처럼 모듈화를 진행할 예정입니다.

- main.rs와 lib.rs로 구조를 분할하고 프로그램 논리를 lib.rs로 이동
- 명령 줄 구문 분석의 구조가 간단하면 main.rs에 남을 수 있음
- 명령 줄 구분 분석 로직이 복잡해지면 lib.rs로 이동해야 함

모듈화 후 main에 남아 있는 역할은 다음으로 제한되어야 합니다.

- 인자 값을 사용하여 명령 줄 구문 분석 논리 호출
- 기타 구성 설정
- lib.rs의 run 함수 호출
- run 함수 반환 오류를 처리

#### 인자 파서 추출
명령 줄 인자를 처리해 봅시다.

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let (query, filename) = parse_config(&args);

    // --snip--
}

fn parse_config(args: &[String]) -> (&str, &str) {
    let query = &args[1];
    let filename = &args[2];

    (query, filename)
}
```

#### 구성 값 그룹화

튜플은 여러 값을 반환할 수 있도록 돕지만 좀 더 명확해야 합니다. 다음의 코드를 봅시다.

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = parse_config(&args);

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    let contents = fs::read_to_string(config.filename)
        .expect("Something went wrong reading the file");

    // --snip--
}

struct Config {
    query: String,
    filename: String,
}

fn parse_config(args: &[String]) -> Config {
    let query = args[1].clone();
    let filename = args[2].clone();

    Config { query, filename }
}
```

#### Config에 대한 생성자 생성
이제 Config 구조체에서 명령줄 인자를 파싱하는게 좋습니다.

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args);

    // --snip--
}

// --snip--

impl Config {
    fn new(args: &[String]) -> Config {
        let query = args[1].clone();
        let filename = args[2].clone();

        Config { query, filename }
    }
}
```

### 오류 처리 수정
적절한 명령 줄 인자가 주어지지 않으면 현재는 배열 인덱스 범위가 벗어났다는 오류가 발생합니다.

```shell
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep`
thread 'main' panicked at 'index out of bounds: the len is 1 but the index is 1', src/main.rs:27:21
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

이제 이것을 개선해 봅시다.

#### 오류 메시지 개선
```rust
    // --snip--
    fn new(args: &[String]) -> Config {
        if args.len() < 3 {
            panic!("not enough arguments");
        }
        // --snip--
```

이제 범위가 벗어나면 우리의 메시지를 출력하고 프로그램이 종료되는데,

```shell
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep`
thread 'main' panicked at 'not enough arguments', src/main.rs:26:13
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

원인에 대한 메시지를 보완되었지만 아직도 사용자 친화적 이지는 않습니다. 

#### panic!을 호출하는 대신 새로운 Result 반환
`panic!`을 호출하는 대신 오류를 호출하는 곳에서 적절히 처리할 수 있도록 `Result<T, E>`로 반환하는게 좋습니다.

```rust
impl Config {
    fn new(args: &[String]) -> Result<Config, &str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        Ok(Config { query, filename })
    }
}
```

#### Config::new 호출과 오류 처리
이제 `Config::new()` 함수를 호출하는 `main`함수에서 결과 값을 처리할 수 있습니다.

```rust
use std::process;

fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        println!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // --snip--
```

이제 출력의 결과가 사용자 친화적이 되었습니다.

```shell
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.48s
     Running `target/debug/minigrep`
Problem parsing arguments: not enough arguments
```

### main에서 논리 추출
main함수에서 논리를 분리해 봅시다.

```rust
fn main() {
    // --snip--

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    run(config);
}

fn run(config: Config) {
    let contents = fs::read_to_string(config.filename)
        .expect("Something went wrong reading the file");

    println!("With text:\n{}", contents);
}

// --snip--
```

현재는 이 작업이 코드가 작아 불필요하게 느껴질 수 도 있겠지만, 코드의 양이 방대해진다면 논리를 분리하는 작업은 모듈화에서 반드시 필요한 작업입니다.

#### run 함수에서 오류 반환
run함수 반환값을 개선해봅시다.

```rust
use std::error::Error;

// --snip--

fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;

    println!("With text:\n{}", contents);

    Ok(())
}
```

#### main에서 run 반환 오류 처리
이제 main함수를 보완해야 합니다.

```rust
fn main() {
    // --snip--

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    if let Err(e) = run(config) {
        println!("Application error: {}", e);

        process::exit(1);
    }
}
```

### 코드를 라이브러리 크레이트로 분리
이제 main.rs와 논리 구조인 lib.rs로 구조를 분할하겠습니다.

파일 이름: src/lib.rs
```rust
use std::error::Error;
use std::fs;

pub struct Config {
    pub query: String,
    pub filename: String,
}

impl Config {
    pub fn new(args: &[String]) -> Result<Config, &str> {
        // --snip--
    }
}

pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    // --snip--
}
```

파일 이름: src/main.rs
```rust
use std::env;
use std::process;

use minigrep::Config;

fn main() {
    // --snip--
    if let Err(e) = minigrep::run(config) {
        // --snip--
    }
}
```

## 테스트 주도 개발로 라이브러리 기능 개발
이제 로직을 src/lib.rs로 이동했으며 그 이외의 명령 줄 인자 및 오류 처리를 src/main.rs에 두었으므로 핵심 코드의 기능 테스트가 수월해졌습니다. 본 내용을 통해 테스트 코드로 기능을 구현하고 확장하는 것을 살펴봅시다.

### 실패한 테스트 작성하기
코드를 구현하기 앞서서 먼저 실패하는 테스트 코드를 작성해봅시다.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn one_result() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.";

        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }
}
```

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    vec![]
}
```

이렇게 실패하는 테스트 코드를 작성하는 이유는 테스트가 통과되기 위한 목표 설정을 할 수 있기 때문인데요, 먼저 테스트 코드를 작성한 후 본 코드를 작성 하는 것이 테스트 주도 개발의 중요 핵심입니다.

```shell
$ cargo test
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished test [unoptimized + debuginfo] target(s) in 0.97s
     Running unittests (target/debug/deps/minigrep-9cd200e5fac0fc94)

running 1 test
test tests::one_result ... FAILED

failures:

---- tests::one_result stdout ----
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `["safe, fast, productive."]`,
 right: `[]`', src/lib.rs:44:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::one_result

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

테스트는 당연히 실패합니다.

### 테스트 통과를 위한 코드 작성
이제 테스트를 통과하기 위해 코드를 작성해 나가 봅시다.

search 함수를 구현하기 위한 논리는 다음과 같습니다.

- 콘텐츠의 각 줄을 반복
- 행에 쿼리 문자열이 포함되어 있는지 확인
- 있다면 반환 목록에 추가
- 없다면 아무것도 하지 않기
- 모든 줄을 반복한 후 결과 목록을 반환

#### lines 메소드로 라인 순회하기
순회 코드는 다음과 같습니다.

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    for line in contents.lines() {
        // do something with line
    }
}
```
여기서 중요하게 살펴볼 것은 `[&str].lines()` 함수입니다. 이 함수를 통해 라인 별로 순회할 수 있습니다.

#### 쿼리로 각 줄 탐색

탐색할 문자열을 각 라인 별로 비교해봅시다.

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    for line in contents.lines() {
        if line.contains(query) {
            // do something with line
        }
    }
}
```

#### 매칭된 라인 저장
문자열이 존재한다면 반환 목록에 추가 합니다.

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

그런 후 테스트를 진행해 봅시다.

```shell
$ cargo test
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished test [unoptimized + debuginfo] target(s) in 1.22s
     Running unittests (target/debug/deps/minigrep-9cd200e5fac0fc94)

running 1 test
test tests::one_result ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests (target/debug/deps/minigrep-9cd200e5fac0fc94)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests minigrep

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

테스트가 통과됐습니다!

#### run 함수에서 search 함수 사용
이제 run함수에 search 함수를 적용합시다.

```rust
pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;

    for line in search(&config.query, &contents) {
        println!("{}", line);
    }

    Ok(())
}
```

## 환경 변수 동작
조금 더 다양한 동작을 실험하기 위해 환경 변수에 따라 다르게 동작하는 코드를 작성해 보도록 합시다.

### 대소문자를 구분하지 않는 search 함수에 대한 실패 테스트 작성
먼저 대소문자를 구분하지 않는 기능에 대한 실패 테스트를 작성해봅시다.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn case_sensitive() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Duct tape.";

        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }

    #[test]
    fn case_insensitive() {
        let query = "rUsT";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Trust me.";

        assert_eq!(
            vec!["Rust:", "Trust me."],
            search_case_insensitive(query, contents)
        );
    }
}
```

### search_case_insensitive 기능 구현
`search_case_insensitive()` 함수를 구현해 봅시다.

```rust
pub fn search_case_insensitive<'a>(
    query: &str,
    contents: &'a str,
) -> Vec<&'a str> {
    let query = query.to_lowercase();
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.to_lowercase().contains(&query) {
            results.push(line);
        }
    }

    results
}
```

테스트를 돌리면,

```shell
$ cargo test
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished test [unoptimized + debuginfo] target(s) in 1.33s
     Running unittests (target/debug/deps/minigrep-9cd200e5fac0fc94)

running 2 tests
test tests::case_insensitive ... ok
test tests::case_sensitive ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests (target/debug/deps/minigrep-9cd200e5fac0fc94)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests minigrep

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

정상적으로 기능 테스트가 되었습니다!

이제 Config 구조체에 관련 필드를 추가해봅시다.

```rust
pub struct Config {
    pub query: String,
    pub filename: String,
    pub case_sensitive: bool,
}
```

그런 후 run함수를 변경해야겠죠.

```rust
pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;

    let results = if config.case_sensitive {
        search(&config.query, &contents)
    } else {
        search_case_insensitive(&config.query, &contents)
    };

    for line in results {
        println!("{}", line);
    }

    Ok(())
}
```

이제 Config의 설정에 따라 대소문자를 무시할 지를 결정할 수 있게 되었습니다.

이제 환경 변수를 이용해 대소문자 무시 유무를 걸정 하도록 코드를 보완합니다.

```rust
use std::env;
// --snip--

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

이제 확인해봅시다!

```shell
$ cargo run to poem.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep to poem.txt`
Are you nobody, too?
How dreary to be somebody!
```

다음 환경 변수를 이용해 봅시다.

```shell
$ CASE_INSENSITIVE=1 cargo run to poem.txt
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep to poem.txt`
Are you nobody, too?
How dreary to be somebody!
To tell your name the livelong day
To an admiring bog!
```

## 표준 출력 대신 표준 오류에 오류 메시지 쓰기
일반 출력은 일반 내용만 출력 하는 것이 좋습니다. 이제 오류 출력을 표준 오류로 변경해 봅시다.

### 오류가 쓰여진 위치 확인
(깨끗한) 출력 결과를 파일로 저장하기 위해 다음처럼 사용합니다.

```shell
$ cargo run > output.txt
```

그런데 현재는 오류 출력도 포함되어 있으므로 표준 오류로 변경해봅시다.

### 오류를 표준 오류로 인쇄

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    if let Err(e) = minigrep::run(config) {
        eprintln!("Application error: {}", e);

        process::exit(1);
    }
}
```

다음처럼 명령 줄 인자를 주지 않으면 표준 파이프라인이 아닌 출력으로 오류가 표시됨을 확인 할 수 있습니다. 

```shell
$ cargo run > output.txt
Problem parsing arguments: not enough arguments
```

명령 줄 인자를 잘 주면,

```shell
$ cargo run to poem.txt > output.txt
```

원하는 결과 값만 파일로 잘 저장이 됩니다.

파일 이름: output.txt
```
Are you nobody, too?
How dreary to be somebody!
```

## 정리
이 장으로 통해 비로서 Rust 코드를 이용해 구조적인 모듈을 만들 수 있었습니다. 본 장을 여러 번 반복한다면 Rust 코드를 이용해 구조적으로 코딩할 수 있는 기본 실력이 갖춰지리라 생각합니다.
