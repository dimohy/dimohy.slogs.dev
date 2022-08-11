## TypeScript 배우기 - 5. 함수에 대한 추가 정보

오늘은 핸드북의 `More On Functions`을 같이 살펴보겠습니다.

함수는 로컬이나 모듈에서 가져온 것 등 모든 애플리케이션의 실행 블록 입니다. 함수가 반환 값이 있을 경우 함수 역시 값으로 취급되며 TypeScript에서 값을 취급하는 것처럼 함수를 호출하는 다양한 방법이 있습니다. 함수를 설명하는 유형을 작성하는 방법에 대해 알아봅시다.


## 함수 유형 표현식

먼저 함수 유형 표현식으로 함수에 대해 먼저 살펴봅시다.

```typescript
function greeter(fn: (a: string) => void) {
  fn("Hello, World");
}
 
function printToConsole(s: string) {
  console.log(s);
}
 
greeter(printToConsole);
```

`(a: string) => void`는 문자열 `a`인자를 가지는 함수(반환값 없음) 유형입니다. 함수 인자와 마찬가지로 만약 인자 유형이 없으면 `any` 유형이 됩니다. (여기서는 string 입니다.)

> 여기서 중요한 것. 인자 유형은 필수입니다. 만약 `(string) => void`라 했을 경우 `string`이라는 인자명의 `any` 유형이 됩니다!

함수 유형 표현식은 마치 C의 함수 포인터와 유사하군요. C#의 딜리게이트와도 유사합니다. 하지만 TypeScript의 함수 유형 표현식이 좀 더 쉽다고 느낍니다.

TypeScript의 다른 데이터 유형 처럼 `Type`을 써서 축약 할 수 있습니다.

```typescript
type GreetFunction = (a: string) => void;
function greeter(fn: GreetFunction) {
  // ...
}
```


## 호출 서명

JavaScript에서 함수는 호출할 수 있을 뿐만 아니라 속성도 가질 수 있습니다. 하지만 `함수 유형 표현식`으로는 속성을 가질 수 없는데요, 객체 유형에 속성과 함수 유형 표현식을 사용해서 속성을 사용할 수 있습니다.

```typescript
type DescribableFunction = {
  description: string;
  (someArg: number): boolean;
};
function doSomething(fn: DescribableFunction) {
  console.log(fn.description + " returned " + fn(6));
}
```

`DescribableFunction`은 객체 유형입니다. `description`이라는 설명 속성과 `(someArg: number): boolean`이라는 함수 유형 표현식으로 호출 가능한 값을 담습니다. 이제 `fn.description`과 `fn()`으로 함수로 호출할 수 있습니다.

구문은 좀 다르죠. `=>` 대신 `:`을 써서 type에서 표현하고 있습니다.

JavaScript의 `Date` 객체와 같은 일부 객체는 `new`를 포함하거나 포함하지 않고 동일한 유형의 호출 및 생성 서명을 임의로 결합할 수 있습니다.

```typescript
interface CallOrConstruct {
  new (s: string): Date;
  (n?: number): number;
}
```


## 제네릭 함수

제네릭 함수를 이해하기 앞서서 제네릭을 사용하지 않았을 때 겪는 문제를 먼저 살펴봅시다.

```typescript
function firstElement(arr: any[]) {
  return arr[0];
}
```

이 함수는 배열의 첫번째 원소의 값을 반환합니다. 인자 유형이 `any[]`이므로 반환값도 `any`가 됩니다. 동적 언어인 JavaScript에서 특별히 문제될 것이 없고 잘 동작할 것입니다. 하지만 약간의 문제는 유형을 특정하지 않았다는 점입니다. (any 유형일 때 어떤 문제가 발생할 수 있는지는 아시죠?)

그래서 제네릭을 사용해 다음처럼 유형을 명시할 수 있습니다.

```typescript
function firstElement<Type>(arr: Type[]): Type | undefined {
  return arr[0];
}
```

이제 제네릭 유형이 `Type`이므로 제네릭 유형 `Type`에 해당하는 모든 유형이 이 함수를 사용할 수 있음을 알 수 있습니다. 또한 `[]` 처럼 유형을 알 수 없을 때도 이 함수를 호출할 수 있게 되었습니다.

> `Type[]`으로 받으므로 반환값은 자연스럽게 `Type`이 됩니다.

