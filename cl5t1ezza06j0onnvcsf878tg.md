---
title: "TypeScript 배우기 - 6. 객체 유형"
datePublished: Wed Jul 20 2022 03:20:55 GMT+0000 (Coordinated Universal Time)
cuid: cl5t1ezza06j0onnvcsf878tg
slug: typescript-6
tags: typescript

---

JavaScript는 데이터를 그룹화 해서 전달할 수 있는 수단으로 객체 유형을 제공합니다. 앞에서 살펴 본 것처럼 다음의 무명 형태로 표현할 수 있습니다.

```typescript
function greet(person: { name: string; age: number }) {
  return "Hello " + person.name;
}
```

이것을 `type`으로 별칭으로 표현할 수도 있고요,

```typescript
type Person = {
  name: string;
  age: number;
};
 
function greet(person: Person) {
  return "Hello " + person.name;
}
```

인터페이스로도 표현할 수 있습니다.

```typescript
interface Person {
  name: string;
  age: number;
}
 
function greet(person: Person) {
  return "Hello " + person.name;
}
```

위의 세가지 표현은 `name`이라는 `string` 유형 속성과 `age`라는 `number` 유형 속성을 가진 객체 유형을 인자로 받는 함수입니다.


## 속성 수식어

TypeScript는 속성이 선택사항인지, 속성이 읽기 전용인지 등의 속성 수식어를 제공합니다.


### 선택적 속성

속성 이름에 `?`을 주면 선택적 속성이 됩니다.

```TypeScript
interface PaintOptions {
  shape: Shape;
  xPos?: number;
  yPos?: number;
}
 
function paintShape(opts: PaintOptions) {
  // ...
}
 
const shape = getShape();
paintShape({ shape });
paintShape({ shape, xPos: 100 });
paintShape({ shape, yPos: 100 });
paintShape({ shape, xPos: 100, yPos: 100 });
```

`xPos`와 `yPos` 속성은 생략해도 되는 속성이 됩니다. 유형은 `number | undefined` 유니언 유형이 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658281892635/dt1Ujqvtp.png align="left")

`xPos`와 `yPos`가 `undefined`가 될 수 있으므로 `undefined`일 때 별도의 처리를 해야 할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658281983637/0BtxtIF87.png align="left")

다음처럼 함수 인자에 초기 값을 줘서 위의 수행을 할 수 도 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658282022531/Ki6dwwHV3.png align="left")

위의 코드는 `paintShape`함수에 구조 해재 패턴을 사용해서 `xPos` 및 `yPos`의 기본 값을 제공했습니다.

> 현재로서는 구조 해제 패턴에 유형 주석을 배치할 수 없습니다. 다음의 구문은 JavaScript에서 다른 의미이기 때문입니다.

```typescript
function draw({ shape: Shape, xPos: number = 100 /*...*/ }) {
  render(shape);
         ~~~~~
> Cannot find name 'shape'. Did you mean 'Shape'?
  render(xPos);
         ~~~~
> Cannot find name 'xPos'.
}
```

JavaScript의 구조 해제 패턴에서 `shape: Shape`는 `shape`가 `Shape` 유형이라는 의미가 아니라 `shape` 속성이 함수 로컬 변수로 `Shape`로 잡겠다는 의미입니다. `xPos: number`도 마찬가지로 해석하기 때문에 위의 코드처럼 사용할 수 없습니다.

### `readonly` 속성

`readonly` 속성은 TypeScript에서 읽기 전용이라는 의미를 부여합니다. JavaScript의 런타임 동작에는 영향을 주지 않지만 읽기 전용이라고 표시된 속성은 TypeScript의 유형 검사에서 수정할 경우 오류로 판단합니다.

```typescript
interface SomeType {
  readonly prop: string;
}
 
function doSomething(obj: SomeType) {
  // We can read from 'obj.prop'.
  console.log(`prop has the value '${obj.prop}'.`);
 
  // But we can't re-assign it.
  obj.prop = "hello";
      ~~~~
> Cannot assign to 'prop' because it is a read-only property.
}
```

하지만 `readonly` 수식어를 썼다고 속성의 내부 내용을 변경할 수 없다는 것은 아닙니다. 속성 자체를 다시 설정할 수 없다는 의미입니다.

