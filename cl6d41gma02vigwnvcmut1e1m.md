## TypeScript 배우기 - 7. 제네릭

> 본 글은 TypeScript 핸드북 [Type Manipulation / Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)의 내용을 정리한 글입니다.

재사용 가능한 코드를 만들기 위해 여러 언어에서 제네릭을 사용합니다. 제네릭을 사용하면 `단일 유형`이 아닌 `다양한 유형`에서 작동할 수 있는 코드를 만들 수 있습니다. 이를 통해 코드를 좀 더 효율적으로 작성할 수 있게 됩니다.


## 기초

먼저 제네릭을 이해기 위해 TypeScript에서 `identity()`라는 인자 값을 그대로 반환하는 함수를  살펴볼 것입니다.

```typescript
function identity(arg: number): number {
  return arg;
}
```

`number` 유형의 인자를 받아 그대로 반환합니다. 이 함수는 쉽고 어떻게 동작할지도 바로 알 수 있습니다. 하지만 이 함수는 `number` 유형만 처리가 가능합니다. 다른 유형도 처리 가능하도록 `any`를 써볼 수 도 있습니다.

```typescript
function identity(arg: any): any {
  return arg;
}
```

이제 JavaScript의 기본 특성(`any`)으로 인해 모든 유형의 인자를 받아 반환하는 함수가 되었습니다. 하지만 TypeScript는 유형 추적이 장점이니 만큼 `any` 유형을 지양해야 합니다. 이를 제네릭으로 개선할 수 있습니다.

> 제네릭은 `일반화`의 개념으로 이해하면 됩니다. 

```typescript
function identity<Type>(arg: Type): Type {
  return arg;
}
```

이제 `identity()` 함수가 받는 유형과 반환하는 유형이 실제 유형이 아니라 `Type`이라는  `제네릭 유형`를 사용하게 되었습니다. 이 함수를 실제 사용하기 전에는 단지 `Type`이 될 수 있는 모든 유형이 후보가 됩니다.

이 후보는 아래의 코드에서 `Type`이 `string`으로 명시되면서 `Type` --> `string`이 됩니다.

```typescript
let output = identity<string>("myString");
```

위의 코드에 의해 `identity()` 함수는 다음의 코드처럼 동작하게 됩니다.

```typescript
function identity(arg: string): string {
  return arg;
}
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1659498279595/gtt6jE9bp.png align="left")

> 다른 정적 언어와 다르게 TypeScript는 JavaScript 코드를 생성하고 유형 표현은 사라지므로 실제로는 `identity()` 함수에 아무런 변화가 없습니다. 이와는 다르게 Java나 C#과 같은 정적 언어는 컴파일 수행중 함수가 `사용`되는 지점에서 `Type` --> `string`으로 변환된 `identity()` 함수를 생성합니다.

`제네릭 유형`를 유추할 수 있는 경우 `제네릭 유형` 표현을 생략할 수 있습니다. `identity()` 함수의 경우 함수 인자 유형에 의해 반환 유형이 유추 되므로 생략할 수 있습니다.

```typescript
let output = identity("myString");
```


## 제네릭 유형 변수 동작

제네릭 유형(여기서는 `Type`)의 자리에 결국에 실제 유형이 사용될 터이지만 제네릭을 사용한 코드에서는 알 수 없으므로 마치 `any` 유형 처럼 동작합니다.

```typescript
function loggingIdentity<Type>(arg: Type): Type {
  console.log(arg.length);
              ~~~~~~~~~~
> Property 'length' does not exist on type 'Type'.
  return arg;
}
```

제네릭 유형 `Type`은 아직 유형이 특정되지 않았으므로 `length`라는 속성이 있는지 알 수 없습니다. 그러므로 이 오류는 정상적이고 안전한 코드를 만들 수 있도록 하는 유용한 오류입니다. 우리는 TypeScript에게 좀 힌트를 알려줘야 합니다. 다음의 코드를 보시죠.

```typescript
function loggingIdentity<Type>(arg: Type[]): Type[] {
  console.log(arg.length);
  return arg;
}
```

이제 `arg`는 제네릭 유형 `Type`인 배열을 의미하게 되었습니다. 배열이므로 `length` 속성에 접근할 수 있습니다. 이제 오류가 없어졌고 정상적인 코드라고 TypeScript는 인정합니다.

`loggingIdentity()` 함수는 이제 어떠한 유형이든 상관없이 배열 형태일 경우 정확히 그 배열 길이를 잘 출력하는 함수가 되었습니다.

`Type[]`은 `Array<Type>`과 같으므로 다음처럼 코드를 표현할 수 도 있습니다.

```typescript
function loggingIdentity<Type>(arg: Array<Type>): Array<Type> {
  console.log(arg.length); // Array has a .length, so no more error
  return arg;
}
```


## 제네릭 유형

제네릭 함수의 유형은 함수 선언과 유사하게 유형 매개변수가 먼저 나열되는 비제네릭 함수의 유형과 유사합니다.

```typescript
function identity<Type>(arg: Type): Type {
  return arg;
}
 
