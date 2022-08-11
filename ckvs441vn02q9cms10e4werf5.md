## Rust #20: 20장 최종 프로젝트: 멀티스레드 웹 서버 구축

## 개요

본 장은 Rust로 웹 서버를 구축하는 것으로 지금까지 익혔던 코드를 정리하도록 하겠습니다.

웹 서버 구축 계획은 다음과 같습니다.

1. TCP와 HTTP에 대해 조금 배웁니다.
2. 소켓에서 TCP 연결을 수신합니다.
3. 적은 수의 HTTP 요청을 구문 분석합니다.
4. 적절한 HTTP 응답을 만듭니다.
5. 스레드 풀로 서버의 처리량을 향상 시킵니다.

##  단일 스레드 웹 서버 구축

웹 서버는 TCP 기반의 HTTP  프로토콜을 통해 가능합니다.

### TCP 연결 리스닝

먼저 프로젝트를 생성합시다.

```shell
$ cargo new hello
     Created binary (application) `hello` project
$ cd hello
```

이제 `src/main.rs`에 다음의 코드를 작성해 봅시다.

| src/main.rs
```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        println!("Connection established!");
    }
}
```

실행하면 이제 `127.0.0.1:7878`을 통해 TCP 연결을 수신할 수 있게 됩니다. 

| 연결 되었을 때
```shell
$ cargo run
warning: unused variable: `stream`
 --> src\main.rs:7:13
  |
7 |         let stream = stream.unwrap();
  |             ^^^^^^ help: if this is intentional, prefix it with an underscore: `_stream`
  |
  = note: `#[warn(unused_variables)]` on by default

warning: `chapter20` (bin "chapter20") generated 1 warning
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target\debug\chapter20.exe`
Connection established!
```

그런데 바로 연결이 종료됨을 알 수 있습니다. 이유는 for문에서 연결에 의해 생성된 `stream`은 바로 소멸 되기 때문입니다.

### 요청 읽기

이제 연결된 후 요청 온 내용을 그대로 출력 해보도록 합시다.

```rust
use std::io::prelude::*;
use std::net::TcpListener;
use std::net::TcpStream;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        handle_connection(stream);
    }
}

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 1024];

    stream.read(&mut buffer).unwrap();

    println!("Request: {}", String::from_utf8_lossy(&buffer[..]));
}
```

```shell
$ cargo run
   Compiling hello v0.1.0 (file:///projects/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.42s
     Running `target/debug/hello`
Request: GET / HTTP/1.1
Host: 127.0.0.1:7878
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64; rv:52.0) Gecko/20100101
Firefox/52.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
Upgrade-Insecure-Requests: 1
������������������������������������
```

웹 브라우저에 따라 출력 되는 HTTP 헤더 정보는 다를 수 있지만 어쨌든 헤더가 그대로 출력 됩니다!

### HTTP 요청 자세히 살펴보기

HTTP는 텍스트 기반 프로토콜이며 다음의 형식을 가집니다.

```
메소드 요청URI HTTP버젼 CRLF
헤더정보 CRLF
메시지 본문
```

첫 번째 줄은 클라이언트가 요청하는 내용입니다. 메소드는 `GET, POST, PUT, DELETE` 등 HTTP에서 사용할 수 있는 메소드로 요청URI에 접근한다는 의미입니다. HTTP버젼은 `HTTP/1.1`등으로 표시하며 사용할 HTTP  버젼을 표시합니다.

두 번째 줄은 헤더 정보로 쿠키 정보 등 서버가 알아야 할 메타 정보를 전송합니다.

세 번째 줄은 메시지 본문으로 메시지 본문이 없을 경우 생략합니다. 메시지 본문은 값을 전달하고자 할 때 쓰입니다.

### 응답 작성

응답은 다음처럼 합니다.

```
HTTP버젼 상태코드 텍스트설명 CRLF
헤더정보 CRLF
메시지 본문
```

정상적인 반환으로 다음의 값을 반환하도록 합시다.

```
HTTP/1.1 200 OK\r\n\r\n
```

