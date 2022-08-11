## TypeScript 배우기 - 8. keyof 타입 연산자

오늘은 TypeScript의 `keyof` 타입 연산자에 대해 알아볼 것입니다. 어렵지 않으므로 빠르게 이해해 봅시다!

```typescript
// 객체 타입 Point 생성
type Point = { x: number, y: number };
// Point의 속성에 의해 "x" | "y" 문자열 리터럴 타입이 됨
type P = keyof Point;

const a:P = "x";
const b:P = "y";
const c:P = "m";
>            ~ 
> Type '"m"' is not assignable to type 'keyof Point'.
```

`keyof` 연산자는 객체 타입의  속성 (그리고 함수 이름)을 유니온 타입으로 조합하여 생성합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1660141532573/Nz7_1ifd31.png align="left")

이 기능은 편집기의 기능을 사용할 수 있으므로 강력합니다. 

타입에 `string` 또는 `number` 인덱스 서명이 있는 경우 `keyof`는 이름 대신 해당 타입을 반환합니다.

```typescript
type Arrayish = { [n: number]: unknown };
type A = keyof Arrayish;  // number
    
type Mapish = { [k: string]: boolean };
type M = keyof Mapish;  // string | number
```

첫번째 `Arrayish`는 `number` 인덱서이므로 `keyof`는 `number`를 반환합니다.
두번째 `Mapish`는 `string` 인덱서이지만 JavaScript에서 `obj[0]`과 `obj["0"]`는 동일하게 동작하므로 `string`이 아닌 `string | number` 유니온 타입이 됩니다.

`keyof` 타입은 매핑된 타입에서 유용하게 사용됩니다.

```typescript
// 타입의 프로퍼티에서 'readonly' 속성을 제거합니다
type CreateMutable<Type> = {
  -readonly [Property in keyof Type]: Type[Property];
};
 
type LockedAccount = {
  readonly id: string;
  readonly name: string;
};
 
type UnlockedAccount = CreateMutable<LockedAccount>;
```

자세한 것은 매핑된 타입에서 나누겠습니다.
