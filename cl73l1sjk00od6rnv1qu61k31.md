---
title: "TypeScript 배우기 - 14. 클래스"
datePublished: Sun Aug 21 2022 17:07:55 GMT+0000 (Coordinated Universal Time)
cuid: cl73l1sjk00od6rnv1qu61k31
slug: typescript-14
tags: typescript

---

> 본 장은 핸드북의 [Classes](https://www.typescriptlang.org/docs/handbook/2/classes.html)의 내용을 최대한 간결하게 정리하고 보완하는 것을 목표로 작성했습니다.


TypeScript는 ES2015에 도입된 클래스를 완벽히 지원합니다.



## 클래스 멤버

가장 간단한 클래스는 다음과 같습니다.

```typescript
class Point {}
```

이 문장은 ES2015 이전에는 음과 같이 JavaScript 코드로 번역됩니다.

```javascript
"use strict";
var Point = /** @class */ (function () {
    function Point() {
    }
    return Point;
}());
```

ES2015부터 다음과 같습니다.

```javascript
"use strict";
class Point {
}
```

클래스를 사용하는 이유는 클래스 또는 클래스의 인스턴스 단위로 격리된 데이터와 기능을 담기 위함입니다. 이것은 객체지향 프로그래밍의 개체 단위가 됩니다.

하지만 위의 코드는 아무런 데이터와 기능을 가지지 않으므로 확장하도록 하겠습니다.

### 필드

필드는 클래스에 데이터를 가지게 합니다.

```typescript
class Point {
  x: number;
  y: number;
}
 
const pt = new Point();
pt.x = 0;
pt.y = 0;
```

`Point`라는 클래스는 위치 정보를 표현하니까 2차원 좌표 `x`, `y` 값을 가지게 되었습니다. 클래스는 `new` 키워드를 통해 클래스의 인스턴스를 생성합니다. 인스턴스는 클래스 형태를 지닌 격리된 저장소 및 기능 집합입니다.

필드는 초기화를 할 수 있습니다. 다음의 코드는 타입 추론을 통해 타입을 지정하고 초기화 하는 코드 입니다.

```typescript
class Point {
  x = 0;
  y = 0;
}
 
const pt = new Point();
// Prints 0, 0
console.log(`${pt.x}, ${pt.y}`);
```

`x`, `y`의 타입이 추론에 의해 `number`가 되었으므로 아래의 문자열 값 대입은 오류입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661053098336/7PHEf3qjL.png align="left")


### --strictPropertyInitialization

[strictPropertyInitialization](https://www.typescriptlang.org/tsconfig#strictPropertyInitialization) 옵션을 통해 반드시 필드를 초기화 해야 하는지 설정할 수 있습니다. 활성화 하면 다음의 코드는 초기화 하지 않았으므로 컴파일 오류입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661053199889/kD86QMaDL.png align="left")

초기화는 생성자에서 할 수 도 있습니다. 이럴 경우 컴파일 오류를 발생하지 않습니다.

```typescript
class GoodGreeter {
  name: string;
 
  constructor() {
    this.name = "hello";
  }
}
```

TypeScript는 생성자에서 호출하는 메서드에서 필드를 초기화 하는 것을 감지하지 않습니다. 그러므로 필드 선언 시 초기값을 부여하거나 생성자에서 초기화 해야 합니다.
 
만약 생성자가 아닌 다른 곳에서 반드시 초기화 될 것이 확실하다면 필드명 끝에 `!`를 붙여서 컴파일 오류를 발생하지 않도록 할 수 있습니다.

```typescript
class OKGreeter {
  // Not initialized, but no error
  name!: string;
}
```


### `readonly`

필드 앞에 `readonly` 수식어를 붙이면 생성자를 제외한 다른 영역에서 필드 값을 수정할 수 없도록 합니다.


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661053491110/-OPiTX-mn.png align="left")


### 생성자

클래스 생성자는 일반 함수와 유사하지만 클래스 인스턴스가 생성될 때 한번 호출됩니다. 타입, 기본값 및 오버로드가 있는 매개변수를 가질 수 있습니다.

