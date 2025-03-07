---
title: "Rust #1: Rust 개발환경 구성"
datePublished: Wed Jul 14 2021 10:00:05 GMT+0000 (Coordinated Universal Time)
cuid: ckr3bb9tz004vovs1f1z5b3nl
slug: rust-creating-devenv
tags: rust

---

## 개요
저는 C# 언어를 주로 사용하고 있고 저급레벨을 다룰 수 있는 C++를 대체할 언어를 찾다가 Rust를 접하게 되었습니다. 사실, Go를 공부할까 Rust를 공부할까 시간 좀 걸렸는데요, 문법적인 익숙함은 Go가 더 친숙해 보였고 유용하게 사용하고 있는 오른소스 웹 서버인 [Caddy](https://caddyserver.com/)가 Go언어로 만들어졌다는데 있었습니다. Rust는 Webassembly에서 가장 활용되는 언어로 [소개](https://blog.scottlogic.com/2021/06/21/state-of-wasm.html?utm_source=thenewstack&utm_medium=website&utm_campaign=platform)되어 있다는데 있는데요, 고민끝에 Rust를 선택했습니다.

이 글은 이미 훌륭한 [Rust 개발환경 구성 컨텐츠](https://dev.to/cthutu/rust-1-creating-your-development-environment-55bi)가 있음에도 불구하고 직접 Rust를 설치하고 개발환경을 구성하면서 그 내용을 기록하기 위해 작성합니다. 좀 더 상세한 정보를 원하시는 분은 위의 링크로 접근하길 바랍니다.

## Rust 설치
[rustup](https://rustup.rs/)를 통해 본인의 운영체제에 맞게 쉽게 설치 가능합니다. 설치완료 후 설치된 Rust 버젼을 확인해 봅시다.

```shell
$ rustc --version
rustc 1.53.0 (53cb7b09b 2021-06-17)
```

rustup을 통해 설치된 구성요소는 다음과 같습니다.

- cargo - 강력한 Rust 패키지 관리 도구입니다.
- clippy - 코드를 스캔하고 코드를 개선하거나 문제를 확인하는 방법에 대한 제안을 받을 수 있는 도구입니다.
- rust-docs - 소스 코드에서 문서 자동 생성을 제공하는 플러그인 입니다.
- rustc - Rust 컴파일러입니다.
- rustfmt - 모든 사람의 Rust 코드가 동일하게 보이도록 하는 코드의 형식을 자동으로 지정하는 코드 포멧터입니다.

## 첫번째 Rust 프로그램
설치된 cargo를 통해 다음처럼 첫번째 Rust 프로그램을 생성할 수 있습니다.

```shell
$ cargo new hello
cd hello
```
hello 디렉토리의 파일목록은 다음과 같습니다.

```shell
d----        2021-07-14  오후 6:05                src
-a---        2021-07-14  오후 6:05              8 .gitignore
-a---        2021-07-14  오후 6:05            174 Cargo.toml
```

기본적으로 git 버전관리 시스템을 사용하는 것을 알 수 있고요, `Cargo.toml`을 통해 생성 패키지에서 사용하는 종속 패키지를 관리할 수 있음을 알 수 있습니다. `Cargo.toml`을 확인해보면,

```toml
[package]
name = "hello"
version = "0.1.0"
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```
`[package]`에서 패키지의 이름과 버젼, 에디션 등을 지정할 수 있고 `[dependencies]`에서 종속 패키지를 등록할 수 있음을 알 수 있습니다. `cargo new`를 통해 생성한 패키지는 종속성이 없으므로 비워져 있습니다. 다음으로 `src`디렉토리를 보면,

```shell
-a---        2021-07-14  오후 6:05             45 main.rs
```

`main.rs` RUST 소스 파일을 확인할 수 있는데요 그 내용을 보면,

```rust
fn main() {
    println!("Hello, world!");
}
```

메인함수로 이루어진 `Hello, World!`를 출력하는 간단한 소스코드임을 확인할 수 있습니다. `cargo`를 이용해 실행해 봅시다.

```shell
$ cargo run
Hello, world!
```

잘 실행되는군요!

## 통합 개발 환경 구성
다음으로 Rust를 효과적으로 사용하기 위한 [Visual Studio Code](https://code.visualstudio.com/)를 구성하도록 하겠습니다. [이곳](https://code.visualstudio.com/#alt-downloads)을 통해 자신에 맞는 설치본을 다운로드 받아 설치할 수 있습니다.
설치가 완료되면 `hello` 디렉토리에서

```shell
$ code .
```
로 `Visual Studio Code`를 실행할 수 있습니다.

Visual Studio Code에서 Rust 개발을 하려면 다음의 VS Code 확장을 설치해야 합니다.

## VS Code 확장

왼쪽 중앙에서 가장 아래의

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1626254354051/6gHfXuIyL.png)

메뉴를 통해 확장을 설치할 수 있는데요, 다음의 확장을 설치하도록 합니다.

- rust-analyzer - Rust의 `intellisense`를 제공합니다.
- CodeLLDB - 코드의 디버그를 위해 설치합니다.

또한 다음의 확장을 설치하면 유용합니다.

- crates - 참조하는 crate(Rust의 패키지)의 버젼을 확인할 수 있습니다.
- Markdown All in One - 마크다운 파일을 작성하기 위해 필요한 확장입니다.
- Even Better TOML - TOML을 지원하는 편집기 지원기능을 제공합니다.
- Rewrap - 라인에 더 잘 맞도록 텍스트(마크다운 파일 및  Rust의 주석)를 재포멧하는 기능을 제공합니다.

여기서 주의해야 할 것은 설치 패키지 중 `rust-analyzer`와 `rust` 둘 다 설치되었다면 충돌이 됩니다. 이 문서는 `rust-analyzer` 패키지를 설치했다는 것을 전제로 진행이 됩니다.

설치가 완료되면 Rust 소스코드가 다음처럼 보일 텐데요,

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1626255349331/QtDjo7rGB.png)

`Run`을 통해 Rust 소스코드를 컴파일 하여 실행할 수 있습니다. 또한 `Debug` 버튼을 통해 디버그를 수행할 수 있는데요, 관련된 설정은 다음에서 소개하도록 하겠습니다.

## VS 코드 디버깅
Visual Studio Code에서 디버깅 관련 설정을 하기 위해 `Ctrl+,`을 눌러 설정으로 이동한 후, `debug`로 검색합니다. 그리고 다음처럼 설정 합시다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1626255649704/m8ogPxNnh.png)

이제 Rust 소스코드에 `F9`키로 중단점을 지정할 수 있습니다.
다음으로 검색을 통해 다음을 찾고 아래와 같이 지정합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1626256068584/fYAXL27pw.png)

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1626256156192/LmAC8Oi5T.png)

이렇게 하면 앞서 언급한 `rustfmt` 구성요소에 의해 파일을 저장할 때마다 마법을 수행합니다. 저장을 통해 모든 코드구성이 정위치에 놓이게 됩니다.

이제 중단점을 지정해서 소스코드를 디버깅 할 수 있게 되었습니다.

---
이 글은 맷 데이비스 님의 [Rust #1: Creating your development environment](https://dev.to/cthutu/rust-1-creating-your-development-environment-55bi)을 절대적으로 참고했습니다. 좀 더 자세한 설명이 필요한 분은 이글을 참고하세요!