```typescript
interface Home {
  readonly resident: { name: string; age: number };
}
 
function visitForBirthday(home: Home) {
  // We can read and update properties from 'home.resident'.
  console.log(`Happy birthday ${home.resident.name}!`);
  home.resident.age++;
}
 
function evict(home: Home) {
  // But we can't write to the 'resident' property itself on a 'Home'.
  home.resident = {
       ~~~~~~~~
> Cannot assign to 'resident' because it is a read-only property.
    name: "Victor the Evictor",
    age: 42,
  };
}
```

`readonly`의 동작을 좀 더 살펴볼 필요가 있습니다. TypeScript는 두 유형의 속성이 호환되는지 살필 때 `readonly`를 고려하지 않으므로 읽기 전용 속성도 별칭을 통해 변경할 수 있습니다.

```typescript
interface Person {
  name: string;
  age: number;
}
 
interface ReadonlyPerson {
  readonly name: string;
  readonly age: number;
}
 
let writablePerson: Person = {
  name: "Person McPersonface",
  age: 42,
};
 
// works
let readonlyPerson: ReadonlyPerson = writablePerson;
 
console.log(readonlyPerson.age); // prints '42'
writablePerson.age++;
console.log(readonlyPerson.age); // prints '43'
```

위의 코드를 보시면 `Person`과 `ReadonlyPerson`으로 수정 가능한 인터페이스와 수정 불가능한 인터페이스를 정의했고, 먼저 수정 가능한 인터페이스로 `writablePerson` 객체를 생성한 후 수정 불가능한 인터페이스의 `readonlyPerson` 변수에 `writablePerson` 객체를 대입했습니다.
이후 `writablePerson.age++` 코드에 의해 `readonlyPerson.age`의 값 역시 증가함을 확인할 수 있습니다.


### 인덱스 서명

배열의 인덱스 및 반환 유형을 다음의 코드처럼 표현할 수 있습니다.

```typescript
interface StringArray {
  [index: number]: string;
}
 
const myArray: StringArray = getStringArray();
const secondItem = myArray[1];
```

TypeScript에서는 `StringArray` 인터페이스로 만들어진 객체는 배열의 요소
에 접근할 때 `number` 유형의 인덱스로 접근하고 그 반환형이 `string` 유형인 것 만을 인정합니다.

> TypeScript는 두가지 이상의 인덱서를 지원하기는 하지만 반환형은 반드시 숫자 인덱서에서 반환되는 유형과 같거나 상위 유형으로 강제합니다. 아닌 경우 오류가 발생합니다.

```typescript
interface Animal {
  name: string;
}
 
interface Dog extends Animal {
  breed: string;
}
 
// Error: indexing with a numeric string might get you a completely separate type of Animal!
interface NotOkay {
  [x: number]: Animal;
> 'number' index type 'Animal' is not assignable to 'string' index type 'Dog'.
  [x: string]: Dog;
}
```

문자열 인덱스 서명은 사전 패턴을 표현하는 강력한 방법이지만 모든 속성이 반환 유형과 같아야 합니다.

```typescript
interface NumberDictionary {
  [index: string]: number;
 
  length: number; // ok
  name: string;
        ~~~~~~
> Property 'name' of type 'string' is not assignable to 'string' index type 'number'.
}
```

인덱스 서명의 반환형이 유니언 유형의 경우 가능합니다.

```typescript
interface NumberOrStringDictionary {
  [index: string]: number | string;
  length: number; // ok, length is a number
  name: string; // ok, name is a string
}
```

다른 속성과 동일하게 인덱스에 대한 할당을 하지 못하도록 `readonly` 수식어를 사용할 수 있습니다.

```typescript
interface ReadonlyStringArray {
  readonly [index: number]: string;
}
 
let myArray: ReadonlyStringArray = getReadOnlyStringArray();
myArray[2] = "Mallory";
> Index signature in type 'ReadonlyStringArray' only permits reading.
```


## 확장 유형

비슷하지만 좀 더 구체적인 유형을 만들어야 하는 이유는 매우 일상적으로 발생합니다. 다음의 기본 주소는,

```typescript
interface BasicAddress {
  name?: string;
  street: string;
  city: string;
  country: string;
  postalCode: string;
}
```

다음의 `단위`가 포함되어야 하는 주소일 경우,

```typescript
interface AddressWithUnit {
  name?: string;
  unit: string;
  street: string;
  city: string;
  country: string;
  postalCode: string;
}
```