```typescript
// s is of type 'string'
const s = firstElement(["a", "b", "c"]);
// n is of type 'number'
const n = firstElement([1, 2, 3]);
// u is of type undefined
const u = firstElement([]);
```


### 추론

위의 코드에서 `Type`을 지정할 필요는 없습니다. TypeScript에서 `Type`을 일반 유형으로 추론했기 때문입니다.

여러 제네릭 유형을 사용할 수 도 있습니다.

```typescript
function map<Input, Output>(arr: Input[], func: (arg: Input) => Output): Output[] {
  return arr.map(func);
}
 
// Parameter 'n' is of type 'string'
// 'parsed' is of type 'number[]'
const parsed = map(["1", "2", "3"], (n) => parseInt(n));
```


### 제약

하지만 모든 유형을 제네릭으로 받는다는 것은 그 유형의 고유한 속성 또는 기능을 사용할 수 없다는 의미이기도  합니다. 제네릭 유형이 어떤 유형인지 조금은 좁혀준다면 그 유형의 고유한 속성과 기능을 사용할 수 있게 됩니다.

```typescript
function longest<Type extends { length: number }>(a: Type, b: Type) {
  if (a.length >= b.length) {
    return a;
  } else {
    return b;
  }
}
 
// longerArray is of type 'number[]'
const longerArray = longest([1, 2], [1, 2, 3]);
// longerString is of type 'alice' | 'bob'
const longerString = longest("alice", "bob");
// Error! Numbers don't have a 'length' property
const notOK = longest(10, 100);
Argument of type 'number' is not assignable to parameter of type '{ length: number; }'.
```

`Type` 제네릭 유형은 이제 `{ length: number }`를 가지는 것으로 제약 합니다. 이제 매개변수 `a`와 `b`는 숫자형의 `length` 속성을 호출할 수 있게 됩니다!

반대로 `length` 속성이 없는 숫자형의 경우 위의 코드처럼 오류가 발생합니다.


## 제한된 값 작업

아래의 코드는 동작해야 하는 코드인 것처럼 보입니다.

```typescript
function minimumLength<Type extends { length: number }>(
  obj: Type,
  minimum: number
): Type {
  if (obj.length >= minimum) {
    return obj;
  } else {
    return { length: minimum };
  }
}
```
> Type '{ length: number; }' is not assignable to type 'Type'.
  '{ length: number; }' is assignable to the constraint of type 'Type', but 'Type' could be instantiated with a different subtype of constraint '{ length: number; }'.

하지만 `{ length : minium }`은 제네릭 `Type` 유형이 아니므로 TypeScript에서는 이를 오류로 평가합니다. 만약 이것이 오류가 아니라면 아래의 동작하지 않아야 하는 코드가 동작하게 됩니다.

```typescript
// 'arr' gets value { length: 6 }
const arr = minimumLength([1, 2, 3], 6);
// and crashes here because arrays have
// a 'slice' method, but not the returned object!
console.log(arr.slice(0));
```


### 유형 인수 지정

유추가 가능할 경우 TypeScript는 유추를 합니다. 하지만 아래의 코드처럼 유추가 불가능 할 수 도 있습니다.

```typescript
function combine<Type>(arr1: Type[], arr2: Type[]): Type[] {
  return arr1.concat(arr2);
}
```

```typescript
const arr = combine([1, 2, 3], ["hello"]);
```
> Type 'string' is not assignable to type 'number'.

아래처럼 수동으로 제네릭 유형을 지정할 수 있습니다.

```typescript
const arr = combine<string | number>([1, 2, 3], ["hello"]);
```


### 올바른 제네릭 함수 작성을 위한 지침

#### 제네릭 유형을 아래로 누름

```typescript
function firstElement1<Type>(arr: Type[]) {
  return arr[0];
}
 
function firstElement2<Type extends any[]>(arr: Type) {
  return arr[0];
}
 
// a: number (good)
const a = firstElement1([1, 2, 3]);
// b: any (bad)
const b = firstElement2([1, 2, 3]);
```

위의 두 함수는 동일한 처리를 하겠지만 `any[]`으로 제약하는 바람에 두번째 함수의 반환 유형은 `any`가 되었습니다.

> 가능하다면 제네릭 유형을 제한하지 말고 사용


#### 더 적은 수의 제네릭 유형 사용

