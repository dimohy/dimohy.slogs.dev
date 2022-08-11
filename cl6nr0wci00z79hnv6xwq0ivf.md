## TypeScript 배우기 - 9. typeof 타입 연산자

이번 시간에는 `typeof` 타입 연산자를 알아보겠습니다.

`tpyeof` 연산자는 이미 JavaScript에 있습니다. 다음의 TypeScript 코드를 JavaScript 코드로 변환한 것을 보시죠.

- TypeScript
  ```typescript
  // Prints "string"
  console.log(typeof "Hello world");
  ```
- JavaScript
  ```javascript
  "use strict";
  // Prints "string"
  console.log(typeof "Hello world");
  ```

하지만 TypeScript에서는 타입에 `typeof`를 쓸 수 있도록 허용합니다.

```typescript
let s = "hello";
let n: typeof s;  // n의 타입은 string
```

그런데 이게 어디서 유용할까요? TypeScript에서 제공하는 `ReturnType<Type>` 유틸리티 타입을 봅시다. ReturnType으로 함수의 반환 타입을 알 수 있습니다.

```typescript
type Predicate = (x: unknown) => boolean;
type K = ReturnType<Predicate>;  // K는 boolean 타입
```

하지만 값은 안됩니다. 값은 유형이 아니기 때문에 제네릭 인자로 올 수 없죠.

```typescript
function f() {
  return { x: 10, y: 3 };
}
type P = ReturnType<f>;
>                   ~  
> 'f' refers to a value, but is being used as a type here. Did you mean 'typeof f'?
```

대신 `typeof`를 사용해야 합니다.

```typescript
function f() {
  return { x: 10, y: 3 };
}
type P = ReturnType<typeof f>;
```


### 제한 사항

하지만 `typeof`는 식별자 또는 해당 속성에만 허용합니다.

```typescript
// Meant to use = ReturnType<typeof msgbox>
let shouldContinue: typeof msgbox("Are you sure you want to continue?");
```