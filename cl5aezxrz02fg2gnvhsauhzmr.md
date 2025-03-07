---
title: "TypeScript 배우기 - 4. 좁히기"
datePublished: Thu Jul 07 2022 02:33:30 GMT+0000 (Coordinated Universal Time)
cuid: cl5aezxrz02fg2gnvhsauhzmr
slug: typescript-4
tags: typescript

---

좁히기(Narrowing)는 유니온 유형 등 공통의 연산 또는 호출을 할 수 없을 때 `typeof`등을 이용해 대상을 좁히는(타겟팅 하는) 것을 말합니다. 다음의 예를 살펴보죠.

```typescript
function padLeft(padding: number | string, input: string): string {
  throw new Error("Not implemented yet!");
}
```

아직까지는 미구현 상태이므로  괜찮습니다. `padLeft`는 `padding`만큼 입력된 문자열을 왼쪽 패딩하는 함수가 될 것입니다. 아래의 코드처럼 구현할 수 있겠죠.

```typescript
function padLeft(padding: number | string, input: string) {
  return " ".repeat(padding) + input;
}
```

> Argument of type 'string | number' is not assignable to parameter of type 'number'.
  Type 'string' is not assignable to type 'number'.

하지만 오류가 발생했습니다. string의 `repeat()` 함수의 인자는 `number`여야 하기 때문인데요, 이를 다음의 코드처럼 변경할 수 있습니다.

```typescript
function padLeft(padding: number | string, input: string) {
  if (typeof padding === "number") {
    return " ".repeat(padding) + input;
  }
  return padding + input;
}
```

`padding` 인자 유형이 `number`일 경우 `" ".repeat(padding)`을 통해 입력된 값 `input` 앞에 공백을 넣어줍니다. `number`가 아닐 경우 `string`이므로 `padding + input`을 통해 앞에 공백을 넣어주게 되겠죠.

이렇게 선언된 유형보다 좀 더 구체적인 유형으로 축소하는 것을 좁히기(Narrowing)라 합니다.

흥미로운 점은 런타임 시점이 아니라 컴파일 시점에 TypeScript가 `typeof`에 의해 정확히 타입을 구분하고 있다는 점입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657078206252/WFSfstSR3.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657078223315/xz7NlBTfl.png align="left")

결과적으로 TypeScript는 `any` 유형이 아닌 이상 코딩 시점에서 정확히 타입을 분별해서 타입에 맞는 코드를 작성하도록 유도하게 됩니다.

`typeof`와 같이 TypeScript에서 좁히는(narrowing) 여러가지 구성을 살펴봅시다.


## `typeof` 유형 가드

위의 코드에서 살펴본 `typeof`는 런타임 때 기본 유형에 대한 문자열을 반환합니다. 기본 유형은 다음과 같습니다.

- "string"
- "number"
- "bigint"
- "boolean"
- "symbol"
- "undefined"
- "object"
- "function"

`typeof`를 이용한 방법을 `typeof` 유형 가드라 합니다. 단점도 있습니다. 아래 코드 처럼 `문자열 또는 문자열 배열` 유니언 유형을 인자로 받을 때 다음의 코드는 문제가 있습니다.

```typescript
function printAll(strs: string | string[] | null) {
  if (typeof strs === "object") {
    for (const s of strs) {
      console.log(s);
    }
  } else if (typeof strs === "string") {
    console.log(strs);
  } else {
    // do nothing
  }
}
```

typeof에서 `null`이 가능한 문자열 배열 유형은 없으므로  인자 값이 `null`일 경우 `typeof`에서 "object"로 반환할 것이고 `for`문에서 오동작을 하게 될 것이기 때문입니다. 다행히 TypeScript에서는 컴파일 시점에서 이를 감지해 오류를 발생해줍니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657079037208/TBB3l-AIM.png align="left")
> Object is possibly 'null'.


## 참/거짓 좁히기

JavaScript는 조건부(`&&` s, `||` s, `if`문, 부울 부정(`!`)등)에 모든 표현식을 사용할 수 있습니다. 예를 들어 if 문에 항상 `boolean` 유형이 필요하지는 않습니다.

```typescript
function getUsersOnlineMessage(numUsersOnline: number) {
  if (numUsersOnline) {
    return `There are ${numUsersOnline} online now!`;
  }
  return "Nobody's here. :(";
}
```

