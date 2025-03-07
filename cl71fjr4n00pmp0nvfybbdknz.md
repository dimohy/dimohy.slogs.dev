---
title: "TypeScript 배우기 - 15. 모듈"
datePublished: Sat Aug 20 2022 04:58:24 GMT+0000 (Coordinated Universal Time)
cuid: cl71fjr4n00pmp0nvfybbdknz
slug: typescript-15
tags: typescript

---

오늘은 TypeScript의 모듈 시스템을 핸드북의 내용을 통해 살펴볼 것입니다.

> 대부분의 내용은 핸드북 [Modules](https://www.typescriptlang.org/docs/handbook/2/modules.html) 내용이 번역된 것입니다.

JavaScript 모듈 (ES 모듈)은 JavaScript 사양에 추가가 되어 2020년까지 대부분의 웹 브라우저와 JavaScript 런타임에서 사용할 수 있습니다.

TypeScript는 2012년부터 이런 형식을 지원했지만 시간이 지남에 따라 JavaScript 사양은 ES 모듈(ES6 모듈)이라는 `import` / `export` 구문 형식으로 정리가 되었습니다.

핸드북의 내용은 ES 모듈과 널리 사용되는 CommonJS 구문을 모두 다룹니다.


## JavaScript 모듈 정의 방법

TypeScript에서는 ECMAScript 2015와 마찬가지로 최상위 수준의 Import 또는 Export가 포함된 파일은 모듈로 간주합니다.

반면에, 최상위 가져오기 또는 내보내기 선언이 없는 파일은 해당 내용이 전역 범위(따라서 모듈에서도)에서 사용 가능한 스크립트로 간주합니다.

모듈은 전역이 아니며 자체 범위에서 실행됩니다. 즉, 모듈에서 선언된 변수나 함수, 클래스 등은 내보내기를 하지 않는 이상 모듈 외부에서 사용할 수 없습니다. 반대로 다른 모듈에서 내보낸 변수, 함수, 클래스, 인터페이스 등을 사용하려면 가져오기 형식 중 하나를 사용하여 가져와야 합니다.


## 비모듈

JavaScript에서는 `export` 또는 최상위 `await`가 없는 JavaScript 파일을 모듈이 아니라 스크립트로 간주합니다.

스크립트 파일의 변수와 타입은 전역 범위에 선언되며 [outFile](https://www.typescriptlang.org/tsconfig/#outFile) 컴파일러 옵션으로 여러 입력 파일을 하나의 출력 파일로 결합하거나 HTML에서 여러 `<script>` 태그를 사용한다고 가정합시다.

현재 `import` 또는 `export`가 없는 파일이 있지만 모듈로 처리하려면 다음 행을 추가합니다.

```typescript
export {};
```

파일을 아무것도 내보내지 않는 모듈로 변경하게 됩니다. 이 구문은 `import`와 `export`가 없으므로 모듈 구성과 상관없이 동작합니다.


## TypeScript에서의 모듈

TypeScript에서 모듈 기반 코드를 작성할 때 고려할 세가지 주요 사항이 있습니다.

- **구문**:  가져오기 및 내보내기에 어떤 구문을 사용할까요?
- **모듈 해석**: 모듈 이름 또는 경로와 파일 사이의 관계는 무엇인가요?
- **모듈 출력 대상**: 만들어진 JavaScript 모듈은 어떤 모양입니까?


### ES 모듈 구문

파일은 `export default`로 선언할 수 있습니다.

```typescript
// @filename: hello.ts
export default function helloWorld() {
  console.log("Hello, world!");
}
```

다음을 통해 가져옵니다.

```typescript
import helloWorld from "./hello.js";
helloWorld();
```

`export default`이외에도 변수와 함수를 둘 이상 내보낼 수 있습니다.

```typescript
// @filename: maths.ts
export var pi = 3.14;
export let squareTwo = 1.41;
export const phi = 1.61;
 
export class RandomNumberGenerator {}
 
export function absolute(num: number) {
  if (num < 0) return num * -1;
  return num;
}
```

이제 다른 파일에서 `import`를 통해 사용할 수 있습니다.

```typescript
import { pi, phi, absolute } from "./maths.js";
 
console.log(pi);
const absPhi = absolute(phi);
```


### 추가적인 가져오기 구문

`import { old as new }`로 이름을 변경하여 가져올 수 있습니다.

```typescript
import { pi as π } from "./maths.js";
 
console.log(π);
```

이것은 하나의 `import` 로 믹스한 구문으로도 가져올 수 있습니다.

```typescript
// @filename: maths.ts
export const pi = 3.14;
export default class RandomNumberGenerator {}
 
// @filename: app.ts
import RandomNumberGenerator, { pi as π } from "./maths.js";
 
RandomNumberGenerator;
         
(alias) class RandomNumberGenerator
import RandomNumberGenerator
 
console.log(π);
```

또한 내보낸 모든 객체를 `* as name`으로 한번에 단일 네임스페이스로 넣을 수 있습니다.

```typescript
// @filename: app.ts
import * as math from "./maths.js";
 
console.log(math.pi);
const positivePhi = math.absolute(math.phi);
```

`import ".file"`을 통해 변수를 포함하지 않은 파일을 가져올 수도 있습니다.

```typescript
// @filename: app.ts
import "./maths.js";
 
console.log("3.14");
```

이런 경우 아무런 `import`를 하지 않습니다. 하지만 `maths.ts`는 평가 되기 때문에 다른 객체에 사이드 이펙트를 줄 수도 있게 됩니다.


#### 특수한 TypeScript ES 모듈 구문

JavaScript 값과 동일한 구문으로 타입을 내보내고 가져올 수 있습니다.

```typescript
// @filename: animal.ts
export type Cat = { breed: string; yearOfBirth: number };
 
export interface Dog {
  breeds: string[];
  yearOfBirth: number;
}
 
// @filename: app.ts
import { Cat, Dog } from "./animal.js";
type Animals = Cat | Dog;
```

TypeScript는 타입 가져오기를 선언하기 위해 두 가지 개념으로 `import` 구문을 확장했습니다.


##### import type

다음은 타입만 가져오는 `import`문 입니다.

```typescript
// @filename: animal.ts
export type Cat = { breed: string; yearOfBirth: number };
'createCatName' cannot be used as a value because it was imported using 'import type'.
export type Dog = { breeds: string[]; yearOfBirth: number };
export const createCatName = () => "fluffy";
 
// @filename: valid.ts
import type { Cat, Dog } from "./animal.js";
export type Animals = Cat | Dog;
 
// @filename: app.ts
import type { createCatName } from "./animal.js";
const name = createCatName();
```


##### 가져오기에 type 포함

TypeScript 4.5에서는 가져온 참조가 타입임을 나타내기 위해 개별 가져오기에서 접두사를 붙일 수 있습니다.

```typescript
// @filename: app.ts
import { createCatName, type Cat, type Dog } from "./animal.js";
 
export type Animals = Cat | Dog;
const name = createCatName();
```

이것들을 함께 사용하면 Babel, swc 또는 esbuild과 같은 비 TypeScript 변환기가 안전하게 제거할 수 있는 가져오기를 할 수 있습니다.


#### CommonJS 동작이 포함된 ES 모듈 구문

TypeScript에는 CommonJS 및 AMD `require`와 직접적으로 관련된 ES 모듈 구문이 있습니다. ES 모듈을 사용한 가져오기는 대부분의 경우 해당 환경에서 요구하는 것과 동일하지만 이 구문을 사용하면 TypeScript 파일에서 CommonJS 출력과 1:1 일치를 얻을 수 있습니다.

```typescript
import fs = require("fs");
const code = fs.readFileSync("hello.ts", "utf8");
```


## CommonJS 구문

CommonJS는 npm에 있는 대부분의 모듈이 전달되는 형식입니다. 위의 ES 모듈 구문을 사용하여 작성하는 경우에도 CommonJS 구문의 작동 방식을 간략히 이해하면 디버깅이 더 쉬워집니다.


### 내보내기

식별자는 전역의 `module`의 `exports` 속성을 설정하여 내보냅니다.

```typescript
function absolute(num: number) {
  if (num < 0) return num * -1;
  return num;
}
 
module.exports = {
  pi: 3.14,
  squareTwo: 1.41,
  phi: 1.61,
  absolute,
};
```

그런 다음 다음 파일을 `require`문을 통해 이러한 파일을 가져올 수 있습니다.

```typescript
const maths = require("maths");
maths.pi;
```

또는 JavaScript의 구조 분해를 통해 단순화 할 수 있습니다.

```typescript
const { squareTwo } = require("maths");
squareTwo;
```


### CommonJS 및 ES 모듈 상호 운용

기본 가져오기와 모듈 네임스페이스 개체 가져오기 간의 구별과 관련하여 CommonJS와 ES 모듈 간의 기능에 불일치가 있습니다. TypeScript에는 [esModuleInterop](https://www.typescriptlang.org/tsconfig/#esModuleInterop)을 사용하여 서로 다른 두 가지 제약 조건 세트 간의 마찰을 줄이기 위한 컴파일러 플래그가 있습니다.


## TypeScript의 모듈 해석 옵션

모듈 확인은 `import` 또는 `require`문에서 문자열을 가져와서 해당 문자열이 참조하는 파일을 결정하는 프로세스입니다.

TypeScript에는 Classic 및 Node.js의 두 가지 해결 전략이 포함되어 있습니다. 컴파일러 옵션 [module](https://www.typescriptlang.org/tsconfig#module) `commonjs`가 아닌 경우 기본값인 Classic은 이전 버전과의 호환성을 위해 포함됩니다. Node 전략은 `.ts` 및 `.d.ts`에 대한 추가 검사와 함께 Node.js가 CommonJS 모드에서 작동하는 방식을 복제합니다.

TypeScript 내에서 모듈 전략에 영향을 미치는 많은 TSConfig 플래그: [moduleResolution](https://www.typescriptlang.org/tsconfig#moduleResolution), [baseUrl](https://www.typescriptlang.org/tsconfig#baseUrl), [paths](https://www.typescriptlang.org/tsconfig#paths), [rootDirs](https://www.typescriptlang.org/tsconfig#rootDirs).


## TypeScript의 모듈 출력 옵션

내보낸 JavaScript 출력에 영향을 주는 두 가지 옵션:

- 어떤 JS 기능이 하향 조정되고(이전 JavaScript 런타임에서 실행되도록 변환됨) 그대로 남아 있는지를 결정하는 [target](https://www.typescriptlang.org/tsconfig#target)

- 모듈이 서로 상호 작용하는 데 사용되는 코드를 결정하는 [module](https://www.typescriptlang.org/tsconfig#module)

사용하는 [target](https://www.typescriptlang.org/tsconfig#target)은 TypeScript 코드를 실행할 것으로 예상되는 JavaScript 런타임에서 사용할 수 있는 기능에 따라 결정됩니다. 지원하는 가장 오래된 웹 브라우저, 실행될 것으로 예상되는 가장 낮은 버전의 Node.js 또는 출처가 될 수 있습니다 - 예를 들어 Electron과 같은 런타임의 고유한 제약 조건

모듈 간의 모든 통신은 모듈 로더를 통해 이루어지며 컴파일러 옵션 [module](https://www.typescriptlang.org/tsconfig#module)은 어느 것이 사용되는지 결정합니다. 런타임에 모듈 로더는 모듈을 실행하기 전에 모듈의 모든 종속성을 찾아 실행하는 역할을 합니다.

예를 들어, 다음은 [module](https://www.typescriptlang.org/tsconfig#module)에 대한 몇 가지 다른 옵션을 보여주는 ES Modules 구문을 사용하는 TypeScript 파일입니다.

```typescript
import { valueOfPi } from "./constants.js";
 
export const twoPi = valueOfPi * 2;
```

### ES2020

```typescript
import { valueOfPi } from "./constants.js";
export const twoPi = valueOfPi * 2;
```

### UMD

```typescript
(function (factory) {
    if (typeof module === "object" && typeof module.exports === "object") {
        var v = factory(require, exports);
        if (v !== undefined) module.exports = v;
    }
    else if (typeof define === "function" && define.amd) {
        define(["require", "exports", "./constants.js"], factory);
    }
})(function (require, exports) {
    "use strict";
    Object.defineProperty(exports, "__esModule", { value: true });
    exports.twoPi = void 0;
    const constants_js_1 = require("./constants.js");
    exports.twoPi = constants_js_1.valueOfPi * 2;
});
```

> ES2020은 원 index.ts와 사실상 동일합니다.

[모듈에 대한 TSConfig 참조](https://www.typescriptlang.org/tsconfig#module)에서 사용 가능한 모든 옵션과 생성된 JavaScript 코드가 어떻게 보이는지 확인할 수 있습니다.


## TypeScript 네임스페이스

TypeScript에는 ES 모듈 표준보다 앞선 네임스페이스라는 자체 모듈 형식이 있습니다. 이 구문에는 복잡한 정의 파일을 생성하는 데 유용한 기능이 많이 있으며, 여전히 [DefinitelyTyped](https://www.typescriptlang.org/dt/)에서 활발하게 사용됩니다. 더 이상 사용되지는 않지만 네임스페이스의 대부분의 기능은 ES 모듈에 존재하며 JavaScript의 방향에 맞게 이를 사용하는 것이 좋습니다. 네임스페이스 참조 페이지에서 네임스페이스에 대해 자세히 알아볼 수 있습니다.

