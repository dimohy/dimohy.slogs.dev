## TypeScript 배우기 - 3. 일반 유형

이번 장에서는 JavaScript 코드의 가장 흔한 타입들을 다루고, 이 타입을 TypeScript에서는 어떻게 표현하는지 설명합니다.
우선 JavaScript 또는 TypeScript 코드에서 기본적이며 흔하게 만날 수 있는 타입을 살펴보는 것부터 시작해봅시다.

## 일반 유형

### 원시 타입 : string, number, boolean
원시 타입이란 객체가 아니면서 또한 메서드도 가지지 않는 데이터 타입을 말합니다. JavaScript는 이러한 원시 타입으로 `string`, `number`, `boolean`이 있습니다. 
- `string`은 "Hello, world!"`와 같은 문자열 값
- `number`는 `42`와 같은 숫자. JavaScript는 정수와 실수를 구분하지 않으므로 `int` 또는 `float`과 같은 것은 존재하지 않음
- `boolean`은 `true`와 `false`라는 두가지 값을 가짐


### 배열
`[1, 2, 3]과 같은 숫자 배열의 경우 `number[]` 구문을 사용할 수 있습니다. 마찬가지로 `string[]`은 문자열 배열을 의미합니다.


### `any`

TypeScript의 `any`는 어떠한 타입도 허용한다는 의미로 그러므로 정적 타입 검사를 수행할 수 없게 됩니다. 아래의 코드는 `any` 유형에 의해 컴파일 오류가 발생하지 않습니다.

```typescript
let obj: any = { x: 0 };
// 아래 이어지는 코드들은 모두 오류 없이 정상적으로 실행됩니다.
// `any`를 사용하면 추가적인 타입 검사가 비활성화되며,
// 당신이 TypeScript보다 상황을 더 잘 이해하고 있다고 가정합니다.
obj.foo();
obj();
obj.bar = 100;
obj = "hello";
const n: number = obj;
```

`any` 유형은 이미 잘 동작하는 JavaScript 코드를 TypeScript 코드로 전환할 때 유용합니다. 하지만 일반적인 경우에 `any` 유형을 사용하는 것은 정적 유형 검사를 할 수 없게 되므로 'noImplicitAny` 플래그를 통해 암묵적으로 유형이 `any`로 간주하는 모든 경우를 오류로 처리할 수 있습니다.


### 변수에 대한 타입 표기
`const`, `var`, `let`을 사용하여 변수를 선언할 때 변수 타입을 명시적으로 지정하거나 TypeScript가 유추가 가능할 경우 생략할 수 있습니다.

```typescript
let myName: string = "Alice";
```

```typescript
// 타입 표기가 필요하지 않습니다. 'myName'은 'string' 타입으로 추론됩니다.
let myName = "Alice";
```


### 함수

함수는 JavaScript에서 데이터를 주고 받는 수단입니다. 이 표현은 핸드북에 있는 표현인데 매력적이네요! TypeScript는 함수로 전달하는 입력 및 반환되는 출력에 타입을 지정할 수 있습니다.


#### 매개변수 타입

함수를 선언할 때 함수에 전달할 매개변수에 타입을 표기할 수 있습니다.

```typescript
// 매개변수 타입 표기
function greet(name: string) {
  console.log("Hello, " + name.toUpperCase() + "!!");
}
```

매개변수 타입에 의해 다음의 코드는 오류가 됩니다.

```typescript
declare function greet(name: string): void;
// ---셍략---
// 만약 실행되면 런타임 오류가 발생하게 됩니다!
greet(42);
      ~~
Argument of type 'number' is not assignable to parameter of type 'string'.
```

#### 반환 타입 표기

TypeScript는 반환 값에 대한 타입도 표기합니다.

```typescript
function getFavoriteNumber(): number {
  return 26;
}
```

이것을 생략할 수 도 있는데 TypeScript는 타입을 유추할 수 있으면 `let`, `var`, `const`에서의 타입 생략과 마찬가지로 생략이 가능합니다.

```typescript
function getFavoriteNumber() {
  return 26; // number이므로 반환 타입은 생략 가능하고 `number`로 유추함
}
```