```rust
fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 1024];

    stream.read(&mut buffer).unwrap();

    let response = "HTTP/1.1 200 OK\r\n\r\n";

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

버퍼에 요청 정보를 담은 후 `response` 문자열을 바이트로 변환해서 스트림에 쓴 후 플러시 합니다. 플러시를 하지 않을 경우 스트림이 닫히기 전까지 쓰기가 지연될 수 있습니다.

이렇게 해서 웹 브라우저로 요청을 다시 하면 오류 대신 빈 페이지가 표시되어야 합니다.

### 실제 HTML 반환

다음처럼 `hello.html`을 만들고

| hello.html
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Hello!</title>
  </head>
  <body>
    <h1>Hello!</h1>
    <p>Hi from Rust</p>
  </body>
</html>
```

소스 코드에서 이 파일을 읽어 반환하도록 합시다.

```rust
use std::fs;
// --snip--

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 1024];
    stream.read(&mut buffer).unwrap();

    let contents = fs::read_to_string("hello.html").unwrap();

    let response = format!(
        "HTTP/1.1 200 OK\r\nContent-Length: {}\r\n\r\n{}",
        contents.len(),
        contents
    );

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

이제 `hello.html`을 읽어서 메시지 본문에 추가를 해줍니다. 이때 헤더 정보의 `Content-Length`에 `hello.html`의 길이를 넣어줘서 브라우저에서 올바르게 본문을 읽을 수 있도록 합니다.

이제 웹 브라우저에서 HTML이 렌더링 되어 잘 표시됨을 확인할 수 있습니다!

### 요청 검증 및 선택적 응답

지금까지는 웹 브라우저가 무엇을 요청했는지 상관없이 동일한 결과를 반환했습니다. 이것을 절대 경로의 `GET` 메소드 요청만 처리하는 것으로 코드를 수정해봅시다.

```rust
// --snip--

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 1024];
    stream.read(&mut buffer).unwrap();

    let get = b"GET / HTTP/1.1\r\n";

    if buffer.starts_with(get) {
        let contents = fs::read_to_string("hello.html").unwrap();

        let response = format!(
            "HTTP/1.1 200 OK\r\nContent-Length: {}\r\n\r\n{}",
            contents.len(),
            contents
        );

        stream.write(response.as_bytes()).unwrap();
        stream.flush().unwrap();
    } else {
        // some other request
    }
}
```

특이한 점은 버퍼에 특정 값이 있는 지를 확인하는 과정에서 "..."아닌 b"..."를 사용한다는 점입니다. 바이트 배열에 직접 비교할 수 있도록 하는 Rust 기능입니다.

나머지 `else` 코드도 보시죠.

```rust
    // --snip--
    } else {
        let status_line = "HTTP/1.1 404 NOT FOUND";
        let contents = fs::read_to_string("404.html").unwrap();

        let response = format!(
            "{}\r\nContent-Length: {}\r\n\r\n{}",
            status_line,
            contents.len(),
            contents
        );

        stream.write(response.as_bytes()).unwrap();
        stream.flush().unwrap();
    }
```

절대 경로의 `GET` 요청이 아닐 경우 404페이지를 표시합니다.

| 404.html
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Hello!</title>
  </head>
  <body>
    <h1>Oops!</h1>
    <p>Sorry, I don't know what you're asking for.</p>
  </body>
</html>
```

### 약간의 리팩토링

현재는 코드가 많이 중복됩니다. 이것을 정리해보도록 합시다.

```rust
// --snip--

fn handle_connection(mut stream: TcpStream) {
    // --snip--

    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();

    let response = format!(
        "{}\r\nContent-Length: {}\r\n\r\n{}",
        status_line,
        contents.len(),
        contents
    );

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

정상적인 요청과 비정상적 요청의 다른 부분은 튜플로 묶었습니다.

## 단일 스레드 서버를 다중 스레드 서버로 전환

현재까지는 단일 스레드에서 해당 요청을 모두 처리한 후 종료하는 식입니다. 요청이 적을 경우 이 처리는 문제가 없지만 동시 접속자가 많을 경우 느려지게 됩니다. 이를 다중 스레드 서버 전환해서 해결해봅시다.

### 현재 서버 구현에서 느린 요청을 시뮬레이션

다음의 코드를 통해 느린 상황을 시뮬레이션 해봅시다.

```rust
use std::thread;
use std::time::Duration;
// --snip--