JavaScript에서는 각 데이터 유형에서 다음의 값을 `false`로 해석하고 아닌 경우 `true`로 해석하게 됩니다.

- 0
- NaN
- ""
- 0n
- null
- undefined

다음의 코드처럼 강제로 값을 변환할 수도 있습니다.

```typescript
// both of these result in 'true'
Boolean("hello"); // type: boolean, value: true
!!"world"; // type: true,    value: true
```

첫번째 줄은 `boolean`으로 변환하고 값을 boolean `true`로 평가하는 반면 두번째 줄은 리터럴 `true`로 변환하고 값을 `true`로 평가한다는 차이점이 있습니다.

JavaScript의 조건부 특징을 이용해 위의 `stars`에서 발생한 `null` 관련 오류는 아래의 코드로 수정할 수 있습니다.

```typescript
function printAll(strs: string | string[] | null) {
  if (strs && typeof strs === "object") {
    for (const s of strs) {
      console.log(s);
    }
  } else if (typeof strs === "string") {
    console.log(strs);
  }
}
```

하지만 다음의 코드처럼 미묘한 문제점이 있습니다.

```typescript
function printAll(strs: string | string[] | null) {
  // !!!!!!!!!!!!!!!!
  //  DON'T DO THIS!
  //   KEEP READING
  // !!!!!!!!!!!!!!!!
  if (strs) {
    if (typeof strs === "object") {
      for (const s of strs) {
        console.log(s);
      }
    } else if (typeof strs === "string") {
      console.log(strs);
    }
  }
}
```

이 코드는 TypeScript에서 오류가 발생하지 않지만 의도하지 않는 동작을 발생시킬 수 있습니다. 왜냐하면 빈 문자열 `""`이 `if`문에서 `false`로 평가되어 아예 출력이 안될 것이기 때문입니다.

다음은 `!` 부정 분기를 이용해 진실성 좁히기의 코드 입니다.

```typescript
function multiplyAll(
  values: number[] | undefined,
  factor: number
): number[] | undefined {
  if (!values) {
    return values;
  } else {
    return values.map((x) => x * factor);
  }
}
```


## 같음 비교 좁히기

TypeScript는 `switch`문이나 `===`, `!==`, `==`, `!=`로 유형을 좁힐 수 있습니다. 다음은 예시 코드입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657080360817/v5yisc7_z.png align="left")

x가 y와 `===`일 때 x와 y가 같은 유형은 `string`뿐이므로 첫번째 조건의 x, y 데이터 유형은 `string`이 됩니다. `else`의 경우 x, y의 유형이 다른 경우이므로 x `string`, y `boolean` 또는 x `number`, y `string | boolean`이 됩니다.

위의 `printAll`에서 `if (strs)`의 오동작을 다음의 코드처럼 수정할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657080823237/S1meVYZI-.png align="left")

JavaScript의 `==`와 `!=`의 느슨한 동등성 검사 역시 올바르게 좁혀집니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657080840784/oc2ZK6c9B.png align="left")


## `in` 좁히기

`in` 연산자는 객체에 해당 속성이 있는지 확인하는 연산자입니다. `in` 연산 또한 잘 좁혀집니다.

```typescript
type Fish = { swim: () => void };
type Bird = { fly: () => void };
 
function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    return animal.swim();
  }
 
  return animal.fly();
}
```

위의 경우 `Fish`에만 `swim` 속성이 있으므로 `in`에 의해 대상을 좁힐 수 있습니다.

만약 동일한 속성이 중복될 경우 유니언 유형으로 좁혀집니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657081156176/F43l3eBWz.png align="left")


## instanceof 좁히기

`instanceof`는 객체가 어떤 객체 유형인지를 런타임에서 식별하는 연산자이므로 대상을 좁힐 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657081237211/SkDQ2LW5F.png align="left")


## 할당

TypeScript는 변수에 할당 할 때 오른쪽의 유형을 보고 왼쪽의 유형을 적절하게 좁힙니다.

```typescript
let x = Math.random() < 0.5 ? 10 : "hello world!";
```

x는 랜덤하게 숫자 `10`이 되거나 문자열 `hello world!`이 되므로 x의 유형은 `number | string` 입니다.

```typescript
x = 1;
console.log(x);
```

그런데 x에 `1`이 대입 했는데 x의 유형은 `number | string`으로 오류 없는 코드입니다.