### 익명 함수

익명함수에서의 타입은 추론이 가능하므로 생략됩니다.

```typescript
// 아래 코드에는 타입 표기가 전혀 없지만, TypeScript는 버그를 감지할 수 있습니다.
const names = ["Alice", "Bob", "Eve"];
 
// 함수에 대한 문맥적 타입 부여
names.forEach(function (s) {
  console.log(s.toUppercase());
                ~~~~~~~~~~~~~
Property 'toUppercase' does not exist on type 'string'. Did you mean 'toUpperCase'?
});
 
// 화살표 함수에도 문맥적 타입 부여는 적용됩니다
names.forEach((s) => {
  console.log(s.toUppercase());
                ~~~~~~~~~~~~~
Property 'toUppercase' does not exist on type 'string'. Did you mean 'toUpperCase'?
});
```

여기서 `s`는 문자열로 추론되므로 `toUppercase`의 오타를 올바로 잡아 오류로 처리합니다.


### 객체 타입

원시 타입을 제외하고 가장 많이 사용되는 타입은 객체 타입 입니다. 객체는 JavaScript에서 속성으로 이루어진 것을 말하는데 `{ x: number; y: number }`등이 됩니다.

```typescript
// 매개 변수의 타입은 객체로 표기되고 있습니다.
function printCoord(pt: { x: number; y: number }) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
printCoord({ x: 3, y: 7 });
```

#### 옵셔널 속성

모든 속성을 전달하지 않아도 될 때가 있죠. 그런 경우 `name?`으로 속성명 뒤에 `?`을 두면 필수 속성이 아니라는 것을 표현할 수 있습니다.

```typescript
function printName(obj: { first: string; last?: string }) {
  // ...
}
// 둘 다 OK
printName({ first: "Bob" });
printName({ first: "Alice", last: "Alisson" });
```

JavaScript에서는 존재하지 않는 속성에 접근할 때 런타임 오류가 발생하지 않고 `undefined` 값을 얻게 되는데 그렇기 때문에 옵셔널 속성을 처리할 때는 해당 값을 사용하기 앞서서  `undefined`인지를 확인해야 합니다.

```typescript
function printName(obj: { first: string; last?: string }) {
  // 오류 - `obj.last`의 값이 제공되지 않는다면 프로그램이 멈추게 됩니다!
  console.log(obj.last.toUpperCase());
// Object is possibly 'undefined'.
  if (obj.last !== undefined) {
    // OK
    console.log(obj.last.toUpperCase());
  }
 
  // 최신 JavaScript 문법을 사용하였을 때 또 다른 안전한 코드
  console.log(obj.last?.toUpperCase());
}
```


### 유니언 타입

TypeScript의 타입 시스템은 기존 타입을 조합하여 새로운 타입으로 만들 수 있습니다.


#### 유니언 타입 정의

타입을 조합하는 첫번째 방법은 유니언 타입을 사용하는 것입니다. 조합은 `|`으로 하며 조합에 해당 타입을 모두 허용합니다. 이 때 허용하는 타입을 유니언 타입의 `멤버`라고 합니다.

```typescript
function printId(id: number | string) {
  console.log("Your ID is: " + id);
}
// OK
printId(101);
// OK
printId("202");
// 오류
printId({ myID: 22342 });
// Argument of type '{ myID: number; }' is not assignable to parameter of type 'string | number'.
```

`printId` 함수의 인자 `id`는 `number` 또는 `string` 타입을 허용합니다. 그러므로 `{ myID: 22342 }`의 객체 타입은 오류가 발생합니다.


#### 유니언 타입 사용

유니언 타입을 사용하게 되면 유니언 타입 멤버가 공통으로 제공하는 값, 속성, 함수일 경우만 오류가 발생하지 않습니다.

```typescript
function printId(id: number | string) {
  console.log(id.toUpperCase());
                 ~~~~~~~~~~~~~
Property 'toUpperCase' does not exist on type 'string | number'.
  Property 'toUpperCase' does not exist on type 'number'.
}
```