let myIdentity: <Type>(arg: Type) => Type = identity;
```

`<Type>(arg: Type) => Type` 형태를 `identity()`가 가지므로 정상적인 코드입니다. 사용은 다음처럼 할 수 있습니다.

```typescript
let value = myIdentity<number>(5);
```

유형 변수의 수와 사용되는 방식이 일치하면 제네릭 유형 매개변수 (여기서는 `Type`) 이름을 동일하게 맞출 필요가 없습니다. 즉, 다음의 코드도 정상 코드입니다.

```typescript
function identity<Type>(arg: Type): Type {
  return arg;
}
 
let myIdentity: <Input>(arg: Input) => Input = identity;
```

TypeScript에서는 객체 리터럴 유형의 호출 서명으로도 작성할 수 있게 허용합니다.

```typescript
function identity<Type>(arg: Type): Type {
  return arg;
}
 
let myIdentity: { <Type>(arg: Type): Type } = identity;
```

그러므로 인터페이스로도 이를 표현할 수 있습니다.

```typescript
interface GenericIdentityFn {
  <Type>(arg: Type): Type;
}
 
function identity<Type>(arg: Type): Type {
  return arg;
}
 
let myIdentity: GenericIdentityFn = identity;
```

제네릭 유형 대신 실제 유형을 적용하고 싶을 수 도 있습니다.

```typescript
interface GenericIdentityFn<Type> {
  (arg: Type): Type;
}
 
function identity<Type>(arg: Type): Type {
  return arg;
}
 
let myIdentity: GenericIdentityFn<number> = identity;
```

`myIdentity`은 인터페이스에 의해 `Type` 제네릭 유형이 `number` 유형이 되었고 `identity()`함수의 `Type` 제네릭 유형에 `number`가 들어갈 수 있으므로 이 코드 역시 올바른 코드입니다.


## 제네릭 클래스

제네릭 함수와 마찬가지로 클래스에도 제네릭을 사용할 수 있습니다.

```typescript
class GenericNumber<NumType> {
  zeroValue: NumType;
  add: (x: NumType, y: NumType) => NumType;
}
 
let myGenericNumber = new GenericNumber<number>();
myGenericNumber.zeroValue = 0;
myGenericNumber.add = function (x, y) {
  return x + y;
};
```

`NumType` 제네릭 유형에는 `number`뿐만 아니라 `string`도 가능하므로 `string`으로 클래스를 사용할 수도 있습니다.

```typescript
let stringNumeric = new GenericNumber<string>();
stringNumeric.zeroValue = "";
stringNumeric.add = function (x, y) {
  return x + y;
};
 
