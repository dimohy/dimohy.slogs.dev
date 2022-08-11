## Rust #14: 14장 Cargo 및 Crates.io에 대한 추가 정보

## 개요
이번 장에서는 다음에 대해 살펴보도록 합니다.

- 릴리즈 프로필을 통해 빌드 사용자 지정
- crates.io에 라이브러리 게시
- 작업 공간으로 대규모 프로젝트 구성
- crates.io에서 바이너리 설치
- 사용자 지정 명령을 이용해서 Cargo 확장

## 릴리즈 프로필로 빌드 사용자 지정

Cargo를 이용해 `dev`및 `release` 빌드를 다음처럼 진행할 수 있습니다.

```shell
$ cargo build
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
$ cargo build --release
    Finished release [optimized] target(s) in 0.0s
```

그리고 Cargo.toml에 프로필 별 별도 설정을 할 수 있습니다.

```toml
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```

기본 설정은 기본 프로필에 의 해 결정되며 위의 설정 값은 기본 프로필의 동일 설정 값을 변경하게 됩니다.

## Crates.io에 크레이트 게시하기

우리는 새로운 패키지를 `creates.io`를 통해 편리하게 설치하여 사용할 수 있습니다. 이와 마찬가지로 우리의 패키지 역시 `creates.io`에 게시하여 다른 사람들과 공유할 수도 있습니다.

### 유용한 문서 주석 작성 

세개의 슬러시('///')를 주면 Cargo에서 API 문서를 만들 때 문서화가 됩니다.

소스코드
```rust
/// 하나의 정수형 값을 입력받아 그것에 1을 더한 값을 반환한다.
///
/// # 예시
/// ```rust
/// let arg = 5;
/// let answer = my_Crate:add_one(arg);
///
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

이제 쉘에서,

```shell
$ cargo doc --open
```

하면 아래와 같은 문서를 확인할 수 있습니다.

문서
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1632904641865/K8V8AGaiu.png)

#### 일반적으로 사용되는 섹션
문서 주석은 마크다운으로 해석되어 HTML로 변환됩니다. 다음은 크레이트 작성자가 문서에 일반적으로 사용한는 몇가지 다른 섹션입니다.

- 패닉 : 기능이 패닉을 일으킬 수 있는 시나리오입니다.
- 오류 : 기능이 오류가 발생할 수 있는 시나리오입니다.
- 안정성 : 기능이 unsafe 호출의 경우

### 테스트로서의 문서 주석
테스트를 진행하면 문서 주석의 예제 코드 블록도 실행이 됩니다!

```shell
   Doc-tests my_crate

running 1 test
test src/lib.rs - add_one (line 5) ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.27s
```

### 포함된 항목에 주석 달기
`//!`를 주면 항목에 바로 문서를 추가할 수 있습니다.

```rust
//! # My Crate
//!
//! `my_crate` is a collection of utilities to make performing certain
//! calculations more convenient.

/// Adds one to the number given.
// --snip--
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1632904981692/mmKjquYkq.png)

### `pub use`를 사용하여 편리한 공개 API 내보내기
공개 API를 사용하려 할 때 각 세부 항목에 들어가서 내용을 살펴보는 것은 접근이 어렵습니다. `pub use`를 사용하면 문서화 시 항목이 표시됩니다.

```rust
//! # Art
//!
//! A library for modeling artistic concepts.

pub mod kinds {
    /// The primary colors according to the RYB color model.
    pub enum PrimaryColor {
        Red,
        Yellow,
        Blue,
    }

    /// The secondary colors according to the RYB color model.
    pub enum SecondaryColor {
        Orange,
        Green,
        Purple,
    }
}

pub mod utils {
    use crate::kinds::*;

    /// Combines two primary colors in equal amounts to create
    /// a secondary color.
    pub fn mix(c1: PrimaryColor, c2: PrimaryColor) -> SecondaryColor {
        // --snip--
    }
}
```


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1632905064862/0EhIqd4UjO.png)

```rust
use art::kinds::PrimaryColor;
use art::utils::mix;

fn main() {
    let red = PrimaryColor::Red;
    let yellow = PrimaryColor::Yellow;
    mix(red, yellow);
}```

위의 화면처럼 kinds의 요소가 확인이 되지 않습니다. 이것을,

```rust
//! # Art
//!
//! A library for modeling artistic concepts.