`BasicAddress`의 대부분의 속성과 동일하지만 반복해서 그 속성을 사용해야만 합니다. 이를 `extends` 키워드를 사용해 다음처럼 간단히 표현할 수 있습니다.

```typescript
interface BasicAddress {
  name?: string;
  street: string;
  city: string;
  country: string;
  postalCode: string;
}
 
interface AddressWithUnit extends BasicAddress {
  unit: string;
}
```

`AddressWithUnit ` 인터페이스는 `unit` 뿐만 아니라 `BasicAddress`의 속성 구성도 그대로 계승하게 됩니다.

다음처럼 여러 유형에서 확장할 수 도 있습니다.

```typescript
interface Colorful {
  color: string;
}
 
interface Circle {
  radius: number;
}
 
interface ColorfulCircle extends Colorful, Circle {}
 
const cc: ColorfulCircle = {
  color: "red",
  radius: 42,
};
```

## 교차 유형

TypeScript에서는 교차 유형이라는 것도 제공합니다. 교차 유형은 기존 유형을 결합하는데 사용할 수 있습니다.

```typescript
interface Colorful {
  color: string;
}
interface Circle {
  radius: number;
}
 
type ColorfulCircle = Colorful & Circle;
```

```typescript
function draw(circle: Colorful & Circle) {
  console.log(`Color was ${circle.color}`);
  console.log(`Radius was ${circle.radius}`);
}
 
// okay
draw({ color: "blue", radius: 42 });
 
// oops
draw({ color: "red", raidus: 42 });
> Argument of type '{ color: string; raidus: number; }' is not assignable to parameter of type 'Colorful & Circle'.
>  Object literal may only specify known properties, but 'raidus' does not exist in type 'Colorful & Circle'. Did you mean to write 'radius'?
```


## 인터페이스 vs 교차

인터페이스와 교차는 다른 방식으로 유사한 결과를 만들 수 있습니다. 인터페이스와 교차의 차이점은 충돌을 처리하는 방법이며, 이 충돌 처리에 따라 인터페이스와 교차 유형 중 하나를 선택할 수 있습니다.


## 제네릭 객체 유형

어떠한 내용 유형인지 상관없는 `Box` 인터페이스는 다음처럼 표현할 수 있습니다.

```typescript
interface Box {
  contents: any;
}
```

하지만 `any` 유형이므로 TypeScript의 유형 검사 및 편집기 기능을 이용할 수 없습니다. 유형을 알지 못할 경우 `unknown` 유형으로 TypeScript의 유형 검사 시점에서 이 객체를 잘못 사용하는 것을 방지할 수 있습니다.

```typescript
interface Box {
  contents: unknown;
}
 
let x: Box = {
  contents: "hello world",
};
 
// we could check 'x.contents'
if (typeof x.contents === "string") {
  console.log(x.contents.toLowerCase());
}
 
// or we could use a type assertion
console.log((x.contents as string).toLowerCase());
```

하지만 유형을 특정할 수 있는 경우 각각 인터페이스로 해당 유형을 기술한 후 함수 오버로드로 이를 표현할 수 있습니다.

```typescript
interface NumberBox {
  contents: number;
}
 
interface StringBox {
  contents: string;
}
 
interface BooleanBox {
  contents: boolean;
}
```

```typescript
function setContents(box: StringBox, newContents: string): void;
function setContents(box: NumberBox, newContents: number): void;
function setContents(box: BooleanBox, newContents: boolean): void;
function setContents(box: { contents: any }, newContents: any) {
  box.contents = newContents;
}
```

하지만 복잡하죠. 이를 제네릭을 이용해 다음처럼 간단히 표현할 수 있습니다.

```typescript
interface Box<Type> {
  contents: Type;
}
```

이제 이 제네릭 인터페이스의 사용은 다음처럼 할 수 있습니다.

```typescript
let box: Box<string>;
```

결국에 `StringBox`와 동일하게 처리가 가능하게 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658285249001/er6Yv6AWw.png align="left")

제네릭을 사용하면 `string` 뿐만 아니라 다양한 유형으로 대체할 수 있다는 점이 장점입니다.

```typescript
interface Box<Type> {
  contents: Type;
}
 
interface Apple {
  // ....
}
 
// Same as '{ contents: Apple }'.
type AppleBox = Box<Apple>;
```

이제 `setContents()`함수를 다음처럼 개선할 수 있습니다.

```typescript
function setContents<Type>(box: Box<Type>, newContents: Type) {
  box.contents = newContents;
}
```