fn handle_connection(mut stream: TcpStream) {
    // --snip--

    let get = b"GET / HTTP/1.1\r\n";
    let sleep = b"GET /sleep HTTP/1.1\r\n";

    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK", "hello.html")
    } else if buffer.starts_with(sleep) {
        thread::sleep(Duration::from_secs(5));
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };

    // --snip--
}
```

`GET /sleep`의 접근은 5초의 대기 시간 후 결과를 반환하도록 했습니다. 이제 웹 브라우저에서 `/sleep`으로 접근하는 동안 다른 창에서 `/`으로 접근해보세요. `/sleep`의 처리가 끝날 때까지 블럭됨을 확인할 수 있습니다.

### 스레드 풀로 처리량 향상

스레드 풀을 이용하면 동시 처리가 가능해집니다. 스레드 풀은 기본적으로 서비스 거부(DoS) 공격으로 보호하기 위해 풀의 스레드 수를 적은 수로 제한합니다.

스레드 풀은 요청 대기 스레드를 가지고 있다가 요청이 왔을 때 사용할 수 있는 스레드를 쓸 수 있도록 반환합니다.

#### 각 요청에 대해 스레드를 생성할 수 있는 경우 코드 구조

스레드 풀을 사용하기 앞서서 연결이 될 때마다 스레드를 생성해서 처리하는 코드를 봅시다.

```rust
fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        thread::spawn(|| {
            handle_connection(stream);
        });
    }
}
```

이 코드는 잘 동작할 것이지만 앞 전 이야기 한 것 처럼 서비스 공격(DoS)에 취약합니다.

#### 유한한 스레드 수에 대한 유사한 인터페이스 만들기

다음의 코드를 보시죠.

```rust
fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }
}
```

아직 `ThreadPool`이 없으므로 컴파일 오류가 발생할 것이지만 이 코드가 최종 동작할 코드가 됩니다. `ThreadPool::new(4)`를 통해 통 4개의 스레드를 할당했고, `thread::spawn()`과 유사한 사용 방식으로 `pool.execute()`를 통해 할당된 스레드에 클로저를 통해 코드가 실행되도록 했습니다.

#### 컴파일러 주도 개발을 이용한 ThreadPool 구조체 만들기

위의 코드는 `ThreadPool`이 없으므로 컴파일 오류가 발생합니다.

```shell
$ cargo check
    Checking hello v0.1.0 (file:///projects/hello)
error[E0433]: failed to resolve: use of undeclared type `ThreadPool`
  --> src/main.rs:10:16
   |
10 |     let pool = ThreadPool::new(4);
   |                ^^^^^^^^^^ use of undeclared type `ThreadPool`

For more information about this error, try `rustc --explain E0433`.
error: could not compile `hello` due to previous error
```

| src/lib.rs
```rust
pub struct ThreadPool;
```

그런 다음 `src/main.rs`를 `src/bin/main.rs`로 옮깁니다. 이렇게 하면 라이브러리 크레이트가 hello 디렉토리의 기본 크레이트가 됩니다. 우리는 여전히 `src/bin/main.rs`를 `cargo run`을 통해 실행할 수 있습니다.

| src/bin/main.rs
```rust
use hello::ThreadPool;
```

여전히 작동하지 않지만 오류 메시지가 달라졌습니다.

```shell
$ cargo check
    Checking hello v0.1.0 (file:///projects/hello)
error[E0599]: no function or associated item named `new` found for struct `ThreadPool` in the current scope
  --> src/bin/main.rs:11:28
   |
11 |     let pool = ThreadPool::new(4);
   |                            ^^^ function or associated item not found in `ThreadPool`

For more information about this error, try `rustc --explain E0599`.
error: could not compile `hello` due to previous error
```

다시 코드를 수정해 봅시다.

| src/lib.rs
```rust
pub struct ThreadPool;

impl ThreadPool {
    pub fn new(size: usize) -> ThreadPool {
        ThreadPool
    }
}
```

다시 `cargo check`를 통해 오류를 확인해봅시다.

```shell
$ cargo check
    Checking hello v0.1.0 (file:///projects/hello)
error[E0599]: no method named `execute` found for struct `ThreadPool` in the current scope
  --> src/bin/main.rs:16:14
   |
16 |         pool.execute(|| {
   |              ^^^^^^^ method not found in `ThreadPool`

For more information about this error, try `rustc --explain E0599`.
error: could not compile `hello` due to previous error
```

다음 구현해야 할 메소드 관련 오류가 발생하는군요.

다음의 `spawn()` 구조를 살펴보고 유사하게 만들어 봅시다.

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
    where
        F: FnOnce() -> T,
        F: Send + 'static,
        T: Send + 'static,
```

| src/lib.rs
```rust
impl ThreadPool {
    // --snip--
    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
    }
}
```

```shell
$ cargo check
    Checking hello v0.1.0 (file:///projects/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.24s
```

이제 소스 코드가 컴파일 됩니다! (아직 아무런 동작을 하지는 않지만) 

#### new의 스레드 수 확인

이제 세부 구현을 해봅시다. `new()`의 인자를 통해 스레드풀을 구성하는 코드를 전개하기 앞서서 0개의 스레드는 의미가 없으므로 다음처럼 코드를 전개해봅시다.

| src/lib.rs
```rust
impl ThreadPool {
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        ThreadPool
    }

    // --snip--
}
```

또한 ThreadPool에 문서 주석을 추가했는데, `cargo doc --open`을 통해 확인할 수 있습니다.

`assert!()`를 사용하는 대신 에러를 반환할 수도 있습니다.

```rust
pub fn new(size: usize) -> Result<ThreadPool, PoolCreationError> {
```

#### 스레드를 저장할 공간 만들기

다시 `spawn()`의 서명을 살펴봅시다.

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
    where
        F: FnOnce() -> T,
        F: Send + 'static,
        T: Send + 'static,
```

`spawn()`에서 반환하는 `JoinHandle<T>`의 경우 T는 클로저를 반환하는 유형입니다. 이제 적용해봅시다.

| src/lib.rs
```rust
use std::thread;

pub struct ThreadPool {
    threads: Vec<thread::JoinHandle<()>>,
}

impl ThreadPool {
    // --snip--
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let mut threads = Vec::with_capacity(size);

        for _ in 0..size {
            // create some threads and store them in the vector
        }

        ThreadPool { threads }
    }

    // --snip--
}
```

`cargo check`를 했을 때 몇 가지 경고가 뜰 수 는 있겠지만 오류가 발생하지 않아야 합니다.

#### ThreadPool에서 Thread로 코드 전송을 담당하는 Worker 구조체

ThreadPool에서 생성한 Thread는 특정 코드만 실행하는 것이 아니라 필요에 따라 실행하는 코드가 달라져야 합니다. 이를 달성하기 위해서는 Thread에 실행 코드를 전송하는 Worker라는 메커니즘이 필요한데 이를 구현하도록 합시다.

| src/lib.rs
```rust
use std::thread;

pub struct ThreadPool {
    workers: Vec<Worker>,
}

impl ThreadPool {
    // --snip--
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id));
        }

        ThreadPool { workers }
    }
    // --snip--
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize) -> Worker {
        let thread = thread::spawn(|| {});

        Worker { id, thread }
    }
}
```

이제 Worker가 스레드를 가지고 있고, ThreadPool은 Worker를 스레드 개수만큼 가지게 되었습니다. (아직 Worker는 동작하지 않습니다.)

#### 채널을 통해 스레드에 요청 보내기

우리는 채널을 배웠습니다. 이 채널을 이용해 Worker 스레드가 실행할 코드를 수신 받아 실행하도록 만들어 봅시다.

| src/lib.rs
```rust
// --snip--
use std::sync::mpsc;

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