pub use self::kinds::PrimaryColor;
pub use self::kinds::SecondaryColor;
pub use self::utils::mix;

pub mod kinds {
    // --snip--
}

pub mod utils {
    // --snip--
}
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1632905157942/7-WxPl0ww.png)

```rust
use art::mix;
use art::PrimaryColor;

fn main() {
    // --snip--
}
```

로 사용할 수 있습니다.

### Crates.io 계정 설정
`Crates.io`에 본인의 패키지를 올리기 위해서는 계정 생성을 한 후 API 키를 받습니다. 그런 후 다음처럼 로그인을 할 수 있습니다.

``shell
$cargo login abcdefghijklmnopqrstuvwxyz012345
```
### 새 크레이트에 메타데이터 추가

자신의 패키지를 게시하기 위해 메타데이터가 필요합니다. Cargom.toml에 관련 메타정보를 설정할 수 있습니다.

```toml
[package]
name = "guessing_game"
```

그러나 게시는 실패합니다.

```shell
$ cargo publish
    Updating crates.io index
warning: manifest has no description, license, license-file, documentation, homepage or repository.
See https://doc.rust-lang.org/cargo/reference/manifest.html#package-metadata for more info.
--snip--
error: api errors (status 200 OK): missing or empty metadata fields: description, license. Please see https://doc.rust-lang.org/cargo/reference/manifest.html for how to upload metadat
```

`package`와 함께 필수 메타정보가 있기 때문입니다. 다음처럼 보완을 해봅시다.

```toml
[package]
name = "guessing_game"
version = "0.1.0"
edition = "2018"
description = "A fun game where you guess what number the computer has chosen."
license = "MIT OR Apache-2.0"