```typescript
x = "goodbye!";
 console.log(x);
```

x에 `goodbye!`을 대입 했는데 오류 없는 코드입니다.

하지만 x에 `true`를 대입하게 되면 오류가 발생합니다. x의 유형이 `boolean`이 아니기 때문입니다.

```typescript
console.log(x);
x = true;
```
> Type 'boolean' is not assignable to type 'string | number'.


## 제어 흐름 분석

앞전에 살펴본 아래의 코드는,

```typescript
function padLeft(padding: number | string, input: string) {
  if (typeof padding === "number") {
    return " ".repeat(padding) + input;
  }
  return padding + input;
}
```

조건에 의해서 반환 블럭의 `padding`의 유형이 `number`이거나 `string`으로 유형을 잘 좁힙니다. 이렇게 도달 가능성을 기반으로 하는 코드 분석을 `제어 흐름 분석`이라고 합니다.
TypeScript에서는 이러한 흐름 분석을 통해 유형 보호 및 할당이 발생할 때 유형을 좁힙니다. 변수를 분석할 때 계속해서 제어 흐름을 통해 분리되거나 다시 병합될 수 있으며 해당 변수는 각 지점에서 다른 유형을 갖는 것으로 관찰 될 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657158027529/YOi7K6Ewu.png align="left")


## 유형 기술 사용

`is`로 유형을 기술하는 것을 통해 좁히기를 할 수 있습니다. 다음의 예를 보죠.

```typescript
function isFish(pet: Fish | Bird): pet is Fish {
  return (pet as Fish).swim !== undefined;
}
```

`pet is Fish`를 통해 `pet`이 Fish일 경우 유형을 `Fish`로 좁히게 됩니다. 다음의 코드로 그 동작을 확인할 수 있습니다.

```typescript
// Both calls to 'swim' and 'fly' are now okay.
let pet = getSmallPet();
 
if (isFish(pet)) {
  pet.swim();
} else {
  pet.fly();
}
```

TypeScript는 `if`문에서 `isFish()`가 참일 경우 `pet`의 유형이 `Fish`라는 것을 알 수 있으며 `else`의 경우 `Fish`가 아닌 다른 유형이 `Bird`라는 것도 알고 있습니다. 이렇게 적절히 유형 기술을 통해 좁히기가 가능합니다.

다음의 코드를 통해 `isFish`가 필터로 유형 좁히기가 가능하다는 것도 살펴볼 수 있습니다.

```typescript
const zoo: (Fish | Bird)[] = [getSmallPet(), getSmallPet(), getSmallPet()];
const underWater1: Fish[] = zoo.filter(isFish);
// or, equivalently
const underWater2: Fish[] = zoo.filter(isFish) as Fish[];
 
// The predicate may need repeating for more complex examples
const underWater3: Fish[] = zoo.filter((pet): pet is Fish => {
  if (pet.name === "sharkey") return false;
  return isFish(pet);
});
```

`zoo`는 `getSmallPet()`의 반환 유형인 `Fish | Bird`가 되는데 `zoo.filter()`를 통해 주어진 `isFish`로 `Fish`유형으로 반환 받을 수 있다는 것을 알 수 있습니다. 또한 `isFish`를 사용하지 않고 람다를 썼을 때 `is Fish`라고 유형 기술 했을 때도 동일하게 `Fish` 유형으로 좁힐 수 있습니다.


## 유니언 식별

원과 사각형과 같은 모양을 표현한다고 합시다.

```typescript
interface Shape {
  kind: "circle" | "square";
  radius?: number;
  sideLength?: number;
}
```

원은 반지름(`radius`)이 필요하고 사각형은 측면 길이(`sideLength`)가 필요합니다.  그리고 원 또는 사각형인지를 구분하는 속성으로 `kind`를 사용했는데 이 유형이 `string`이 아니라 리터럴 유니언일 경우 문자열로 인해 발생하는 오타를 막을 수 있습니다. 예를 들어

```typescript
function handleShape(shape: Shape) {
  // oops!
  if (shape.kind === "rect") {
  // ...
  }
}
```
> This condition will always return 'false' since the types '"circle" | "square"' and '"rect"' have no overlap.

여기서 "rect"는 `kind`의 유니언 멤버가 아니므로 오류가 발생합니다.

