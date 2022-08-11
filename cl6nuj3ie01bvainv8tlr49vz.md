## TypeScript 배우기 - 12. 매핑된 타입

TypeScript는 마치 매크로 언어처럼 기존 타입을 기반으로 새로운 타입을 쉽게 만들 수 있습니다.

매핑된 타입은 사전에 선언되지 않은 속성 타입을 선언하는데 사용하는 인덱스 서명 구문을 기반으로 합니다.

아래는 인덱스 서명 구문으로 표현된 예시입니다.

```typescript
type OnlyBoolsAndHorses = {
  [key: string]: boolean | Horse;
};
 
const conforms: OnlyBoolsAndHorses = {
  del: true,
  rodney: false,
};
```

JavaScript에서 속성은 `a["propertyName"]`으로 표현될 수 있으므로 `[key: string]: boolean | Horse`로 표현할 수 있습니다. 그러므로 `conforms`의 `del` 및 `rodney`는 `boolean` 타입을 반환하므로 정상 코드가 됩니다.

매핑된 타입은 `in keyof`를 사용해서 제네릭 인자의 타입을 순회하여 동일한 속성을 가진 새로운 타입을 정의합니다.

```typescript
type OptionsFlags<Type> = {
  [Property in keyof Type]: boolean;
};
```

다음은 매핑된 타입의 예시입니다.

```typescript
type FeatureFlags = {
  darkMode: () => void;
  newUserProfile: () => void;
};
 
// FeatureFlags의 속성 (함수 포함)을 그대로 받아 반환형을 `boolean`으로 바꿔줌
// type FeatureOptions = {
//    darMode: boolean;
//    newUserProfile: boolean;
// }
type FeatureOptions = OptionsFlags<FeatureFlags>;
```


###  매핑 수식어

`readonly`와 `?` 앞에 `-` 또는 `+`의 접두사를 붙일 수 있습니다. (`+` 접두사는 생략 가능)

```typescript
// Removes 'readonly' attributes from a type's properties
type CreateMutable<Type> = {
  // `-readonly`에 의해 `readonly`가 제거됨
  -readonly [Property in keyof Type]: Type[Property];
};
 
type LockedAccount = {
  readonly id: string;
  readonly name: string;
};
 
// type UnlockedAccount = {
//     id: string;
//     name: string;
// }
type UnlockedAccount = CreateMutable<LockedAccount>;
```

```typescript
// Removes 'optional' attributes from a type's properties
type Concrete<Type> = {
  // `-?`에 의해 `?`가 제거됨
  [Property in keyof Type]-?: Type[Property];
};
 
type MaybeUser = {
  id: string;
  name?: string;
  age?: number;
};
 
// type USer = {
//    id: string;
//    name: string;
//    age: number;
// }
type User = Concrete<MaybeUser>;
```


## `as`로 키 재매핑

TypeScript 4.1 이상부터 `as` 절을 이용해서 매핑된 타입의 키를 다시 매핑할 수 있습니다.

```
type MappedTypeWithNewProperties<Type> = {
    [Properties in keyof Type as NewKeyType]: Type[Properties]
}
```

다음은 템플릿 리터럴 타입과 결합해서 이전 속성에서 새로운 속성 이름을 만드는 예제 코드입니다.

```typescript
type Getters<Type> = {
    [Property in keyof Type as `get${Capitalize<string & Property>}`]: () => Type[Property]
};
 
interface Person {
    name: string;
    age: number;
    location: string;
}
 
// type LazyPerson = {
//     getName: () => string;
//     getAge: () => number;
//     getLocation: () => string;
// }
type LazyPerson = Getters<Person>;
```

조건부 타입을 통해 `never`를 생성해서 키를 필터릴 할 수도 있습니다.

```typescript
// Remove the 'kind' property
type RemoveKindField<Type> = {
    [Property in keyof Type as Exclude<Property, "kind">]: Type[Property]
};
 
interface Circle {
    kind: "circle";
    radius: number;
}
 
// type KindlessCircle = {
//     radius: number;
// }
type KindlessCircle = RemoveKindField<Circle>;
```

`string | number | symbol` 유니온 타입 뿐만 아니라 다른 유니온 타입에 대해 임의의 매핑을 할 수 있습니다.

```typescript
type EventConfig<Events extends { kind: string }> = {
    [E in Events as E["kind"]]: (event: E) => void;
}
 
type SquareEvent = { kind: "square", x: number, y: number };
type CircleEvent = { kind: "circle", radius: number };
 
// type Config = {
//      square: (event: SquareEvent) => void;
//      circle: (event: CircleEvent) => void;
// }
type Config = EventConfig<SquareEvent | CircleEvent>
```

대단하군요. 매크로 언어와 같은 느낌입니다.


### 추가 탐색

매핑된 타입은 다른 조작 섹션의 기능과 잘 결합하여 작동합니다. 다음은 `pii` 속성이 리터럴 `true`로 설정되어 있는지의 유무에 따라 `true` 또는 `false`를 반환하는 조건부 타입을 사용하는 매핑된 타입의 예시입니다.

```typescript
type ExtractPII<Type> = {
// Type의 키를 순회하며
// `{ pii: true }`가 있는 경우 true, 없는 경우 false 타입이 됨
  [Property in keyof Type]: Type[Property] extends { pii: true } ? true : false;
};
 
type DBFields = {
  id: { format: "incrementing" };
  name: { type: string; pii: true };
};
 
// type ObjectsNeedingGDPRDeletion = {
//     id: false;
//     name: true;
// }
type ObjectsNeedingGDPRDeletion = ExtractPII<DBFields>;
```