```typescript
function filter1<Type>(arr: Type[], func: (arg: Type) => boolean): Type[] {
  return arr.filter(func);
}
 
function filter2<Type, Func extends (arg: Type) => boolean>(
  arr: Type[],
  func: Func
): Type[] {
  return arr.filter(func);
}
```

두 번째 함수 보다는 첫 번째 함수가 좀 더 가독성이 좋습니다.

> 항상 가능한 한 적은 수의 제네릭 유형 사용


#### 제네릭 유형은 반드시 두 번 나타나야 합니다.

다음의 코드는 굳이 제네릭 함수로 만들 필요가 없습니다.

```typescript
function greet<Str extends string>(s: Str) {
  console.log("Hello, " + s);
}
 
greet("world");
```

아래의 코드가 더 좋은 코드입니다.

```typescript
function greet(s: string) {
  console.log("Hello, " + s);
}
```

제네릭 유형은 여러 유형을 처리하기 위함입니다. 그렇지 않는 함수는 제네릭 함수로 표현하지 않는 것이 좋습니다.

> 제네릭 유형이 하나의 유형으로만 적용될 경우 사용을 강력히 재고


## 선택적 매개변수

TypeScript에서는 함수의 인자를 선택적으로 사용하기 위한 선택적 매개변수를 아래의 코드처럼 `?`를 주어 사용할 수 있습니다.

```typescript
function f(x?: number) {
  // ...
}
f(); // OK
f(10); // OK
```

이때 x는 `number`가 아니라 `number | undefined` 유니언 유형이 됩니다.

다음처럼 기본값을 제공할 수 도 있습니다.

```typescript
function f(x = 10) {
  // ...
}
```

하지만 x에 `undefined`를 대입할 수 도 있으므로 다음의 코드는 모드 정상적인 코드입니다ㅣ.

```typescript
declare function f(x?: number): void;
// cut
// All OK
f();
f(10);
f(undefined);
```


### 콜백의 선택적 매개변수

콜백의 선택적 매개변수는 다음처럼 사용할 수 있으리라 기대합니다.

```typescript
function myForEach(arr: any[], callback: (arg: any, index?: number) => void) {
  for (let i = 0; i < arr.length; i++) {
    callback(arr[i], i);
  }
}
```

위의 코드는 아래의 코드처럼 잘 호출됩니다.

```typescript
myForEach([1, 2, 3], (a) => console.log(a));
myForEach([1, 2, 3], (a, i) => console.log(a, i));
```

하지만 아래의 코드는 어떨까요?

```typescript
myForEach([1, 2, 3], (a, i) => {
  console.log(i.toFixed());
});
```
> Object is possibly 'undefined'.

`i`는 `undefined`일 수 있으므로 이렇게 사용할 수 없습니다.

TypeScript는 콜백함수로 전달하는 매개변수를 설령 사용하지 않더라도 이를 무시합니다. 즉 아래의 코드로 사용할 수 있습니다.

```typescript
function myForEach(arr: any[], callback: (arg: any, index: number) => void) {
  for (let i = 0; i < arr.length; i++) {
    callback(arr[i], i);
  }
}

myForEach([1, 2, 3], (a) => console.log(a));
myForEach([1, 2, 3], (a, i) => {
  console.log(i.toFixed());
});
```


## 함수 오버로드

TypeScript에서는 다양한 인수 수 및 유형으로 호출할 수 있도록 오버로드 서명을 통해 이를 가능하게 합니다.

```typescript
function makeDate(timestamp: number): Date;
function makeDate(m: number, d: number, y: number): Date;
function makeDate(mOrTimestamp: number, d?: number, y?: number): Date {
  if (d !== undefined && y !== undefined) {
    return new Date(y, mOrTimestamp, d);
  } else {
    return new Date(mOrTimestamp);
  }
}
const d1 = makeDate(12345678);
const d2 = makeDate(5, 5, 5);
const d3 = makeDate(1, 3);
> No overload expects 2 arguments, but overloads do exist that expect either 1 or 3 arguments.
```

오버로드 서명에 의해 마지막 두개의 인자 호출은 오류가 되었습니다.


### 오버로드 서명 및 구현 서명

오버로드 서명과 구현 서명이 일치하지 않으면 오류 입니다.

```typescript
function fn(x: string): void;
function fn() {
  // ...
}
// Expected to be able to call with zero arguments
fn();
> Expected 1 arguments, but got 0.
```

아래는 유사한 오류의 예입니다.