struct Job;

impl ThreadPool {
    // --snip--
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id));
        }

        ThreadPool { workers, sender }
    }
    // --snip--
}
```

`ThreadPool`의 `execute()`에 의해 Worker 스레드로 실행할 코드 블럭을 전송할 것이므로 `ThreadPool`이 `Sender<Job>`을 갖게 합니다. (아직 Job은 아무것도 없습니다.)

이제 `Worker`가 `receiver`를 가지도록 코드를 좀 더 수정해봅시다.

| src/lib.rs
```rust
impl ThreadPool {
    // --snip--
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, receiver));
        }

        ThreadPool { workers, sender }
    }
    // --snip--
}

// --snip--

impl Worker {
    fn new(id: usize, receiver: mpsc::Receiver<Job>) -> Worker {
        let thread = thread::spawn(|| {
            receiver;
        });

        Worker { id, thread }
    }
}
```

이제 `ThreadPool`이 채널을 이용해 메시지를 보낼 수 있게 되었고 `Worker`가 메시지를 수신할 수 있게 되었습니다.

하지만 다음처럼 오류가 발생합니다.

```shell
$ cargo check
    Checking hello v0.1.0 (file:///projects/hello)
error[E0382]: use of moved value: `receiver`
  --> src/lib.rs:27:42
   |
22 |         let (sender, receiver) = mpsc::channel();
   |                      -------- move occurs because `receiver` has type `std::sync::mpsc::Receiver<Job>`, which does not implement the `Copy` trait
...
27 |             workers.push(Worker::new(id, receiver));
   |                                          ^^^^^^^^ value moved here, in previous iteration of loop

For more information about this error, try `rustc --explain E0382`.
error: could not compile `hello` due to previous error
```

이유는 다수가 하나의 `receiver`를 소유하려 했기 때문인데요, 이를 피하기 위해 전에 배웠던 `Arc`를 이용해 다수가 소유권을 공유하도록 변경해봅시다.

```rust
use std::sync::Arc;
use std::sync::Mutex;
// --snip--

impl ThreadPool {
    // --snip--
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    // --snip--
}

// --snip--

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        // --snip--
    }
}
```

이제 다시 코드가 컴파일 됩니다!

#### execute 메소드 구현

이제 `execute()` 메소드를 구현해 봅시다.

| src/lib.rs
```rust
// --snip--