```typescript
class Point {
  x: number;
  y: number;
 
  // Normal signature with defaults
  constructor(x = 0, y = 0) {
    this.x = x;
    this.y = y;
  }
}
```

```typescript
class Point {
  // Overloads
  constructor(x: number, y: string);
  constructor(s: string);
  constructor(xs: any, y?: any) {
    // TBD
  }
}
```

하지만 클래스 생성자는 함수와 다른 다음의 특징이 있습니다.

- 타입 매개변수를 가질 수 없습니다.
- 반환 타입이 없습니다. (클래스의 인스턴스를 반환합니다)


#### Super 호출

베이스 클래스가 있을 경우 생성자에서 `super()`를 호출해서 초기화 작업을 수행할 수 있습니다. `this`의 초기화 작업 전에 호출 되어야 합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661053900259/MHWgOgkM_.png align="left")


종종 잊을 수 있는 `super()` 호출을 TypeScript는 필요할 때 알려줍니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661054035652/kmCXnVhd6.png align="left")


### 메서드

클래스의 함수 속성을 메서드라고 하니다. 메서드는 함수 및 생성자와 동일한 타입 표현을 쓸 수 있습니다.

```typescript
class Point {
  x = 10;
  y = 10;
 
  scale(n: number): void {
    this.x *= n;
    this.y *= n;
  }
}
```

메서드에서 클래스의 필드에 접근하려면 항상 `this.`을 붙여야 합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661054193798/kTUfuOHkj.png align="left")


### Getter / Setter

특정 값의 접근하거나 설정할 때 로직이 필요한 경우가 있을 수 있습니다. 다음은 `length`의 값을 읽거나 설정할 때 로직을 구현합니다.

```typescript
class C {
  _length = 0;
  get length() {
    return this._length;
  }
  set length(value) {
    this._length = value;
  }
}
```

하지만 읽거나 설정할 때 특별한 로직이 필요 없는 경우 그냥 공개 필드를 노출하는 것이 좋습니다.

다음은 TypeScript에서 Getter / Setter에 의해 추론하는 방법입니다.
- `get`은 있지만 `set`은 없는 경우 `readonly`와 동일하게 처리
- `setter`의 매개변수 타입이 지정되지 않았을 경우 `getter`의 반환 타입으로 타입 추론
- [`Getter`와 `Setter`는 동일한 멤버 표시성](https://www.typescriptlang.org/docs/handbook/2/classes.html#member-visibility)을 가져야 합니다.

하지만 TypeScript 4.3부터 가져오고 설정하기 위해 다른 타입을 허용합니다.

```typescript
class Thing {
  _size = 0;
 
  get size(): number {
    return this._size;
  }
 
  set size(value: string | number | boolean) {
    let num = Number(value);
 
    // Don't allow NaN, Infinity, etc
 
    if (!Number.isFinite(num)) {
      this._size = 0;
      return;
    }
 
    this._size = num;
  }
}
```

하지만 위의 코드는 설정 값의 타입과 상관없이 여전히 `size`가 `number` 타입임을 나타냅니다.


### 인덱스 서명

클래스는 배열 접근과 유사하게 인덱스 서명을 만들 수 있습니다.

```typescript
class MyClass {
  [s: string]: boolean | ((s: string) => boolean);
 
  check(s: string) {
    return this[s] as boolean;
  }
}
```

하지만 인덱스 서명은 메서드 타입도 캡쳐해야 하므로 이런 타입을 유용하게 사용하기 쉽지 않습니다. 일반적으로 클래스 인스턴스 자체가 아닌 다른 위치에 인덱싱된 데이터를 저장하는 것이 좋습니다.


## 클래스 상속

객체 지향 기능이 있는 다른 언어와 마찬가지로 JavaScript의 클래스는 베이스 클래스에서 상속 받을 수 있습니다.


### `implements`절

`implements`절을 사용하여 특정 `interface`를 구현할 수 있습니다. 하지만 `interface` 명세를 정확히 구현하지 않으면 오류가 발생합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661055511912/tJYgtOMij.png align="left")

클래스는 또한 한 개 이상의 다양한 인터페이스를 구현할 수 있습니다.


#### 주의사항

