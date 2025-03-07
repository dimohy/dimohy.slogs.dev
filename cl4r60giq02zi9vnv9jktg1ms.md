---
title: "TypeScript 배우기 - 2. 기초"
datePublished: Thu Jun 23 2022 15:14:20 GMT+0000 (Coordinated Universal Time)
cuid: cl4r60giq02zi9vnv9jktg1ms
slug: typescript-2
tags: typescript

---

JavaScript는 다양한 연산을 내장하고 있습니다.

```javascript
// 'message'의 프로퍼티 'toLowerCase'에 접근한 뒤
// 이를 호출합니다
message.toLowerCase();

// 'message'를 호출합니다
message();
```

그런데 message에 `toLowerCase()`로 접근한 뒤 다시 message를 함수처럼 부르고 있씁니다.

하지만 message가 무엇인지 알 수 없다면 동작은 기대하지 않는 결과를 발생할 것입니다.

- `message`가 호출 가능한가?
- `toLowerCase`라는 프로퍼티가 있는가?
- `toLowerCase`가 만약 있다면 호출 가능한가?
- 만약 두 값이 모두 호출 가능하다면 무엇이 반환되는가?

이런 질문은 JavaScript로 코드를 작성하게 되면 생각해야 하는 문제이며, 결국에 모든 것을 개발자가 추적해야 하는 문제에 놓이게 됩니다.

```javascript
const message = "Hello World!";
```

message가 문자열인 경우 두번째 `message()`에서 다음의 유형 오류가 발생합니다.

```
TypeError: message is not a function
```

코드가 짧아 지금은 예상할 수 있는 결과지만 코드가 복잡해지면 결코 찾기 쉬운 코드가 아닙니다. 실행하기 전에 미리 이런 오류를 잡을 수 있다면 좋겠습니다.

```javascript
function fn(x) {
  return x.flip();
}
```

위의 코드는 fn 함수가 호출되기 전까지 인자의 x가 `flip`이라는 함수를 가졌는지를 알 수 없습니다. 만약 가졌다면 함수의 결과값이 반환 될 것이고 만약 없다면 함수로 호출할 수 없을 것입니다. 이것은 동작 중에 판가름 되어 오동작의 여지를 남깁니다.

이에 대한 대안으로 정적 타입 시스템을 이용해서 코드가 실행되기 전에 문제를 예측하는 것입니다.


## 정적 타입 검사

만약에 실행하기 전에 이런 유형 문제를 확인할 수만 있다면 실행 시점에서 파악할 수 있는 다양한 문제점을 미리 점검할 수 있습니다.  TypeScript을 이용한다면 이 문제를 해결 할 수 있습니다.

```typescript
const message = "hello!";
 
message();
~~~~~~~~~
This expression is not callable.
  Type 'String' has no call signatures.
```

TypeScript에서는 코드를 실행하기 전에 `message`가 호출될 수 없음을 확인할 수 있습니다.


## 예외가 아닌 실행 실패

위와 같이 명확한 오류를 실행 시점이 아닌 컴파일 시점에서 확인할 수 있는 것은 어찌 보면 당연한 동작입니다. 하지만 존재하지 않는 속성에 접근하는 것은 JavaScript에서는 정상적인 동작입니다.

```javascript
const user = {
  name: "Daniel",
  age: 26,
};
user.location; // undefined 를 반환
```

하지만 TypeScript에서는 이를 오류로 간주하고 다음의 오류 메시지를 출력합니다.

```typescript
const user = {
  name: "Daniel",
  age: 26,
};
 
user.location;
     ~~~~~~~~
Property 'location' does not exist on type '{ name: string; age: number; }'.
```

이는 JavaScript의 표현의 유연성을 희생하지만 이로 인해 오타로 인해 발생하는 버그를 미연에 찾을 수 있게 됩니다.

```typescript
const announcement = "Hello World!";
 
// 바로 보자마자 오타인지 아실 수 있나요?
announcement.toLocaleLowercase();
announcement.toLocalLowerCase();
 
// 아마 아래와 같이 적으려 했던 것이겠죠...
announcement.toLocaleLowerCase();
```