면적을 구하는 `getArea()` 함수를 만들어봅시다. 이는 `kind`에 따라 계산 방법이 달라져야 하는데요, 다음의 코드는

```typescript
function getArea(shape: Shape) {
  return Math.PI * shape.radius ** 2;
}
```
> Object is possibly 'undefined'.

radius가 `radius?`로 정의 되어 있으므로 `undefined`일 수 있다는 오류 입니다. 만약 `kind`로 분기했을 때 어떻게 달라질까요?

```typescript
function getArea(shape: Shape) {
  if (shape.kind === "circle") {
    return Math.PI * shape.radius ** 2;
  }
}
```
> Object is possibly 'undefined'.

당연히 여전히 동일한 오류가 발생합니다. 이를 막기 위해 `!`를 줘서 `radius`가 `undefined`가 아님을 알려주면 해당 오류는 사라집니다.

```typescript
function getArea(shape: Shape) {
  if (shape.kind === "circle") {
    return Math.PI * shape.radius! ** 2;
  }
}
```

하지만 이것은 이상적이진 않습니다. 왜냐하면 `Circle`에서는 필요 없는 속성 `sideLength`가 접근이 되고 `Square`에서는 필요없는 속성 `radius`가 접근될 것이기 때문인데요. 다음의 방법으로 이를 해결할 수 있습니다.

```typescript
interface Circle {
  kind: "circle";
  radius: number;
}
 
interface Square {
  kind: "square";
  sideLength: number;
}
 
type Shape = Circle | Square;
```

이야. 아름답습니다. 각각의 인터페이스는 독립적이면서 `type Shape`에 의해 잘 결합되었습니다. 이제 오류 정보는 좀 더 정확히 표현됩니다.

```typescript
function getArea(shape: Shape) {
  return Math.PI * shape.radius ** 2;
}
```
> Property 'radius' does not exist on type 'Shape'.

이제 `kind` 속성에 의한 좁히기로 다음 처럼 정확히 `radius`에 접근하는 안전한 코드를 작성할 수 있게 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657160428360/s-Yx7XeOy.png align="left")

비단 `if`문 뿐만 아니라 `switch`문에서도 동일하게 좁히기가 동작합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657160477323/a-atswG_A.png align="left")

동적 언어인 JavaScript에 정적 타입 검사를 오묘하게 잘 결합했네요!


## `never` 유형

좁히기가 진행되면서 더이상 아무런 유형이 없을 경우를 TypeScript는 `never` 유형으로 표현해서 존재해서는 안되는 상태로 나타냅니다.

> 이는 동적 언어인 JavaScript가 필연적으로 가질 수 있는 상황을 해결하기 위해 나왔습니다. 가령, `number | string`의 유니언 유형의 경우 `number`, `string`이외의 `never` 유형이 나올 일이 없겠죠. 하지만 이러한 타입 체크는 TypeScript의 정적 타입 체크의 기술인 것이지 JavaScript에서는 얼마든지 나올 수 있습니다.

```typescript
type Shape = Circle | Square;
 
function getArea(shape: Shape) {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.sideLength ** 2;
    default:
      const _exhaustiveCheck: never = shape;
      return _exhaustiveCheck;
  }
}
```

만약 Shape에 새로운 `Triangle` 유니언 멤버가 추가되었다면 다음처럼 `never` 유형으로 인해 오류가 발생하며 이를 통해 누락할 수 있는 처리를 할 수 있습니다.

```typescript
interface Triangle {
  kind: "triangle";
  sideLength: number;
}
 
type Shape = Circle | Square | Triangle;
 
function getArea(shape: Shape) {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.sideLength ** 2;
    default:
      const _exhaustiveCheck: never = shape;
            ~~~~~~~~~~~~~~~~
> Type 'Triangle' is not assignable to type 'never'.
      return _exhaustiveCheck;
  }
}
```

## 정리
TypeScript에서 다양한 상황에서 유형을 좁힐 수 있다는 것을 알게 되었습니다. 유형 좁히기로 정확히 코딩 하는 것은 다양한 예외 상황과 버그를 피할 수 있는 훌륭한 장치입니다. 또한 TypeScript에서 제공하는 편집 도구 기능을 잘 사용할 수 있게 하겠죠.

다음 시간에는 핸드북의 `More on Functions`를 살펴보도록 하겠습니다.