위의 코드의 경우 `toUpperCase` 함수는 `string` 타입에만 사용할 수 있는 함수로 `number`에서는 제공하지 않으므로 `number | string` 유니온 타입으로 호출할 수 없는 함수가 됩니다.

이를 해결하려면 `typeof`를 이용해 제공하는 변수의 타입이 무엇인지 분기해야 합니다. 이 방법은 JavaScript의 그것과 동일합니다.

```typescript
function printId(id: number | string) {
  if (typeof id === "string") {
    // 이 분기에서 id는 'string' 타입을 가집니다
 
    console.log(id.toUpperCase());
  } else {
    // 여기에서 id는 'number' 타입을 가집니다
    console.log(id);
  }
}
```

또다른 예시는 `string[] | string` 유니온 타입일 경우 `Array.isArray()` 등으로 분기하는 것입니다.

```typescript
function welcomePeople(x: string[] | string) {
  if (Array.isArray(x)) {
    // 여기에서 'x'는 'string[]' 타입입니다
    console.log("Hello, " + x.join(" and "));
  } else {
    // 여기에서 'x'는 'string' 타입입니다
    console.log("Welcome lone traveler " + x);
  }
}
```

유니온 타입 멤버들이 모두 가지고 있는 속성이나 함수 호출은 허용됩니다.

```typescript
// 반환 타입은 'number[] | string'으로 추론됩니다
function getFirstThree(x: number[] | string) {
  return x.slice(0, 3);
}
```

위의 코드의 경우 `number[]`와 `string` 모두 `slice`함수를 가지므로 오류 없는 정상 코드입니다.


### 타입 별칭

위의 코드들은 별도의 타입 별칭을 사용하지 않는 코드 였습니다. 하지만 실제로 코드를 작성할 때는 반복되는 유형을 타입 별칭으로 정의해서 사용하는 것이 편합니다.

```typescript
type Point = {
  x: number;
  y: number;
};
 
// 앞서 사용한 예제와 동일한 코드입니다
function printCoord(pt: Point) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
 
printCoord({ x: 100, y: 100 });
```

위의 코드는 `{ x: numner; y: number }`로 별칭(alias) 하였습니다. 이후 별칭된 이름으로 그 타입을 사용할 수 있습니다.

또한 다음의 유니언 타입 또한 타입 별칭으로 사용할 수 있습니다.

```typescript
type ID = number | string;
```

타입 별칭은 완전한 새로운 타입을 만드는 것이 아닙니다. 다음의 코드는 다른 타입 별칭이지만 오류가 발생하지 않습니다. 타입 별칭이 의미하는 타입이 동일하게 `string`이기 때문입니다.

```typescript
declare function getInput(): string;
declare function sanitize(str: string): string;
// ---중간 생략---
type UserInputSanitizedString = string;
 
function sanitizeInput(str: string): UserInputSanitizedString {
  return sanitize(str);
}
 
// 보안 처리를 마친 입력을 생성
let userInput = sanitizeInput(getInput());
 
// 물론 새로운 문자열을 다시 대입할 수도 있습니다
userInput = "new input";
```

### 인터페이스

타입 별칭과 유사하지만 다른 `인터페이스`는 해당 인터페이스에서 규정한 구조와 능력에만 초점을 맞추어서 사용하는 방식입니다.

```typescript
interface Point {
  x: number;
  y: number;
}
 
function printCoord(pt: Point) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
 
printCoord({ x: 100, y: 100 });
```

위의 `printCoord` 함수는 매개변수로 받는 `pt`가 `Point` 인터페이스와 맞게 전달이 된다면 정상 코드로 해석하며, 인터페이스에 정의된 방식 -- 여기서는 `x`라는 `number` 속성과 `y`라는 `number` 속성 -- 에만 부합하면 동작합니다. 이렇게 타입이 가지는 구조와 능력에만 관심을 가진다는 점에서 TypeScript는 구조적 타입 시스템이라고 불립니다.


#### 타입 별칭과 인터페이스의 차이점

그렇다면 매우 유사해 보이는 타입 별칭과 인터페이스와는 어떤 차이점이 있을까요? 대표적인 차이점은 `interface`는 새 속성을 추가할 수 있지만 `type`은 불가능 하다는 점입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656485597334/m1Aoq_eTA.png align="left")