type Job = Box<dyn FnOnce() + Send + 'static>;

impl ThreadPool {
    // --snip--

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.send(job).unwrap();
    }
}

// --snip--
```

오! 클로저를 박싱해서 채널을 이용해 전송을 하게 됩니다! 이제 이것을 수신하기 위한 Worker를 구현하도록 합시다.

```rust
// --snip--

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let job = receiver.lock().unwrap().recv().unwrap();

            println!("Worker {} got a job; executing.", id);

            job();
        });

        Worker { id, thread }
    }
}
```

이제 스레드는 채널에서 실행할 코드가 오기 전까지 블럭되어 있다가 메시지를 수신받으면 깨어나 그 코드를 실행합니다!

```shell
$ cargo run
   Compiling hello v0.1.0 (file:///projects/hello)
warning: field is never read: `workers`
 --> src/lib.rs:7:5
  |
7 |     workers: Vec<Worker>,
  |     ^^^^^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(dead_code)]` on by default

warning: field is never read: `id`
  --> src/lib.rs:48:5
   |
48 |     id: usize,
   |     ^^^^^^^^^

warning: field is never read: `thread`
  --> src/lib.rs:49:5
   |
49 |     thread: thread::JoinHandle<()>,
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

warning: 3 warnings emitted

    Finished dev [unoptimized + debuginfo] target(s) in 1.40s
     Running `target/debug/main`
Worker 0 got a job; executing.
Worker 2 got a job; executing.
Worker 1 got a job; executing.
Worker 3 got a job; executing.
Worker 0 got a job; executing.
Worker 2 got a job; executing.
Worker 1 got a job; executing.
Worker 3 got a job; executing.
Worker 0 got a job; executing.
Worker 2 got a job; executing.
```