```typescript
function fn(x: boolean): void;
// Argument type isn't right
function fn(x: string): void;
> This overload signature is not compatible with its implementation signature.
function fn(x: boolean) {}
```

```typescript
function fn(x: string): string;
// Return type isn't right
function fn(x: number): boolean;
> This overload signature is not compatible with its implementation signature.
function fn(x: string | number) {
  return "oops";
}
```

### 좋은 오버로드 작성하기

제네릭과 마찬가지로 함수 오버로드도 따라야 하는 지침이 있습니다. 다음의 코드를 살펴봅시다.

```typescript
function len(s: string): number;
function len(arr: any[]): number;
function len(x: any) {
  return x.length;
}
```

이제 함수 오버로드에 의해 `string`이나 `any[]` 유형만 호출이 가능해졌습니다. 그러므로 아래의 코드중 마지막 유니언 타입은 오류가 발생합니다.

```typescript
len(""); // OK
len([0]); // OK
len(Math.random() > 0.5 ? "hello" : [0]);
```
> No overload matches this call.
>   Overload 1 of 2, '(s: string): number', gave the following error.
>     Argument of type 'number[] | "hello"' is not assignable to parameter of type 'string'.
>       Type 'number[]' is not assignable to type 'string'.
>   Overload 2 of 2, '(arr: any[]): number', gave the following error.
>     Argument of type 'number[] | "hello"' is not assignable to parameter of type 'any[]'.
>       Type 'string' is not assignable to type 'any[]'.

그러므로 두개의 유형을 받아야 할 경유 오버로드 함수보다는 유니언 유형을 인자로 받는게 좋습니다.

```typescript
function len(x: any[] | string) {
  return x.length;
}
```

> 가능한 경우 오버로드 대신 유니언 유형을 매개변수로 사용


#### 함수에서 this 선언

JavaScript에서는 `this`를 매개변수로 가질 수 없다고 명시되어 있으므로 TypeScript에서는 해당 구문 공간을 이용해서 함수 본문에서 this에 대한 유형을 선언할 수 있도록 합니다.

```typescript
interface DB {
  filterUsers(filter: (this: User) => boolean): User[];
}
 
const db = getDB();
const admins = db.filterUsers(function (this: User) {
  return this.admin;
});
```

이 패턴은 함수가 호출될 때 다른 객체가 제어하는 콜백 스타일 API에서 일반적입니다. 이 동작을 얻으려면 람다 형태가 아닌 함수 형태를 사용해야 합니다.

```typescript
interface DB {
  filterUsers(filter: (this: User) => boolean): User[];
}
 
const db = getDB();
const admins = db.filterUsers(() => this.admin);
```
> The containing arrow function captures the global value of 'this'.
> Element implicitly has an 'any' type because type 'typeof globalThis' has no index signature.


## 알아야 할 다른 유형

다음으로 알아야 할 추가 유형에 대해 살펴봅시다.

### `void`

`void`란 값을 반환하지 않는 함수의 반환 유형을 나타냅니다.

```typescript
// The inferred return type is void
function noop() {
  return;
}
```

JavaScript에서는 값을 반환하지 않을 경우 암시적으로 `undefined`를 반환합니다. 하지만 TypeScript에서 `void`는 `undefined`와 같지 않습니다.

> undeinfed와 void는 같지 않습니다.


### `object`

`object`는 기본 유형(string, number, bigint, boolean, symbol, null, undefined)를 제외한 모든 유형의 값을 참조합니다. 이것은 빈 객체 유형 `{}`과 전역 유형 `Object`과도 다릅니다. 여러분은 `Object`를 사용하지 않을 가능성이 큽니다.

> `object`는 `Object`가 아닙니다. 항상 object를 사용!

JavaScript에서 함수는 객체입니다. 속성이 있고 프로토타입 체인에 `Object.prototype`이 있고 `instanceof Object`이고 `Object.keys`를 호출할 수 있습니다. 이러한 이유로 함수 유형은 TypeScript에서 `object`로 간주합니다.


### `unknown`

알 수 없는 유형은 모든 값을 나타냅니다. `any`와 유사하지만 아무것도 하지 않기 때문에 좀 더 안전합니다.

```typescript
function f1(a: any) {
  a.b(); // OK
}
function f2(a: unknown) {
  a.b();
> Object is of type 'unknown'.
}
```