다른 객체 지향 언어와 달리 TypeScript의 `구현`은 구현 대상인 인터페이스의 명세 점검만 할 뿐 어떠한 관련도 없다는 것을 알아야 합니다. 즉 인터페이스를 구현한 클래스는 인터페이스의 특징을 계승하지 않습니다. 아래의 코드를 보시죠.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661055525722/Op6Qy6XQF.png align="left")

`NameChecker`가 `Checkable` 인터페이스를 구현했으므로 `check` 함수의 매개변수 `name` 타입이 암묵적으로 적용될 것이라 기대하지만 그렇지 않습니다.

또한 아래 코드를 보면 선택적 필드를 클래스에서 정의하지 않았으므로 쓸 수 없음을 알 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661055613488/ETGYIMFa1.png align="left")

다시 돌아와서 TypeScript의 구현은 단지 정적 타입 검사의 한 가지 방법일 뿐이라는 것을 알아야 합니다.


### `extends` 절

클래스는 베이스 클래스에서 `확장` 할 수 있습니다. 파생 클래스에서 베이스 클래스의 모든 속성과 메서드가 있으며 추가 멤버도 정의할 수 있습니다.

```typescript
class Animal {
  move() {
    console.log("Moving along!");
  }
}
 
class Dog extends Animal {
  woof(times: number) {
    for (let i = 0; i < times; i++) {
      console.log("woof!");
    }
  }
}
 
const d = new Dog();
// Base class method
d.move();
// Derived class method
d.woof(3);
```


#### 메서드 재정의

파생 클래스는 베이스 클래스의 필드 또는 속성을 `super.`를 이용해 쉽게 재정의할 수 있습니다.

TypeScript는 파생 클래스가 항상 베이스 클래스의 하위 타입이 되도록 합니다. 예를 들어 다음의 코드는 합법적인 코드입니다.

```typescript
class Base {
  greet() {
    console.log("Hello, world!");
  }
}
 
class Derived extends Base {
  greet(name?: string) {
    if (name === undefined) {
      super.greet();
    } else {
      console.log(`Hello, ${name.toUpperCase()}`);
    }
  }
}
 
const d = new Derived();
d.greet();
d.greet("reader");
```

베이스 클래스 참조를 통해 파생 클래스 인스턴스의 특정 필드나 속성에 접근하는 것은 TypeScript에서는 일반적입니다. (정적 언어에서는 일방적이지 않습니다!) 즉, 다음의 코드는 TypeScript에서 올바른 코드입니다.

```typescript
// Alias the derived instance through a base class reference
const b: Base = d;
// No problem
b.greet();
```

서명을 따르지 않았을 경우 TypeScript은 다음과 같이 컴파일 오류를 발생합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661056064329/RGdpO8wM4.png align="left")

컴파일 오류가 발생했음에도 생성된 다음의 코드를 실행했을 때,

```typescript
const b: Base = new Derived();
// Crashes because "name" will be undefined
b.greet();
```

매개변수 `name`가 `undefined`가 될 것이므로 `name.toUpperCase()`에서 예외가 발생할 것입니다.


#### 타입 전용 필드 선언

ES2022 이상이거나 [useDefineForClassFields](https://www.typescriptlang.org/tsconfig#useDefineForClassFields) 옵션을 사용할 경우 `declare`를 통해 필드 선언에 대한 런타임 효과가 없어야 함을 TypeScript에 알려줍니다.

```typescript
interface Animal {
  dateOfBirth: any;
}
 
interface Dog extends Animal {
  breed: any;
}
 
class AnimalHouse {
  resident: Animal;
  constructor(animal: Animal) {
    this.resident = animal;
  }
}
 
class DogHouse extends AnimalHouse {
  // Does not emit JavaScript code,
  // only ensures the types are correct
  declare resident: Dog;
  constructor(dog: Dog) {
    super(dog);
  }
}
```

#### 초기화 순서

초기화 순서는 다른 언어와 다를 수 있습니다. 아래의 코드를 통해 살펴봅시다.

```typescript
class Base {
  name = "base";
  constructor() {
    console.log("My name is " + this.name);
  }
}
 
class Derived extends Base {
  name = "derived";
}
 
// Prints "base", not "derived"
const d = new Derived();
```

- 베이스 클래스 필드가 먼저 초기화
- 베이스 클래스 생성자가 실행
- 파생 클래스 필드가 초기화
- 파생 클래스 생성자가 실행


##  멤버 가시성

TypeScript에서는 특정 메서드나 속성이 클래스 외부에 노출할 지를 제어할 수 있습니다.


### `public`

`public`은 클래스 멤버의 기본 가시성입니다. `public` 멤버는 외부에서 자유롭게 접근할 수 있습니다.

```typescript
class Greeter {
  public greet() {
    console.log("hi!");
  }
}
const g = new Greeter();
g.greet();
```

`public`은 기본 가시성 수식어이기 때문에 생략해도 됩니다. 하지만 코드 스타일 또는 가독성을 위해 `public`을 사용하는 것을 선택해도 됩니다.


### `protected`

`protected` 멤버는 클래스를 상속 받는 하위 클래스에서만 볼 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661096462025/48Rw_It9W.png align="left")