이제 웹 브라우저에서 `/sleep`을 호출해도 다른 요청도 즉시 처리되는 것을 확인할 수 있습니다.

그런데 `while let`을 이용하지 않은 것이 이상할 수 도 있습니다.

| src/lib.rs
```rust
// --snip--

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || {
            while let Ok(job) = receiver.lock().unwrap().recv() {
                println!("Worker {} got a job; executing.", id);

                job();
            }
        });

        Worker { id, thread }
    }
}
```

이것은 느린 요청에 의해 의도대로 동작하지 않습니다!

## 정상적인 종료 및 정리

지금까지 전개한 코드는 의도한 대로 스레드풀을 이용해서 비동기적으로 요청에 응답하였습니다. 이제 몇가지 경고와 예외 처리를 좀 더 진행해보도록 합시다.

### ThreadPool의 Drop 트레잇 구현

스레드풀이 삭제되면 모든 Worker의 스레드가 중지하고 정리되어야 합니다.

| src/lib.rs
```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            worker.thread.join().unwrap();
        }
    }
}
```

```shell
$ cargo check
    Checking hello v0.1.0 (file:///projects/hello)
error[E0507]: cannot move out of `worker.thread` which is behind a mutable reference
  --> src/lib.rs:52:13
   |
52 |             worker.thread.join().unwrap();
   |             ^^^^^^^^^^^^^ move occurs because `worker.thread` has type `JoinHandle<()>`, which does not implement the `Copy` trait

For more information about this error, try `rustc --explain E0507`.
error: could not compile `hello` due to previous error
```

이런! 우리는 소유권 때문에 `worker.thread`를 통해 `join()`을 호출할 수 없습니다. `join()`을 호출하기 위해 `thread`의 소유권을 가져와야 하는데 `Worker`의 `thread`가 `Option`이면 됩니다.

이를 해결하기 위해 `Worker`를 다음처럼 수정해야 합니다.

| src/lib.rs
```rust
struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}
```

다시 컴파일러를 통해 문제를 확인해봅시다.

```shell
$ cargo check
    Checking hello v0.1.0 (file:///projects/hello)
error[E0599]: no method named `join` found for enum `Option` in the current scope
  --> src/lib.rs:52:27
   |
52 |             worker.thread.join().unwrap();
   |                           ^^^^ method not found in `Option<JoinHandle<()>>`

error[E0308]: mismatched types
  --> src/lib.rs:72:22
   |
72 |         Worker { id, thread }
   |                      ^^^^^^
   |                      |
   |                      expected enum `Option`, found struct `JoinHandle`
   |                      help: try using a variant of the expected enum: `Some(thread)`
   |
   = note: expected enum `Option<JoinHandle<()>>`
            found struct `JoinHandle<_>`

Some errors have detailed explanations: E0308, E0599.
For more information about an error, try `rustc --explain E0308`.
error: could not compile `hello` due to 2 previous errors
```

먼저 두 번째 오류부터 해결해봅시다.

| src/lib.rs
```rust
impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        // --snip--

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

그런 후 `drop()`의 오류를 해결해봅시다.

| src/lib.rs
```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}
```

`take()`를 통해 `thread`를 가져오고 그 자리에 `None`을 둡니다. 이제 가져온 thread를 통해 `join()`메소드를 호출할 수 있게 되었습니다.

### 작업 수신을 중지하도록 스레드에 신호 보내기

하지만 아직 `join()`에 반응하지 않습니다. 이유는 Worker 스레드가 메시지를 수신 받기 위해 무한정 대기하기 때문입니다. 이를 해결하기 위해 메시지를 두 가지 타입으로 만들어봅시다.

| lib.rs
```rust
enum Message {
    NewJob(Job),
    Terminate,
}
```

```rust
pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}