`any`는 본문에 값이 없어도 모든 값을 허용하는 함수를 설명할 수 있기 때문에 함수 유형을 설명할 때 유용합니다. 반대로 `unknown`는 알 수 없는 유형의 값을 반환하는 함수를 설명하는데 유용합니다.

```typescript
function safeParse(s: string): unknown {
  return JSON.parse(s);
}
 
// Need to be careful with 'obj'!
const obj = safeParse(someRandomString);
```


### `never`

`never`는 함수에서 결코 값을 반환하지 않는다는 의미로 쓰입니다. 예로 아래의 코드는 예외가 발생하므로 반환 유형이 `never`가 됩니다.

```typescript
function fail(msg: string): never {
  throw new Error(msg);
}
```

또한 관찰되지 않는 유형도 `never`가 됩니다. 유니언 값을 분기할 때 아무것도 남아있지 않다고 판단할 때도 유용하게 쓰입니다.


### `Function`

전역 유형인 함수는 바인딩, 호출, 적용 및 JavaScript의 모든 함수 값에 있는 속성을 설명합니다. 또한 Function 유형의 값을 항상 호출할 수 있는 특수 속성이 있습니다. 이러한 호출은 `any`를 반환합니다

```typescript
function doSomething(f: Function) {
  return f(1, 2, 3);
}
```

하지만 이는 형식화되지 않은 함수 호출이며 `any`를 반환하기 때문에 피하는 것이 좋습니다. 항상 호출할 수 있도록 하는 장치로 `() => void`이 좀 더 안전합니다.


## 나머지 매개변수 및 인수

### 나머지 매개변수

`...`를 이용해서 함수에 여려 매개 변수를 받을 수 있습니다.

```typescript
function multiply(n: number, ...m: number[]) {
  return m.map((x) => n * x);
}
// 'a' gets value [10, 20, 30, 40]
const a = multiply(10, 1, 2, 3, 4);
```


### 나머지 인수

반대로 스프레드 구문을 이용해서 배열을 다양한 수의 인수로 제공할 수 있습니다.,

```typescript
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
arr1.push(...arr2);
```

TypeScript에서는 배열에서 인수를 제공할 때는 배열의 길이가 변하지 않을 것을 전제합니다.

```typescript
// Inferred type is number[] -- "an array with zero or more numbers",
// not specifically two numbers
const args = [8, 5];
const angle = Math.atan2(...args);
```
> A spread argument must either have a tuple type or be passed to a rest parameter.

이를 해결하려는 방법은 `const`가 가장 간단한 솔루션입니다.

```typescript
// Inferred as 2-length tuple
const args = [8, 5] as const;
// OK
const angle = Math.atan2(...args);
```


## 매개변수 분해

다음의 코드처럼 압축 해제할 수 있습니다. JavaScript에서는 다음과 같습니다.

```typescript
function sum({ a, b, c }) {
  console.log(a + b + c);
}
sum({ a: 10, b: 3, c: 9 });
```

TypeScript에서는 다음과 같이 합니다.

```typescript
function sum({ a, b, c }: { a: number; b: number; c: number }) {
  console.log(a + b + c);
}
```

`type`을 이용해 다음처럼 사용할 수 있습니다.

```typescript
// Same as prior example
type ABC = { a: number; b: number; c: number };
function sum({ a, b, c }: ABC) {
  console.log(a + b + c);
}
```


## 함수 할당 가능성

### 반환 유형 `void`

TypeScript에서는 함수가 무언가를 반환하는 것으로 `void`로 강제하지 안습니다. 그러므로 아래의 코드는 정상 코드가 됩니다.

```typescript
type voidFunc = () => void;
 
const f1: voidFunc = () => {
  return true;
};
 
const f2: voidFunc = () => true;
 
const f3: voidFunc = function () {
  return true;
};
```

그리고 반환 유형은 `void`로 유지됩니다.

```typescript
const v1 = f1();
 
const v2 = f2();
 
const v3 = f3();
```

다음의 코드도 `push()`가 숫자를 반환하고 `forEach()`의 매개변수 유형의 반환 값이 `void`이지만 마찬가지로 정상 코드입니다.

```typescript
src.forEach((el) => dst.push(el));
```


## 정리

오늘 함수에 대한 더 많은 상세한 내용을 살펴봤습니다. 다음 시간에는 핸드북의 객체 유형에 대해 살펴 보도록 하겠습니다.