### 타입 단언

TypeScript가 파악하고 있는 타입보다 좀 더 명확한 타입을 알고 있을 경우 `as`를 사용해서 코드로 표현할 수 있습니다.

```typescript
const myCanvas = document.getElementById("main_canvas") as HTMLCanvasElement;
```

타입 표기와 마찬가지로 타입 단언 표현은 컴파일러에 의해 제거됩니다. 코드가 `.tsx`가 아닌 경우 꺽쇠 괄호를 사용하는 방법도 가능합니다.

```typescript
const myCanvas = <HTMLCanvasElement>document.getElementById("main_canvas");
```

하지만 TypeScript에서는 타입 단언에 의해 변환되는 타입을 `보다 구체적인` 또는 `덜 구체적인` 의미를 가지는 타입 (다른 표현으로 상속관계)만 허용합니다. 다음의 코드 처럼 `string`을 `number`로 변환하는 타입 단언은 허용하지 않습니다.

```typescript
const x = "hello" as number;
// Conversion of type 'string' to type 'number' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first.
```

이런 특징 떄문에 유효할 수 있는 강제 변환이 허용되지 않을 수 도 있습니다. 이런 경우 아래의 코드 처럼 `any`로 변환한 뒤 다시한번 변환하는 것으로 두번의 단언으로 가능합니다.

```typescript
declare const expr: any;
type T = { a: 1; b: 2; c: 3 };
// ---중간 생략---
const a = (expr as any) as T;
```


### 리터럴 타입

TypeScript에서는 구체적인 문자열 또는 숫자를 타입으로 표현할 수 있습니다.

```typescript
let changingString = "Hello World";
changingString = "Olá Mundo";
// 변수 `changingString`은 어떤 문자열이든 모두 나타낼 수 있으며,
// 이는 TypeScript의 타입 시스템에서 문자열 타입 변수를 다루는 방식과 동일합니다.
changingString;
// let changingString: string
      
let changingString: string
 
const constantString = "Hello World";
// 변수 `constantString`은 오직 단 한 종류의 문자열만 나타낼 수 있으며,
// 이는 리터럴 타입의 표현 방식입니다.
constantString;
// const constantString: "Hello World"
```

이를 리터럴 타입이라고 합니다.

```typescript
let x: "hello" = "hello";
// OK
x = "hello";
// ...
x = "howdy";
Type '"howdy"' is not assignable to type '"hello"'.
```

하지만 하나의 리터럴 타입만 허용하는 것은 거의 의미가 없죠. 그렇기 때문에 대부분 유니언과 함께 사용되게 됩니다.

```typescript
function printText(s: string, alignment: "left" | "right" | "center") {
  // ...
}
printText("Hello, world", "left");
printText("G'day, mate", "centre");
// Argument of type '"centre"' is not assignable to parameter of type '"left" | "right" | "center"'.
```

숫자 리터럴 또한 리터럴 타입으로 유니언과 함께 사용될 수 있습니다.

```typescript
function compare(a: string, b: string): -1 | 0 | 1 {
  return a === b ? 0 : a > b ? 1 : -1;
}
```

유니언에 의해 리터럴이 아닌 타입과도 함께 사용할 수 있습니다.

```typescript
interface Options {
  width: number;
}
function configure(x: Options | "auto") {
  // ...
}
configure({ width: 100 });
configure("auto");
configure("automatic");
          ~~~~~~~~~~~
Argument of type '"automatic"' is not assignable to parameter of type 'Options | "auto"'.
```

`boolean`또한 리터럴 타입으로 표현할 수 있는데 사실 `boolean`는 `true | false` 유니언 타입의 별칭이라 할 수 있습니다. 매우 강력한 타입 시스템이네요!


#### 리터럴 추론

TypeScript에서는 변수가 초기화 되면 해당 객체의 속성은 변화할 수 있다고 가정합니다.

```typescript
declare const someCondition: boolean;
// ---중간 생략---
const obj = { counter: 0 };
if (someCondition) {
  obj.counter = 1;
}
```

동일한 사항이 문자열에도 적용됩니다.

