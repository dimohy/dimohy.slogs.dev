## TypeScript 배우기 - 1. 시작

TypeScript 홈페이지에서 제공하는 [TypeScript 핸드북](https://www.typescriptlang.org/ko/docs/handbook/intro.html)을 통해 타입스크립트를 같이 배우도록 합시다.


## TypeScript 핸드북은?

일반 프로그래머가 TypeScript를 배울 수 있도록 돕는 종합 문서입니다. 학습으로 흐름에 맞게 문서의 왼쪽 메뉴를 위에서 아래로 전개하며 학습할 수 있으며 문서 하단의 단계 버튼을 통해 다음 단계로 넘어갈 수 있습니다.


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655960076550/gNsDEmbjd.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655960096553/ChhzprZCz.png align="left")

TypeScript 핸드북은 TypeScript 언어에 대한 완전한 설명서는 아니지만 모든 특징과 동작에 대한 종합적인 가이드 입니다. 핸드북은 다음의 학습 목표가 있습니다.

- 일반적으로 사용하는 TypeScript 구문 및 패턴을 읽고 이해할 수 있다.
- 중요한 컴파일러 옵션의 동작을 이해한다.
- 대부분의 타입 시스템 동작을 올바르게 이해한다.
- 간단한 함수, 객체 또는 클래스를 작성할 수 있다.

또한 참고할 수 있는 레퍼런스를 문서 메뉴를 통해 제공합니다. 학습을 진행하면서 좀 더 알아야 할 내용을 레퍼런스를 통해 살펴볼 수 있습니다.

핸드북의 작성 목적은 다음과 같습니다.

- 몇 시간 안에 편하게 읽을 수 있는 문서 (최대한 간결함을 유지 함)
- 언어 명세를 대체하는 목적이 아니라 대략적으로 쉽게  이해를 돕기 위한 목적
- 도구와의 상호작용에 대해 다루지 않음

이전 프로그램 경험에 따라 별도의 소개 문서를 제공하고 있습니다.

- [새로운 개발자를 위한 TypeScript](https://www.typescriptlang.org/docs/handbook/typescript-from-scratch.html)
- [JavaScript 개발자를 위한 TypeScript](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html)
- [OOP 프로그래머를 위한 TypeScript](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-oop.html)
- [함수형 프로그래머를 위한 TypeScript](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html)

또한 별도의 [Epub](https://www.typescriptlang.org/assets/typescript-handbook.epub) 및 [PDF](https://www.typescriptlang.org/assets/typescript-handbook.pdf) 형식의 파일 형태로도 핸드북을 제공하고 있습니다.

TypeScript가 무엇인지 이해하기 위해 [새로운 개발자를 위한 TypeScript](https://www.typescriptlang.org/docs/handbook/typescript-from-scratch.html) 문서를 통해 살펴보도록 합시다.


## TypeScript란?

TypeScript가 무엇인지 이해하기 위해서는 JavaScript의 문제에 대해 살펴볼 필요가 있습니다.

ECMAScript라고도 하는 JavaScript는 브라우저에서 동작하는 간단한 스크립트 언어로 시작되었습니다. 처음에는 웹 페이지에 필요한 짧은 코드 조각으로 사용될 것으로 예상 되었습니다. 하지만 웹 생태계가 성장하면서 복잡한 코드로 실행되게 되었고 최초 간단히 동작하는 코드로 설계되었던 다양한 동적 기능들로 인해 수 많은 오동작이 발생할 여지가 생겼습니다. 다음은 그런 코드의 예시 입니다.

```javascript
if ("" == 0) {
  // 이것은 참이다. 하지만 왜?
}
if (1 < x < 3) {
  // x가 어떠한 값이라도 참!
}
```

JavaScript에서는 존재하지 않는 속성에도 접근이 가능하므로 다음의 코드가 실행됩니다. 하지만 결과는 기대한 결과가 아니죠.

```javascript
const obj = { width: 10, height: 15 };
// 왜 이게 NaN이죠? 스펠링은 어려워요!
const area = obj.width * obj.heigth;
```

### TypeScript는 정적 유형 검사기 입니다.

이런 문제로 인해 TypeScript가 등장했습니다.  TypeScript는 JavaScript의 완전한 상위 집합(수퍼셋) 이지만 유형 검사를 동작 중에 수행하지 않고 컴파일 시점에서 정적 검사를 수행합니다. 정적 검사란 코드를 실행하지 않고 오류를 감지하는 것을 말합니다. 그리고 값의 종류에 따라 오류가 무엇인지 판별하는 것을 정적 유형 검사라 합니다.

TypeScript는 실행 전에 프로그램 오류가 있는지를 확인하고 값의 종류에 따라 수행하는 정적 유형 검사기 입니다. 예를 들어 마지막 예제는 obj의 유형에 의해 TypeScript가 오류를 발견합니다.

```
Property 'heigth' does not exist on type '{ width: number; height: number; }'. Did you mean 'height'?
```

#### 구문

TypeScript는 JavaScript의 상위 집합(수퍼셋) 입니다. 그러므로 JavaScript 구문은 정상적인 TypeScript 구문입니다. 즉 JavaScript 코드를 가져와서 TypeScript 파일에 넣을 수도 있습니다.

#### 유형

하지만 TypeScript는 유형이 존재하는 상위 집합이므로 다양한 종류의 유형에 대한 규칙을 추가합니다.  다음의 JavaScript는 `Infinity`를 반환하는 실행되는 코드이지만 TypeScript는 무의미한 코드로 간주하여 다음과 같은 오류를 발생합니다.

```
The right-hand side of an arithmetic operation must be of type 'any', 'number', 'bigint' or an enum type.
```


#### 런타임 동작

TypeScript는 JavaScript의 런타임 동작에 관여하지는 않습니다. JavaScript에서 0으로 나누면 `Infinity`가 되고 TypeScript에서는 이 런타임 동작을 변경하지는 않습니다. 즉 TypeScript를 JavaScript로 변환하더라도 이 동작을 다른 방식으로 동작하게 하지는 않습니다.

이는 TypeScript가 취하는 기본 약속입니다.


#### 지워진 유형

TypeScript의 컴파일러가 코드 검사를 마치면 컴파일된 결과 코드로 유형을 지웁니다. 즉, 완전한 JavaScript 코드가 생성됩니다. 이것으로 TypeScript가 유추한 것에 따라 프로그램 동작을 변경하지 않는 것을 의미합니다.


## 정리

앞으로 사용하게 될 TypeScript 핸드북과 간략하게 TypeScript에 대해 살펴보았습니다. 제가 생성하게 될 글은 아무래도 TypeScript 핸드북의 요약 자료 성질이 되어 좀 더 내용이 축소가 되고 저의 생각이 가미되므로 읽고 나서 다시 핸드북을 살펴 보기를 추천 합니다.

## 원본
- [핸드북에 대해서 (About this Handbook)](https://www.typescriptlang.org/ko/docs/handbook/intro.html)
- [새로운 개발자를 위한 TypeScript (TypeScript for the new Programmer)](https://www.typescriptlang.org/docs/handbook/typescript-from-scratch.html)