#### protected 멤버 노출

파생 클래스는 베이스 클래스의 계약을 따라야 하지만 떄에 따라 베이스 클래스의 서브타입을 노출하도록 할 수 있습니다. 다음은 `protected` 멤버를 `public`으로 만드는 코드입니다.

```typescript
class Base {
  protected m = 10;
}
class Derived extends Base {
  // No modifier, so default is 'public'
  m = 15;
}
const d = new Derived();
console.log(d.m); // OK
```

`Dervied`는 `m`을 자유롭게 읽고 쓸 수 있으므로 특별한 "보안" 문제는 없습니다. 다만 파생 클래스에서 `protected` 맴버를  `public`으로 변경하는 것은 의도하지 않은 경우 주의해야 합니다.


#### 계층 간 protected 접근

객체 지향 언어에 따라 베이스 클래스의 `protected` 멤버 접근에 대한 제한이 다릅니다. TypeScript의 경우 베이스 클래스 참조를 통한 `protected`는 접근할 수없도록 합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661097023255/Qdrg3F46t.png align="left")


### `private`

`protected`와 비슷하지만 하위 클래스에서도 멤버에 대한 접근을 허용하지 않습니다.


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661097089241/aSeIy8aA2.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661097106198/AL7wIT-CY.png align="left")

`private` 멤버는 파생 클래스에서 볼 수 없으므로 가시성을 상승 시킬 수 없습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661097151902/9YXXtInHP.png align="left")


#### 인스턴스 간 private 접근

TypeScript은 같은 클래스의 다른 인스턴스의 경우  `private` 멤버 접근을 하용합니다.


```typescript
class A {
  private x = 10;
 
  public sameAs(other: A) {
    // No error
    return other.x === this.x;
  }
}
```


#### 주의 사항

TypeScript의 다른 타입 시스템과 마찬가지로 `private`, `protected` 타입은 타이 검사 시점에만 적용됩니다. 예를 들어 JavaScript의 런타임 시 `in` 또는 단순 속성 조회에서 `private` 또는 `protected` 멤버에 액세스 할 수 있음을 의미합니다.

```typescript
class MySafe {
  private secretKey = 12345;
}
```

```typescript
// In a JavaScript file...
const s = new MySafe();
// Will print 12345
console.log(s.secretKey);
```

`private` 또한 타입 검사 중에 대괄호 표기법을 통해 접근할 수 있습니다. 이런 특징으로 인해 단위테스트에서 `private` 선언 필드의 테스트가 용이하지만 결국에 TypeScript의 멤버 가시성 소프트웨어 private은 타입 검사의 일환으로 개인 정보를 엄격하게 적용하지 않는다는 단점이 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661097769797/Vq5uMq-SJ.png align="left")