// --snip--

impl ThreadPool {
    // --snip--

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.send(Message::NewJob(job)).unwrap();
    }
}

// --snip--

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let message = receiver.lock().unwrap().recv().unwrap();

            match message {
                Message::NewJob(job) => {
                    println!("Worker {} got a job; executing.", id);

                    job();
                }
                Message::Terminate => {
                    println!("Worker {} was told to terminate.", id);

                    break;
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

이제 메시지를 수신했을 때 `Terminate`일 경우 Worker 스레드를 종료할 수 있게 되었습니다.

이제 이에 맞게 `drop()` 메소드도 수정합시다.

| src/lib.rs
```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        println!("Sending terminate message to all workers.");

        for _ in &self.workers {
            self.sender.send(Message::Terminate).unwrap();
        }

        println!("Shutting down all workers.");

        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}
```

이제 모든 Worker 스레드에 `Terminate` 메시지를 보내 스레드를 종료하게 한 다음 `join()`을 통해 종료됐음을 다시 한번 확인합니다.

두 개의 개별 루프가 필요한 이유를 살펴봅시다. 만약에 루프가 하나라면 지금 종료되어야 할 스레드가 `Terminate`를 수신했다는 보장이 없습니다. 다음의 코드를 통해 좀 더 살펴보도록 합시다.

```rust
fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming().take(2) {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }

    println!("Shutting down.");
}
```

3개의 요청을 했을 때 다음과 유사한 실행 결과를 확인할 수 있습니다.

```shell
$ cargo run
   Compiling hello v0.1.0 (file:///projects/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 1.0s
     Running `target/debug/main`
Worker 0 got a job; executing.
Worker 3 got a job; executing.
Shutting down.
Sending terminate message to all workers.
Shutting down all workers.
Shutting down worker 0
Worker 1 was told to terminate.
Worker 2 was told to terminate.
Worker 0 was told to terminate.
Worker 3 was told to terminate.
Shutting down worker 1
Shutting down worker 2
Shutting down worker 3
```

참고용 전체 코드는 다음과 같습니다.

| src/bin/main.rs
```rust
use hello::ThreadPool;
use std::fs;
use std::io::prelude::*;
use std::net::TcpListener;
use std::net::TcpStream;
use std::thread;
use std::time::Duration;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }

    println!("Shutting down.");
}

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 1024];
    stream.read(&mut buffer).unwrap();

    let get = b"GET / HTTP/1.1\r\n";
    let sleep = b"GET /sleep HTTP/1.1\r\n";

    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK", "hello.html")
    } else if buffer.starts_with(sleep) {
        thread::sleep(Duration::from_secs(5));
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();

    let response = format!(
        "{}\r\nContent-Length: {}\r\n\r\n{}",
        status_line,
        contents.len(),
        contents
    );

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

| src/lib.rs
```rust
use std::sync::mpsc;
use std::sync::Arc;
use std::sync::Mutex;
use std::thread;

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;

enum Message {
    NewJob(Job),
    Terminate,
}

impl ThreadPool {
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.send(Message::NewJob(job)).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        println!("Sending terminate message to all workers.");

        for _ in &self.workers {
            self.sender.send(Message::Terminate).unwrap();
        }

        println!("Shutting down all workers.");

        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let message = receiver.lock().unwrap().recv().unwrap();

            match message {
                Message::NewJob(job) => {
                    println!("Worker {} got a job; executing.", id);

                    job();
                }
                Message::Terminate => {
                    println!("Worker {} was told to terminate.", id);

                    break;
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

## 정리
간단한 웹 서버를 구현하는 것으로 Rust 코드 따라하기를 함께 했습니다. 본 글의 소스 코드는 [Rust 프로그래밍 언어 문서](https://doc.rust-lang.org/nightly/book/title-page.html)의 것이며 본 글은 그 코드를 따라하면서 Rust 언어를 익히기 위함입니다. 모두 수고하셨습니다!