호출되지 않는 함수,

```typescript
function flipCoin() {
  // 본래 의도는 Math.random()
  return Math.random < 0.5;
Operator '<' cannot be applied to types '() => number' and 'number'.
}
```

또는 기본적인 논리 오류

```typescript
const value = Math.random() < 0.5 ? "a" : "b";
if (value !== "a") {
  // ...
} else if (value === "b") {
This condition will always return 'false' since the types '"a"' and '"b"' have no overlap.
  // 이런, 이 블록은 실행되지 않겠군요
}
```


## 프로그램 도구로서의 타입

TypeScript의 타입 검사기는 변수 또는 올바른 속성에 접근하고 있는지 여부를 검사할 수 있는 정보를 제공합니다. 이 정보를 이용해 Visual Studio Code 등의 코드 편집기에서 타입 검사기를 통해 즉각적인 오류 및 힌트를 제공받을 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655994862957/ZEW_1fexy.png align="left")

TypeScript를 지원하는 코드 편집기는 오류를 자동으로 고쳐주는 기능, 코드를 간편하게 재조직하는 리팩토링, 변수의 정의로 빠르게 이동하는 네비게이션, 주어진 변수에 대한 모든 참조 검색 등의 기능을 제공합니다. 이 모든 기능은 타입 검사기를 기반으로 하며 완전히 크로스 플랫폼으로 동작합니다.


## `tsc`, TypeScript 컴파일러

`tsc`는 TypeScript 컴파일러입니다. 다음처럼 설치할 수 있습니다.

```
npm install -g typescript
```

이제 첫번째 TypeScript 프로그램인 hello.ts를 작성해봅시다.

| hello.ts
```typescript
// 세상을 맞이하세요.
console.log("Hello world!");
```

이제 tsc를 사용해봅시다.

```
tsc hello.ts
```

짠! 그런데 아무것도 표시가 안되는군요. tsc는 컴파일러이므로 TypeScript 코드에 문제가 없다면 동일한 파일명으로 `hello.js`가 생성됩니다.

| hello.js
```javascript
// 세상을 맞이하세요.
console.log("Hello world!");
```

`tsc`에서 타입 검사 오류를 발생하는지 확인하기 위해 아래의 코드를 다시 작성해봅시다.

```typescript
// 아래는 실무 수준에서 범용적으로 쓰이는 환영 함수입니다
function greet(person, date) {
  console.log(`Hello ${person}, today is ${date}!`);
}
 
greet("Brendan");
```

다시 `tsc hello.cs`를 실행하면 다음의 오류를 얻게 됩니다.

```
hello.ts:6:3 - error TS2554: Expected 2 arguments, but got 1.
```

TypeScript는 `greet()`에서 인자를 하나 누락했다는 것을 잘 알려줍니다.


## 오류 발생시키기

위의 코드는 오류 메시지를 출력하지만 여전히 `hello.js` 파일을 생성합니다. 이는 TypeScript에 의해 JavaScript로 변환되는 코드의 능력을 제안하지 않는 핵심 가치에 의해 기반한 동작입니다.

이는 기존의 JavaScript를 TypeScript로 전환하는 과정에서 부분적인 JavaScript를 수용하게 됩니다. 변환 과정중의 코드도 역시 JavaScript 코드로 잘 컴파일하고, 변환이 완료되면 더이상 오류 메시지는 없게 됩니다.

만약 오류가 발생했을 때 js 파일을 생성하지 않도록 하려면 다음처럼 컴파일 옵션을 부여할 수 있습니다.

```
tsc --noEmitOnError hello.ts
```


## 명시적 타입

TypeScript는 명시적 타입을 통해 타입 검사를 수행합니다. `greet` 함수를 다음처럼 수정해봅시다.

```typescript
function greet(person: string, date: Date) {
  console.log(`Hello ${person}, today is ${date.toDateString()}!`);
}

greet("Maddison", Date());
                  ~~~~~~
Argument of type 'string' is not assignable to parameter of type 'Date'.
```

여전이 오류가 발생하는데, JavaScript에서는 Date()를 호출하면 `string`을 반환하기 때문입니다. `Date()` 를 `new Date()`로 수정해봅시다.