이와 달리 JavaScript의 `private` 표기 (#)는 컴파일 후 비공개로 유지되며 대괄호 표기법 접근에도 접근되지 않으므로 하드 private 입니다.

```typescript
class Dog {
  #barkAmount = 0;
  personality = "happy";
 
  constructor() {}
}
```

```javascript
"use strict";
class Dog {
    #barkAmount = 0;
    personality = "happy";
    constructor() { }
}
```

ES2021 이하로 컴파일 할 경우 `#`은 클래스 이름으로 `_Dog_` 접두사로 치장됩니다.

```typescript
"use strict";
var _Dog_barkAmount;
class Dog {
    constructor() {
        _Dog_barkAmount.set(this, 0);
        this.personality = "happy";
    }
}
_Dog_barkAmount = new WeakMap();
```


JavaScript에서 악의적인 행위자로부터 클래스 값을 보호하기 위해서는 클로저, WeakMaps, private 필드(#)와 같은 엄격한 런타임 private 정보를 제공하는 메커니즘을 사용해야 합니다. 런타임 중에 이러한 추가된 개인 정보 확인은 성능에 영향을 줄 수 있습니다.


## 정적 멤버

클래스는 static 구성원이 있을 수 있습니다. 이러한 멤버는 클래스 자체로 접근할 수 있습니다.

```typescript
class MyClass {
  static x = 0;
  static printX() {
    console.log(MyClass.x);
  }
}
console.log(MyClass.x);
MyClass.printX();
```

정적 멤버 역시 `public`, `protected`, `private` 가시성 수식어를 사용할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661098164912/0j00SFi1i.png align="left")

또한 정적 멤버도 상속됩니다.

```typescript
class Base {
  static getGreeting() {
    return "Hello world";
  }
}
class Derived extends Base {
  myGreeting = Derived.getGreeting();
}
```


### 특수 정적 이름

몇 가지 이름은 예약되어 있습니다. `name` 및 `length`등의 함수 속성은 정적 멤버로 정의하는데 유효하지 않습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661098295207/_-LOxESwH.png align="left")


### 정적 클래스가 없는 이유는 무엇인가요?

TypeScript(및 JavaScript)는 `static class`가 없습니다. 이유는 데이터와 기능을 클래스 내부에 강제로 포함하기 때문입니다. 이는 클래스 뿐만 아니라 일반 객체(또는 최상위 함수)도 마찬가지 입니다.

```typescript
// Unnecessary "static" class
class MyStaticClass {
  static doSomething() {}
}
 
// Preferred (alternative 1)
function doSomething() {}
 
// Preferred (alternative 2)
const MyHelperObject = {
  dosomething() {},
};
```


## `static` 클래스 블록

정적 블록을 사용하면 클래스 내의 private 필드에 액세스 할 수 있는 자체 범위를 사용하여 일련의 코드를 작성할 수 있습니다.

```typescript
class Foo {
    static #count = 0;
 
    get count() {
        return Foo.#count;
    }
 
    static {
        try {
            const lastInstances = loadLastInstances();
            Foo.#count += lastInstances.length;
        }
        catch {}
    }
}
```


## 제네릭 클래스

인터페이스와 마찬가지로 클래스는 제네릭일 수 있습니다. 제네릭 클래스가 new로 인스턴스화되면 해당 타입 매개변수는 함수 호출과 마찬가지 방식으로 유추됩니다. 


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661098910695/Np-89k_L0i.png align="left")

클래스에서 인터페이스와 마찬가지로 제네릭 제약 조건 및 기본값을 사용할 수 있습니다.


### 정적 멤버의 타입 매개변수

다음의 경우, 참조에 대한 제네릭의 경우 정적 멤버의 타입 매개변수가 될 수 없습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661098996751/_a2nTYEpa.png align="left")


## 런타임시 클래스의 `this`

TypeScript는 JavaScript의 몇가지 독특한 런타임 동작을 수정하지 않습니다. 예를 들어 `this`의 동작입니다.

```typescript
class MyClass {
  name = "MyClass";
  getName() {
    return this.name;
  }
}
const c = new MyClass();
const obj = {
  name: "obj",
  getName: c.getName,
};
 
// Prints "obj", not "MyClass"
console.log(obj.getName());
```

`obj`의 `getName`에 의해 `this`는 `MyClass`의 인스턴스가 아닌 `obj`가 됩니다. 그러므로 `obj`가 출력되는데 이는 다른 언어에서는 가지지 않는 JavaScript만의 특징입니다.

