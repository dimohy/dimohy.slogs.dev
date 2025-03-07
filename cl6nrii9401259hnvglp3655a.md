---
title: "TypeScript 배우기 - 10. 인덱싱 엑세스 타입"
datePublished: Wed Aug 10 2022 15:24:35 GMT+0000 (Coordinated Universal Time)
cuid: cl6nrii9401259hnvglp3655a
slug: typescript-10
tags: typescript

---

인덱싱된 엑세스 타입을 사용하면 다른 타입의 특정 속성을 조회해서 그 타입을 취할 수 있습니다.

```typescript
type Person = { age: number; name: string; alive: boolean };
type Age = Person["age"]; // Age의 타입은 number가 됨
```

여기서 인덱싱 타입 역시 타입이라는 점에서 유니온, `keyof`, 다른 타입을 사용할 수 있습니다.

```typescript
type Person = { age: number; name: string; alive: boolean };

type I1 = Person["age" | "name"]; // string | number

type I2 = Person[keyof Person]; // string | number | boolean

type AliveOrName = "alive" | "name";
type I3 = Person[AliveOrName]; // string | boolean
```

존재하지 않는 속성을 인덱싱하려고 하면 오류가 표시됩니다.

```typescript
type I1 = Person["alve"];
>                 ~~~~
> Property 'alve' does not exist on type 'Person'.
```

임의 타입으로 인덱싱하는 예는 `number` 타입을 사용해서 배열 요소의 타입을 가져오는 것입니다. 이것을 `typeof`와 결합하여 배열 리터럴의 요소 타입을 쉽게 가져올 수 있습니다.

```typescript
const MyArray = [
  { name: "Alice", age: 15 },
  { name: "Bob", age: 23 },
  { name: "Eve", age: 38 },
];

// number 인덱싱은 곧 배열 요소를 가리키므로 `typeof`에 의해 `{ name: string, age: number}` 타입이 됨
type Person = typeof MyArray[number];

// 그중에 "age" 속성의 `number`를 타입을 취함
type Age = typeof MyArray[number]["age"];

// 또는 이렇게
type Age2 = Person["age"];
```

하지만 인텍싱은 타입이어야 합니다. 즉 `const` 변수 참조는 오류를 발생합니다.

```typescript
const key = "age";
type Age = Person[key];
>                 ~~~
> Type 'key' cannot be used as an index type.
> 'key' refers to a value, but is being used as a type here. Did you mean 'typeof key'?
```

하지만 다음은 리터럴 타입이므로 정상 코드입니다.

```typescript
type key = "age";
type Age = Person[key]; // key는 리터럴 타입
```