[dependencies]
```

### Crates.io 게시
```shell
$ cargo publish
    Updating crates.io index
   Packaging guessing_game v0.1.0 (file:///projects/guessing_game)
   Verifying guessing_game v0.1.0 (file:///projects/guessing_game)
   Compiling guessing_game v0.1.0
(file:///projects/guessing_game/target/package/guessing_game-0.1.0)
    Finished dev [unoptimized + debuginfo] target(s) in 0.19s
   Uploading guessing_game v0.1.0 (file:///projects/guessing_game)
```

했을 때 잘 게시가 됐다면 축하합니다! 하지만 아마도 처음엔 안되었을 텐데요, cargo에서 친절하게 원인을 표시해주니 그것 대로 해결해 나가면 됩니다. 특히 `crates.io`이미 사용하고 있는 패키지 명은 중복해서 쓸수가 없습니다!

### 기존 상자의 새 버전 게시
한번 게시가 완료되면 동일한 버젼으로 다시 게시가 안됩니다. 버전을 올리거나 기존 버전을 삭제해야 합니다.

### cargo yank로 Crates.io 버젼 제거
`cargo yank`로 기존 버전을 삭제할 수 있습니다. 하지만 기존 버전을 재 사용 할 수 있다는 의미는 아닙니다.

```shell
$ cargo yank --vers 1.0.1
```

만약 해당 버전을 다시 되돌릴려면 뒤에 `--undo` 명령어 인자를 추가 합니다.

```shell
$ cargo yank --vers 1.0.1 --undo
```

## Cargo 작업공간
Cargo를 이용해 프로젝트 상위 개념인 `작업공간`을 만들 수 있습니다. 다음을 따라 해봅시다.

먼저 `add`라는 디렉토리를 만듭시다. 그런 후 `add` 디렉토리 안에서 `adder` 실행 크레잇을 만든 후 차례대로 `add_one`과 `add_two` 라이브러리 크레잇을 만들어 봅시다.

`add` 디렉토리에 다음과 같이 `Cargo.toml`를 만듭니다.

```toml
[workspace]

members = [
    "adder",
    "add-one",
    "add-two",
]
```

그러면 디렉토리 구조는 다음과 같게 됩니다.
```
├── Cargo.toml
├── adder
│   ├── Cargo.toml
│   └── src
│       └── main.rs
├── add_one
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
├── add_two
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
```

`cargo build`를 하게 되면 `add`에 속해 있는 세개의 크레잇이 모두 빌드 됨을 확인할 수 있습니다.

```shell
$  cargo build
   Compiling add-one v0.1.0 (W:\rust-learning\add\add-one)
   Compiling adder v0.1.0 (W:\rust-learning\add\adder)
   Compiling add-two v0.1.0 (W:\rust-learning\add\add-two)
    Finished dev [unoptimized + debuginfo] target(s) in 0.55s
```

이제 `add-two`에 다음의 코드를 삽입 한 후,

```rust
pub fn add_two(x: i32) -> i32 {
    x + 1
}
```

Cargo는 작업공간에서 크레잇을 서로 의존한다고 가정하지 않으므로 직접 의존성을 명시해야 합니다.

adder/Cargo.toml
```toml
[dependencies]
add-two = { path = "../add-two" }
```

adder/src/main.rs
```rust
use add_two;

fn main() {
    let num = 10;
    println!(
        "Hello, world! {} plus one is {}!",
        num,
        add_two::add_two(num)
    );
}
```

이제 의존성에 의해 `adder`보다 `add-two`가 먼저 컴파일 됨을 확인할 수 있습니다.

```shell
$ cargo build
   Compiling add-two v0.1.0 (W:\rust-learning\add\add-two)
   Compiling add-one v0.1.0 (W:\rust-learning\add\add-one)
   Compiling adder v0.1.0 (W:\rust-learning\add\adder)
    Finished dev [unoptimized + debuginfo] target(s) in 0.61s
```

Cargo는 프로젝트가 여러 개일 경우 실행 크레잇을 찾아 실행을 해줍니다.

```shell
$ cargo run  
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target\debug\adder.exe`
Hello, world! 10 plus one is 11!
```

실행 크레잇이 여러개일 경우 프로젝트를 지정 할 수 있습니다.

```shell
cargo run -p adder
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target\debug\adder.exe`
Hello, world! 10 plus one is 11!
```

#### 작업 영역의 외부 패키지에 따라 다름

최상위 Cargo.lock에 종속성에 대한 정보가 포함되어 있지만 Cargo가 자동으로 외부 종속성을 프로젝트에 주입하지는 않습니다. 필요한 경우 개별 프로젝트의 Cargo.toml에 외부 패키지를 포함해야 합니다.

#### 작업 영역에 테스트 추가

테스트 역시 작업공간에 포함된 모든 프로젝트의 테스트를 수행합니다.

```shell
$ cargo test
   Compiling add-one v0.1.0 (file:///projects/add/add-one)
   Compiling adder v0.1.0 (file:///projects/add/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.27s
     Running target/debug/deps/add_one-f0253159197f7841

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running target/debug/deps/adder-49979ff40686fa8e

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests add-one

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

특정 프로젝트만 테스트를 수행하려면 다음과 같이 할 수 있습니다.

```shell
$ cargo test -p add_two
```

`crates.io`에 자신의 크레잇을 게시하려면 작업공간에 있는 모든 프로젝트를 개별 게시해야 합니다.

## cargo install을 이용해 Crates.io에서 바이너리 설치
이 명령을 사용하면 바이너리 크레잇을 로컬에 설치할 수 있습니다. 이것은 시스템 패키지 관리자를 대체하는 것은 아니고 개발 도구를 쉽게 설치하는 방법을 제공하는 것입니다.

```shell
cargo install ripgrep
    Updating crates.io index
  Downloaded ripgrep v11.0.2
  Downloaded 1 crate (243.3 KB) in 0.88s
  Installing ripgrep v11.0.2
--snip--
   Compiling ripgrep v11.0.2
    Finished release [optimized + debuginfo] target(s) in 3m 10s
  Installing ~/.cargo/bin/rg
   Installed package `ripgrep v11.0.2` (executable `rg`)
```

이후 `rg` 명령어를 통해 파일을 검색하기 위한 Rust 도구를 사용할 수 있게 됩니다.

## 사용자 지정 명령을로 Cargo 확장
실행 경로에 `cargo-somthing`이라는 이름의 실행 파일을 배치하면 `cargo somthing`으로 그 기능을 호출해 사용할 수 있습니다. 이 명령어는 `cargo --list`를 통해서도 표출이 되게 됩니다.

## 정리
릴리즈  빌드, Crates.io에 게시, Cargo 작업공간 설정, Crates.io를 통해 바이너리 크레잇 설치 방법, 사용자 지정 명령을 살펴봤습니다.