유형 별칭 `type`에도 제네릭을 사용할 수 있습니다.

```typescript
interface Box<Type> {
  contents: Type;
}
```

이것을 `type`을 사용해서 다음처럼 쓸 수 있습니다.

```typescript
type Box<Type> = {
  contents: Type;
};
```

하지만 유형 별칭은 인터페이스와 달리 개체 유형 이상을 설명할 수 있으므로 다른 종류의 도우미 유형을 작성하는데 사용할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658285424733/QFQRKoNgR.png align="left")


### `Array` 유형

제네릭 객체 유형은 포함된 요소와 독립적으로 작동하는 일종의 컨테이너 유형일 수 있습니다. 다양한 데이터 유형에서 재사용할 수 있도록 데이터 구조가 이런 방식으로 작동하는게 이상적입니다.

실제로 `numer[]`, `string[]`는 `Array<number>` 및 `Array<string>`의 의미입니다.

```typescript
function doSomething(value: Array<string>) {
  // ...
}
 
let myArray: string[] = ["hello", "world"];
 
// either of these work!
doSomething(myArray);
doSomething(new Array("hello", "world"));
```

Array 자체가 제네릭 유형임을 확인할 수 있습니다.

```typescript
interface Array<Type> {
  /**
   * Gets or sets the length of the array.
   */
  length: number;
 
  /**
   * Removes the last element from an array and returns it.
   */
  pop(): Type | undefined;
 
  /**
   * Appends new elements to an array, and returns the new length of the array.
   */
  push(...items: Type[]): number;
 
  // ...
}
```

최신 JavaScript에서는 `Map<K, V>`, `Set<T>` 및 `Promise<T>`와 같이 제네릭인 다른 데이터 구조도 제공합니다. 이것의 의미는 제네릭 유형의 유형 집합과 함께 작동할 수 있다는 점입니다.


### `ReadonlyArray` 유형

`ReadonlyArray`는 배열의 요소가 변경되지 않아야 하는 것을 설명합니다.

```typescript
function doStuff(values: ReadonlyArray<string>) {
  // We can read from 'values'...
  const copy = values.slice();
  console.log(`The first value is ${values[0]}`);
 
  // ...but we can't mutate 'values'.
  values.push("hello!");
         ~~~~
> Property 'push' does not exist on type 'readonly string[]'.
}
```

속성에 대한 `readonly` 수식어와 마찬가지로 배열의 내용이 변경될 수 없음을 나타내는 도구입니다. 이렇게 표현하면 이 배열이 어떠한 함수에 전달되어도 변경되는 것을 걱정할 필요가 없습니다.

`ReadonlyArray` 유형은 `new`로 생성할 수 없습니다.

```typescript
new ReadonlyArray("red", "green", "blue");
    ~~~~~~~~~~~~~
> 'ReadonlyArray' only refers to a type, but is being used as a value here.
```

대신 다음과 같이 사용합니다.

```typescript
const roArray: ReadonlyArray<string> = ["red", "green", "blue"];
```

TyoeScruot 가 `Array<Type>`의 약식 구문인 `Type[]`을 제공하는 것처럼 마찬가지로 `readonly Type[]`로 `ReadonlyArray<Type>`을 표현할 수 있습니다.

```typescript
function doStuff(values: readonly string[]) {
  // We can read from 'values'...
  const copy = values.slice();
  console.log(`The first value is ${values[0]}`);
 
  // ...but we can't mutate 'values'.
  values.push("hello!");
         ~~~~
> Property 'push' does not exist on type 'readonly string[]'.
}
```

하지만 주의할 점은` readonly` 수식어와 달리 할당 가능성은 Array와 ReadonlyArray간 양방향이 아닙니다.

```typescript
let x: readonly string[] = [];
let y: string[] = [];
 
x = y;
y = x;
~
> The type 'readonly string[]' is 'readonly' and cannot be assigned to the mutable type 'string[]'.
```


### 튜플 유형

튜플 유형은 포함된 요소의 수와 특정 위치에 포함된 유형을 정확히 알고 있는 또다른 종류의 Array 유형입니다.

```typescript
type StringNumberPair = [string, number];
```

여기서 `StringNumberPair`은 `stirng`과 `number` 유형의 튜플 유형입니다. 물론 JavaScript의 런타임 시점에서는 이런 표현이 의미가 없지만 TypeScript에서는 배열의 요소 수와 포함된 유형에 대한 정보가 있으므로 중요합니다.

