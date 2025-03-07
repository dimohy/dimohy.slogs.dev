---
title: "TypeScript 배우기 - 11. 조건부 타입"
datePublished: Wed Aug 10 2022 16:20:38 GMT+0000 (Coordinated Universal Time)
cuid: cl6ntil7v01bx9hnv0rr57x5m
slug: typescript-11
tags: typescript

---

조건부 타입은 입력 타입과 출력 타입간의 관계를 설명하는데 도움이 됩니다. 다음은 핸드북에 나온 예제 입니다.

```typescript
interface Animal {
  live(): void;
}
interface Dog extends Animal {
  woof(): void;
}

// Dog는 Animal 타입으로 할당할 수 있으므로 참. 그러므로 Example1는 number가 됨
type Example1 = Dog extends Animal ? number : string;

// RegExp는 Animal 타입으로 할당할 수 없으므로 거짓. 그러므로 Example2는 string이 됨
type Example2 = RegExp extends Animal ? number : string;
```

조건부 타입은 다음과 같이 표현됩니다.
```
SomeType extends OtherType ? TrueType : FalseType;
```
`SomeType`이 `OtherType`으로 할당할 수 있으면 `TrueType`이 되고 그렇지 않으면 `FalseType`이 됩니다.

위의 코드는 너무나 간단해서 조건부 타입이 정말로 유용하게 쓰일 수 있는지 알기가 어려운데요, 제네릭과 함께 사용될 때 그 힘이 발휘됩니다. 다음의 코드를 보시죠.

```typescript
interface IdLabel {
  id: number /* some fields */;
}
interface NameLabel {
  name: string /* other fields */;
}
 
function createLabel(id: number): IdLabel;
function createLabel(name: string): NameLabel;
function createLabel(nameOrId: string | number): IdLabel | NameLabel;
function createLabel(nameOrId: string | number): IdLabel | NameLabel {
  throw "unimplemented";
}
```

앞서 학습했던 오버로드 함수군요. 장황합니다. 이것을 제네릭 함수와 함께 조건부 타입으로 다음과 같이 표현할 수 있습니다.

```typescript
type NameOrId<T extends number | string> = T extends number
  ? IdLabel
  : NameLabel;
```

오, 제네릭 인자의 타입에 따라 `NameOrId<T>`의 타입이 `IdLabel` 또는 `NameLabel`로 결정됩니다.

이제 위의 장황한 오버로드 함수를 다음의 간결한 제네릭 함수로 표현할 수 있습니다.

```typescript
function createLabel<T extends number | string>(idOrName: T): NameOrId<T> {
  throw "unimplemented";
}

// 인자 타입이 string이므로 반환 타입은 `NameLabel`이 됨
let a = createLabel("typescript");

// 인자 타입이 number이므로 반환 타입은 `IdLabel`이 됨
let b = createLabel(2.8);

// `random()`에 의해 인자 타입은 `string | number`가 되므로 반환 타입 역시 `NameLabel | IdLabel`이 됨
let c = createLabel(Math.random() ? "hello" : 42);
```


### 조건부 타입의 제약 조건

다음의 코드를 확인해 봅시다.

```typescript
type MessageOf<T> = T["message"];
> Type '"message"' cannot be used to index type 'T'.
```

`type MessageOf<T>`는 `T`의 `message` 속성 타입을 취하고 있습니다. 하지만 제네릭 타입 `T`는 어떠한 `extends`도 가지지 않으므로 `message`를 알 수 없습니다. 그러므로 오류입니다.

이것을 다음처럼 표현하면 TypeScript는 더이상 오류를 표시하지 않습니다.

```typescript
// T는 message 속성이 있고 그 타입은 알 수 없다는 것을 표현
type MessageOf<T extends { message: unknown }> = T["message"];
 
interface Email {
  message: string;
}
 
// EmailMessageContents는 Email에 `message` 속성이 있으므로 그 타입인 `string`이 됨
type EmailMessageContents = MessageOf<Email>;
```

하지만 해당 속성이 없을 경우 기본 값으로 `naver` 타입이 되게 할 수는 없을까요? 이때 조건부 타입을 사용할 수 있습니다.

```typescript
// message 속성이 있을경우 참이므로 `T["message"]` 타입이 됨. 없을 경우 `never` 타입이 됨
type MessageOf<T> = T extends { message: unknown } ? T["message"] : never;
 
interface Email {
  message: string;
}
 
interface Dog {
  bark(): void;
}

// string
type EmailMessageContents = MessageOf<Email>;

// never
type DogMessageContents = MessageOf<Dog>;
```

다름은 다른 예시 입니다.

```typescript
type Flatten<T> = T extends any[] ? T[number] : T;

// Extracts out the element type.
type Str = Flatten<string[]>;

// Leaves the type alone.
type Num = Flatten<number>;
```

`Flatten<string[]>`은 `any[]`로 할당할 수 있으므로 참이 되어 `T[number]` 타입인 `string`이 됩니다.
`Flatten<number>`의 경우 `any[]`로 할당할 수 없으므로 거짓이 되어 `T` 타입인 `number`가 됩니다.


### 조건부 유형 내 추론

`Flatten<string[]>` 경우 처럼 타입을 추출하는 방식을 보았습니다. 이런 방법처럼 `infer` 키워드로 관심 있는 타입을 얻을 수 있습니다. 다음의 예시는 `ReturnType<T>`를 `infer`를 이용해서 구현한 것입니다.

```typescript
// `infer Return`는 타입이 되고 그 타입은 `Return`으로 쓸 수 있음
// 즉, 함수의 반환형을 `Return`으로 취할 수 있게 됨
type GetReturnType<Type> = Type extends (...args: never[]) => infer Return
  ? Return
  : never;

// number
type Num = GetReturnType<() => number>;

// string
type Str = GetReturnType<(x: string) => string>;

// boolean[]
type Bools = GetReturnType<(a: boolean, b: boolean) => boolean[]>
```

오버로드 함수의 다중 호출 서명의 경우 가장 포괄적인 서명을 선택합니다. 

```typescript
declare function stringOrNum(x: string): number;
declare function stringOrNum(x: number): string;
declare function stringOrNum(x: string | number): string | number;
 
// string | number
type T1 = ReturnType<typeof stringOrNum>;
```


## 분배 조건부 타입

다음의 경우를 살펴봅시다. 타입을 `타입[]`으로 확장하는 조건부 타입입니다.

```typescript
type ToArray<Type> = Type extends any ? Type[] : never;
```

유니온 타입을 사용하면 어떻게 될까요?

```typescript
type ToArray<Type> = Type extends any ? Type[] : never;
 
// string[] | number[]
type StrArrOrNumArr = ToArray<string | number>;
```

`string | number`는 `ToArray<string> | ToArray<number>`가 되어 결국에 `string[] | number[]`이 됩니다.

이 동작은 기본 동작입니다. 만약에 이 동작 대신 `(string | number)[]`이 되고자 한다면 다음처럼 가능 합니다.

```typescript
type ToArrayNonDist<Type> = [Type] extends [any] ? Type[] : never;

// (string | number)[]
type StrArrOrNumArr = ToArrayNonDist<string | number>;
```