이런 동작을 보통 원하지는 않습니다. 이것을 완화하는 몇 가지 방법이 있습니다.


###  화살표 함수 (람다 함수)

메서드 정의 대신 화살표 함수 속성을 사용하면 이러한 원치 않는 동작을 완화할 수 있습니다.

```typescript
class MyClass {
  name = "MyClass";
  getName = () => {
    return this.name;
  };
}
const c = new MyClass();
const g = c.getName;
// Prints "MyClass" instead of crashing
console.log(g());
```

몇 가지 장단점이 있습니다.

- (장점) 런타임 시 `this`의 값이 정확함을 보장
- (단점) 인스턴스마다 고유한 복사본을 가지게 되므로 좀 더 많은 메모리를 사용
- (단점) 베이스 클래스 메서드를 가져올 프로토타입 체인에 항목이 없기 때문에 파생 클래스에서 `super.getName`를 사용할 수 없음


### `this` 매개변수

메서드 또는 함수 정의에서 `this` 이름이 지정된 초기 매개변수는 TypeScript에서 특별한 의미를 가집니다. 다음 매개변수는 컴파일 중에 지워집니다.

TypeScript의 아래 코드는

```typescript
// TypeScript input with 'this' parameter
function fn(this: SomeType, x: number) {
  /* ... */
}
```

JavaScript의 아래 코드로 변환됩니다.

```javascript
// JavaScript output
function fn(x) {
  /* ... */
}
```

TypeScript는 `this` 매개변수가 있는 함수 호출이 올바른 컨텍스트에서 수행하는지를 확인합니다. 화살표 함수를 사용하는 대신 `this` 매개 변수를 사용해서 메서드가 올바르게 호출되도록 정적으로 적용할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661099611145/cRe-6c09E.png align="left")

이 방법은 화살표 함수 접근의 반대되는 장단점이 있습니다.

- (단점) JavaScript 호출자는 여전히 클래스 메서드를 잘못 사용할 수 있음
- (장점) 클래스 인스턴스당 하나가 아닌 클래스 정의당 하나의 함수만 사용
- (장점) 기본 메서드 정의는 여전히 `super`를 통해 호출 가능


## `this` 타입

클래스에서 `this` 라는 특수 타입은 현재 클래스의 타입을 동적으로 참조합니다. 이것이 얼마나 유용한지 살펴봅시다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661099887726/Vp09QSKVo.png align="left")

여기서 TypeScript는 반환 타입을 `Box`가 아닌 `this`로 유추했습니다. 이제 Box의 하위 클래스를 만들어 봅시다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661099949839/FIpZno3xG.png align="left")

매개변수에서도 this 타입을 사용할 수 있습니다.

```typescript
class Box {
  content: string = "";
  sameAs(other: this) {
    return other.content === this.content;
  }
}
```

이것은 `other: Box`라고 하는 것과 다릅니다. 파생 클래스의 경우 베이스 클래스 `Box`가 아닌 `ClearableBox`로 타입을 허용합니다. 반대로 `this` 타입을 사용할 경우 파생 클래스의 인스턴스만 허용합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661100145631/n2apN6GgS.png align="left")


### `this` - 기반 타입 가드

클래스 및 인터페이스의 메서드에 대한 반환 위치에서 `this is Type`을 사용할 수 있습니다. 타입 축소와 결합하면 대상 개체의 타입이 지정된 타입으로 축소됩니다.

```typescript
class FileSystemObject {
  isFile(): this is FileRep {
    return this instanceof FileRep;
  }
  isDirectory(): this is Directory {
    return this instanceof Directory;
  }
  isNetworked(): this is Networked & this {
    return this.networked;
  }
  constructor(public path: string, private networked: boolean) {}
}
 
class FileRep extends FileSystemObject {
  constructor(path: string, public content: string) {
    super(path, false);
  }
}
 
class Directory extends FileSystemObject {
  children: FileSystemObject[];
}
 
interface Networked {
  host: string;
}
 
const fso: FileSystemObject = new FileRep("foo/bar.txt", "foo");
 
if (fso.isFile()) {
  fso.content;
  
const fso: FileRep
} else if (fso.isDirectory()) {
  fso.children;
  
const fso: Directory
} else if (fso.isNetworked()) {
  fso.host;
  
const fso: Networked & FileSystemObject
}
```