```typescript
function greet(person: string, date: Date) {
  console.log(`Hello ${person}, today is ${date.toDateString()}!`);
}
 
greet("Maddison", new Date());
```

이제 오류가 발생하지 않는군요!

명시적인 타입 표시를 무조건 고수할 필요는 없습니다.  컴파일 시점에서 타입을 추론할 수 있다면 `let`을 써서 다음과 같이 사용할 수 도 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655996182163/V0lxPJbXl.png align="left")
``typescript
let msg = "hello there!";
```

TypeScript 타입 검사기는 `msg` 변수를 `string`으로 정확히 추론 합니다.


## 지워진 타입

이제 `tsc`로 컴파일하여 출력된 JavaScript 코드를 살펴보죠.

```javascript
"use strict";
function greet(person, date) {
    console.log("Hello ".concat(person, ", today is ").concat(date.toDateString(), "!"));
}
greet("Maddison", new Date());
```

1. `person`과 `date` 인자는 더이상 타입을 표기하지 않습니다.
1. `템플릿 문자열`이 연결 연간자(+)로 변환되어 문자열을 표시합니다.

> 템플릿 문자열의 경우 타겟 JavaScript 버젼에 ES2015 이상의 경우 템플릿 문자열로 표현됩니다.


## 다운레벨링

앞의 `템플릿 문자열`의 사례처럼 아래의 내용에서,

```typescript
`Hello ${person}, today is ${date.toDateString()}!`;
```

아래의 내용으로 변환되었습니다.

```javascript
"Hello " + person + ", today is " + date.toDateString() + "!";
```

이것은 ECMASCript 버젼에 따라 달라지며 TypeScript는 상위 또는 하위 버젼으로의 변환을 지원합니다. 이를 다운레벨링이라고 합니다.

`--target es2015`의 옵션으로 타겟 버젼을 변경할 수 있습니다. 이에 따라 생성되는 JavaScript 코드는 다음과 같습니다.

```javascript
function greet(person, date) {
  console.log(`Hello ${person}, today is ${date.toDateString()}!`);
}
greet("Maddison", new Date());
```

> 이제 대부분의 웹브라우저는 ES2015를 지원합니다.


## 엄격도

TypeScript는 정적 타입 검사를 느슨하게 하거나 또는 엄격하게 할 수 있습니다.  목적에 따라 이를 선택할 수 있습니다. 켜고 끌 수 있는 몇가지 플래그가 있으며 기본적으로 `--strict` 플래그는 활성화 되어 있습니다. 이를 옵션으로 변경하거나 `tsconfig.json`에서 `"strict":true` 또는 `false`로 조절할 수 있습니다.

다음으로 중요한 두가지 옵션인 `noImplicitAny`와  `strictNullChecks`에 대해 살펴봅시다.

### `noImplicitAny`
몇몇의 경우에 TypeScript는 값의 타입을 추론하지 않고 `any` 타입으로 간주합니다. 이는 JavaScript의 기본 동작이기도 합니다. 하지만 `any` 타입을 허용한다는 것은 TypeScript의 정적 타입 검사를 백분 활용하지 못하는 것일지도 모릅니다. `noImplicitAny` 플래그를 활성화 하면 모든 암묵적으로 `any`로 추론되는 변수까지 오류를 발생 시킵니다.

### `strictNullChecks`
`null`과 `undefined`와 같은 값을 명시적으로 처리하도록 활성화 합니다. 이를 허용했을 때 수많은 버그가 발생하기 때문입니다.


## 정리
함께 핸드북의 `The Basics`에 대해 살펴보았습니다. 이 시간을 통해 TypeScript를 사용해야 하는 의미를 확인했으며 TypeScript가 올바른 JavaScript를 생성하므로 여러분이 JavaScript 개발자로 취업 되더라도 TypeScript 정적 타입 검사기를 사용해서 버그 없는 JavaScript 코드를 생성할 수 있음도 알게 되었습니다.

다음 시간에는 핸드북의 `언어의 원시 타입들(Everyday Types)`에 대해 살펴보도록 합시다.