```typescript
const req = { url: "https://example.com", method: "GET" };
handleRequest(req.url, req.method);
// Argument of type 'string' is not assignable to parameter of type '"GET" | "POST"'.
```

위의 예시에서 `req.method`는 `string`이고 코드의 어느 지점에서 `GUESS`등으로 바뀔 수도 있으므로 `handleRequest`함수의 두번째 인자에서 요구하는 `"GET" | "POST"` 유닌온 타입과 다르다고 평가하여 오류가 발생합니다.
이를 해결하려면 다음처럼 처리할 수 있습니다.

```typescript
// 수정 1:
const req = { url: "https://example.com", method: "GET" as "GET" };
// 수정 2
handleRequest(req.url, req.method as "GET");
```

또는 `as const`를 이용해서 전체를 리터럴로 처리할 수 있습니다.

```typescript
declare function handleRequest(url: string, method: "GET" | "POST"): void;
// ---중간 생략---
const req = { url: "https://example.com", method: "GET" } as const;
handleRequest(req.url, req.method);
```

`as const`는 일반적인 `const`와 유사하게 작동하는데 해당 객체의 모든 속성에 `string`또는 `number`와 같은 보다 일반적인 타입이 아닌 리터럴 타입의 값이 사용되도록 보장합니다.


### `null`과 `undefined`

JavaScript에서는 빈값 `null`과 초기화 되지 않는 값 `undefined`의 원시 값이 존재합니다.
TypeScript에서는 각 값에 대응하는 두가지 타입으로 존재합니다.

#### `strictNullChecks` 플래그 미설정
`strictNullChecks` 플래그가 설정되지 않았으면 모든 타입에 `null` 또는 `undefined`가 대입될 수 있습니다. 하지만 이는 대부분 버그로 이어질 수 있으므로 `strictNullChecks` 플래그를 활성화 하는 것을 권장합니다.

#### `strictNullChecks` 플래그 설정
`strictNullChecks` 플래그가 설정되었아면 `null`과 `undefined`는 타입으로 간주합니다.

```typescript
function doSomething(x: string | undefined) {
  if (x === undefined) {
    // 아무 것도 하지 않는다
  } else {
    console.log("Hello, " + x.toUpperCase());
  }
}
```


#### Null이 아님 단언 연산자 (접미사 !)

TypeScript에서는 `null` 또는 `undefined`가 아닐 경우 `!` 접미사를 통해 검사를 하지 않도록 할 수 있습니다.

```typescript
function liveDangerously(x?: number | undefined) {
  // 오류 없음
  console.log(x!.toFixed());
}
```

하지만  `!` 접미사를 사용하면 검사를 수행하지 않으므로 해당 값이 반드시 `null` 또는 `undefined`가 아닌 경우에만 사용해야 합니다.


### 열거형

열거형은 JavaScript에서 제공하는 것이 아니라 TypeScript에서 언어와 런타임 수준에서 추가하는 기능입니다. 열거형은 [Enums](https://www.typescriptlang.org/ko/docs/handbook/enums.html)에서 자세히 다룹니다.


### bigint

ES2020 이후 아주 큰 정수를 다루기 위한 `bigint`라는 원시 타입이 JavaScript에 추가되었습니다.

```typescript
// BigInt 함수를 통하여 bigint 값을 생성
const oneHundred: bigint = BigInt(100);
 
// 리터럴 구문을 통하여 bigint 값을 생성
const anotherHundred: bigint = 100n;
```


### symbol

`symbol`은 전역적으로 고유한 참조값을 생성하는데 사용할 수 있는 원시 타입이며, `Symbol()` 함수를 통해 생성할 수 있습니다.

```typescript
const firstName = Symbol("name");
const secondName = Symbol("name");
 
if (firstName === secondName) {
This condition will always return 'false' since the types 'typeof firstName' and 'typeof secondName' have no overlap.
  // 절대로 일어날 수 없습니다
}
```

## 정리
오늘은 TypeScript의 일반 타입에 대해 알아보았습니다. 다음 시간에는 핸드북의 `Narrowing`에 대해 살펴보겠습니다.