this 기반의 타입 가드의 일반적인 사용 사례는 특정 필드의 지연 유효성 검사를 허용하는 것입니다. 예를 들어 아래의 경우 `hasValue`가 `true`로 확인되었을 때 상자 안에 있는 값에서 `undefined`를 제거합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661100518588/eOleZG0Em.png align="left")


## 매개변수 속성

TypeScript는 생성자 매개변수를 통해 클래스 속성으로 변환할 수 있는 특수 구문을 제공합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661100672311/0ygypJDok.png align="left")


## 클래스 표현식

클래스 표현식은 클래스 선언과 유사하지만 클래스 표현식에는 이름이 필요하지 않습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661100777284/6dIacxmJE.png align="left")


## `abstract` 클래스 및 멤버

TypeScript에서도 클래스, 메서드 및 필드를 추상으로 만들 수 있습니다.

> 추상 메서드 또는 필드는 구현이 제공되지 않은 것입니다. 이러한 멤버는 직접 인스턴스화 할 수 없는 추상 클래스 내부에 있어야 합니다.

추상 클래스의 역할은 모든 추상 멤버를 구현하는 하위 클래스의 기본 클래스 역할을 하는 것입니다. 클래스에서 추상 멤버가 없는 경우 이를 구체(concrete)라고 합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661100910710/bR2pEcoXX.png align="left")


구체가 없는 클래스는 인스턴스화 할 수 없습니다. 대신 파생 클래스로 추상 멤버를 구현합니다.

```typescript
class Derived extends Base {
  getName() {
    return "world";
  }
}
 
const d = new Derived();
d.printName();
```

베이스 클래스의 추상 멤버의 구현을 잊어버리면 오류가 발생합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661100974951/FH_-Vq4KI.png align="left")


### 추상 구성 서명

추상 클래스에서 파생된 클래스의 인스턴스를 생성하는 클래스 생성자 함수를 생각해봅시다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661101068186/MRb3wR09-.png align="left")


TypeScript에서는 추상 클래스를 인스턴스화 할 수 없으므로 오류를 올바르게 발생하지만 원하는 것은 아닙니다. 또한 아래의 코드는 합법적인 코드가 됩니다.

```typescript
// Bad!
greet(Base);
```

대신 구성 서명이 있는 무언가를 받아들이는 함수를 작성할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661101426669/_qYqptkeN.png align="left")

이제 TypeScript는 호출할 수 있는 클래스 생성자 함수에 대해 올바르게 알려줍니다. `Derived`는 구체적이기 때문에 호출할 수 있지만 `Base`는 호출할 수 없습니다.


##  클래스 간의 관계

대부분의 경우 TypeScript의 클래스는 다른 타입과 마찬가지로 구조적으로 비교됩니다. 이것은 다른 프로그래밍 언어의 특징과 다른 점입니다.

예를 들어, 이 두 클래스는 동일하기 때문에 서로 대신 사용할 수 있습니다.

```typescript
class Point1 {
  x = 0;
  y = 0;
}
 
class Point2 {
  x = 0;
  y = 0;
}
 
// OK
const p: Point1 = new Point2();
```

유사하게 명시적 상속이 없더라도 클래스 간의 하위 타입 관계는 존재합니다.

```typescript
class Person {
  name: string;
  age: number;
}
 
class Employee {
  name: string;
  age: number;
  salary: number;
}
 
// OK
const p: Person = new Employee();
```

이것은 간단하게 들리지만 다른 것보다 낯설게 보이는 몇 가지 경우가 있습니다.

빈 클래스에는 구성원이 없습니다. 구조적 타입 시스템에서 멤버가 없는 타입은 일반적으로 다른 것의 상위 유형입니다. 따라서 빈 클래스를 작성하면(하지 마세요!), 그 대신 아무 것도 사용할 수 있습니다.

```typescript
class Empty {}
 
function fn(x: Empty) {
  // can't do anything with 'x', so I won't
}
 
// All OK!
fn(window);
fn({});
fn(fn);
```