console.log(stringNumeric.add(stringNumeric.zeroValue, "test"));
```


## 제네릭 제약 조건

앞의 코드에서 `length` 속성을 사용하고자 했지만 제네릭 유형을 특정하지 않는 이상 불가능 하다는 것을 알 수 있었습니다.

```typescript
function loggingIdentity<Type>(arg: Type): Type {
  console.log(arg.length);
              ~~~~~~~~~~
> Property 'length' does not exist on type 'Type'.
  return arg;
}
```

하지만 `extends` 키워드를 사용하면 이제 가능해 집니다.

```typescript
interface Lengthwise {
  length: number;
}
 
function loggingIdentity<Type extends Lengthwise>(arg: Type): Type {
  console.log(arg.length); // Now we know it has a .length property, so no more error
  return arg;
}
```

이를 제네릭 제약 조건이라고 하며 `Type` 제네릭 유형은 이제 반드시 `length` 속성이 있는 유형만 허용하게 됩니다.

```typescript
loggingIdentity(3);
                ~
> Argument of type 'number' is not assignable to parameter of type 'Lengthwise'.
```

숫자 3은 `number` 유형이고 `length`을 가지고 있지 않으므로 오류가 발생합니다. 이 오류는 `loggingIdentity()` 함수가 정상적으로 수행하기 위해 필요한 오류가 됩니다.

하지만 `length` 속성을 제공하는 모든 유형은 `loggingIdentity()` 함수를 사용할 수 있습니다.

```typescript
loggingIdentity({ length: 10, value: 3 });
```


## 제네릭 제약 조건에서 유형 매개변수 사용

`keyof` 등의 키워드를 제약 조건에 사용하면 유형 검사를 좀 더 강화할 수 있습니다.

```typescript
function getProperty<Type, Key extends keyof Type>(obj: Type, key: Key) {
  return obj[key];
}
 
let x = { a: 1, b: 2, c: 3, d: 4 };
 
getProperty(x, "a");
getProperty(x, "m");
                ~ 
> Argument of type '"m"' is not assignable to parameter of type '"a" | "b" | "c" | "d"'.
```

`getProperty()` 함수는 인자의 속성을 반환하는 함수입니다. `obj[key]`를 통해 어떤 `key`의 속성도 반환할 수 있지만 제네릭 매개변수를 통해 `extends keyof Type`으로 `Key` 제네릭 유형을 제한 헀고, 그것으로 `key`로 받아서 `x`의 속성에 없는 "m"의 경우 오류로 처리합니다.
훌륭하지 않나요?


## 제네릭에서 클래스 유형 사용

다음의 경우와 같이 팩토리를 만들 때 생성자 함수로 클래스 유형을 참조해야 할 수 있습니다.

```typescript
function create<Type>(c: { new (): Type }): Type {
  return new c();
}
```

고급 예제는 다음과 같습니다.

```typescript
class BeeKeeper {
  hasMask: boolean = true;
}
 
class ZooKeeper {
  nametag: string = "Mikle";
}
 
class Animal {
  numLegs: number = 4;
}
 
class Bee extends Animal {
  keeper: BeeKeeper = new BeeKeeper();
}
 
class Lion extends Animal {
  keeper: ZooKeeper = new ZooKeeper();
}
 
function createInstance<A extends Animal>(c: new () => A): A {
  return new c();
}
 
createInstance(Lion).keeper.nametag;
createInstance(Bee).keeper.hasMask;
```

이러한 패턴은 [mixins](https://www.typescriptlang.org/docs/handbook/mixins.html) 패턴을 강화하는데 사용됩니다.


## 정리

오늘은 TypeScript의 제네릭을 학습했습니다. 제네릭은 많은 언어에서 사용하는 방식이므로 제네릭 사용법을 익히면 다른 언어에서도 제네릭을 잘 사용할 수 있게 됩니다. 또한 제네릭을 효과적으로 사용하면 다양한 유형을 처리하기 위한 중복 코드를 없앨 수 있습니다.