```typescript
function doSomething(pair: [string, number]) {
  const a = pair[0];
       
const a: string
  const b = pair[1];
       
const b: number
  // ...
}
 
doSomething(["hello", 42]);
```

이제 튜플 유형에서 벗어난 값을 대입하려 할 때 오류가 발생합니다.

```typescript
function doSomething(pair: [string, number]) {
  // ...
 
  const c = pair[2];
                 ~
> Tuple type '[string, number]' of length '2' has no element at index '2'.
}
```

또한 튜플 유형은 구조 해제를 통해서도 유용합니다.

```typescript
function doSomething(stringHash: [string, number]) {
  const [inputString, hash] = stringHash;
 
  console.log(inputString);
                  
const inputString: string
 
  console.log(hash);
               
const hash: number
}
```

> 튜플 유형은 규약 기반 API에서 유용합니다.

또한 아래와 같이도 표현 가능합니다.

```typescript
interface StringNumberPair {
  // specialized properties
  length: 2;
  0: string;
  1: number;
 
  // Other 'Array<string | number>' members...
  slice(start?: number, end?: number): Array<string | number>;
}
```

튜플 유형은 선택적 속성을 가질 수 도 있습니다. 선택적 속성은 튜블 요소 끝에만 가질 수 있습니다.

```typescript
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658286498064/VIolvYG2f.png align="left")
```

선택적 요소에 의해 길이가 `2 | 3`의 리터럴 유니언 유형이라는 것이 흥미롭네요.

튜플은 또한 나머지 요소를 다음처럼 표현할 수 있습니다.

```typescript
type StringNumberBooleans = [string, number, ...boolean[]];
type StringBooleansNumber = [string, ...boolean[], number];
type BooleansStringNumber = [...boolean[], string, number];
```

나머지 요소에 대한 표현이 앞과 중간, 끝에 위치할 수 있는 이유는 유형이 다르기 때문입니다. 만약 유형이 같다면 반드시 마지막에 위치해야 합니다.

나머지 요소가 있는 튜플에는 `길이`에 대한 설정이 없습니다

```typescript
const a: StringNumberBooleans = ["hello", 1];
const b: StringNumberBooleans = ["beautiful", 2, true];
const c: StringNumberBooleans = ["world", 3, true, false, true, false, true];
```

선택적 요소 또는 나머지 요소가 왜 유용할까요? TypeScript가 함수의 매개변수 목록과 튜플에 대응할 수 있도록 하기 때문입니다. 튜플 유형은 나머지 매개변수 및 인수에서 사용할 수 있으므로 다음과 같이 사용할 수 있습니다.

```typescript
function readButtonInput(...args: [string, number, ...boolean[]]) {
  const [name, version, ...input] = args;
  // ...
}
```

이것은 기본적으로 다음과 같습니다.

```typescript
function readButtonInput(name: string, version: number, ...input: boolean[]) {
  // ...
}
```

이것은 나머지 매개변수를 사용해서 가변 개수의 인수를 사용하지만 중간 변수를 도입하고 싶지 않을 때 유용합니다.


### `readonly` 튜플 유형

튜플 유형은 마찬가지로 `readonly` 수식어를 사용할 수 있습니다.

```typescript
function doSomething(pair: readonly [string, number]) {
  pair[0] = "hello!";
       ~
> Cannot assign to '0' because it is a read-only property.
}
```

튜플은 대부분은 생성된 이후 수정되지 않아야 하는 목적에 사용되므로 대부분의 경우 읽기 전용 튜플을 사용하는 것이 좋습니다. 이는 `as const`가 읽기 전용 튜플 유형으로 유추 된다는 점에서도 중요합니다.

```typescript
let point = [3, 4] as const;
 
function distanceFromOrigin([x, y]: [number, number]) {
  return Math.sqrt(x ** 2 + y ** 2);
}
 
distanceFromOrigin(point);
                   ~~~~~
> Argument of type 'readonly [3, 4]' is not assignable to parameter of type '[number, number]'.
>   The type 'readonly [3, 4]' is 'readonly' and cannot be assigned to the mutable type '[number, number]'.
```


## 정리

오늘은 핸드북의 `Object Types`에 대해 알아보았습니다. 다음시간에는 핸드북의 `Type Manipulation`을 순서대로 살펴보도록 하겠습니다.
