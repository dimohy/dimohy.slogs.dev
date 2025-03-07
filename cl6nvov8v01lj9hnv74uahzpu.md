---
title: "TypeScript 배우기 - 13. 템플릿 리터럴 타입"
datePublished: Wed Aug 10 2022 17:21:30 GMT+0000 (Coordinated Universal Time)
cuid: cl6nvov8v01lj9hnv74uahzpu
slug: typescript-13
tags: typescript

---

핸드북 타입 조작 섹션의 마지막인 `템플릿 리터럴 타입`입니다. 템플릿 리터럴 타입은 문자열 리터럴 타입을 기반으로 유니온 타입을 통해 확장됩니다.

다음의 코드를 보시죠.

```typescript
type World = "world";
 
// Greeting 타입은 "hello world" 문자열 리터럴 타입
type Greeting = `hello ${World}`;
```

보간된 위치에서 유니온 타입을 사용할 수 있습니다.

```typescript
type EmailLocaleIDs = "welcome_email" | "email_heading";
type FooterLocaleIDs = "footer_title" | "footer_sendoff";
 
// 유니온 결합에 의해 `"welcome_email_id" | "email_heading_id" | "footer_title_id" | "footer_sendoff_id"`으로 확장
type AllLocaleIDs = `${EmailLocaleIDs | FooterLocaleIDs}_id`;
```

다음과 같이 두 개의 유니온 타입이 교차 곱해집니다.

```typescript
type AllLocaleIDs = `${EmailLocaleIDs | FooterLocaleIDs}_id`;
type Lang = "en" | "ja" | "pt";
 
// "en_welcome_email_id" | "en_email_heading_id" | "en_footer_title_id" | "en_footer_sendoff_id" | "ja_welcome_email_id" | "ja_email_heading_id" | "ja_footer_title_id" | "ja_footer_sendoff_id" | "pt_welcome_email_id" | "pt_email_heading_id" | "pt_footer_title_id" | "pt_footer_sendoff_id"
type LocaleMessageIDs = `${Lang}_${AllLocaleIDs}`;
```

일반적으로 큰 문자열 합집합의 경우 사전을 사용해야 하지만 작은 문자열 합집합의 경우에 유용합니다.


### 타입 문자열 합집합

템플릿 리터럴 타입은 타입 내부의 정보를 이용해서 새로운 문자열을 정의할 때 유용합니다.

다음의 객체 타입의 속성이 변경될 때 특정한 동작을 하기를 원한다고 합시다.
```typescript
const passedObject = {
  firstName: "Saoirse",
  lastName: "Ronan",
  age: 26,
};
```

이 때 필요한 것은 변경 이벤트 이름과 이 이벤트 이름으로 콜백 함수를 등록한 두개의 매개변수가 필요합니다. 여기서 변경 이벤트 이름은 `firstNameChanged`, `lastNameChanged`, `ageChanged`가 되게 하고 싶습니다.

이를 다음처럼 표현할 수 있습니다.

```typescript
const person = makeWatchedObject({
  firstName: "Saoirse",
  lastName: "Ronan",
  age: 26,
});
 
// makeWatchedObject has added `on` to the anonymous Object
 
person.on("firstNameChanged", (newValue) => {
  console.log(`firstName was changed to ${newValue}!`);
});
```

`makeWatchedObject()` 함수에 의해 주어진 매개 변수의 객체에 추가로 `속성명` + `Changed`의 이벤트가 생성되고 속성이 변경될 때 콜벡 함수가 실행되도록 하는 `on()` 함수가 추가됩니다.

```typescript
type PropEventSource<Type> = {
    on(eventName: `${string & keyof Type}Changed`, callback: (newValue: any) => void): void;
};
 
/// Create a "watched object" with an 'on' method
/// so that you can watch for changes to properties.
declare function makeWatchedObject<Type>(obj: Type): Type & PropEventSource<Type>;
```

Type의 모든 속성에 대한 이벤트명으로 유니언 리터럴 타입으로 생성하는 코드를 볼 수 있습니다.
그리고 `makeWatchedObject()` 함수는 반환형으로 `Type`과 `PropEventSource<Type>` 타입을 결합합니다.

이제 Type의 속성명으로 생성된 이벤트명의 유니온 리터럴 타입에 의해 다음의 코드에서 올바르지 않은 이벤트명은 오류로 처리할 수 있게 되었습니다.

```typescript
const person = makeWatchedObject({
  firstName: "Saoirse",
  lastName: "Ronan",
  age: 26
});
 
person.on("firstNameChanged", () => {});

// Prevent easy human error (using the key instead of the event name)
person.on("firstName", () => {});
>          ~~~~~~~~~
> Argument of type '"firstName"' is not assignable to parameter of type '"firstNameChanged" | "lastNameChanged" | "ageChanged"'.

// It's typo-resistant
person.on("frstNameChanged", () => {});
>          ~~~~~~~~~~~~~~~
> Argument of type '"frstNameChanged"' is not assignable to parameter of type '"firstNameChanged" | "lastNameChanged" | "ageChanged"'.
```


### 템플릿 리터럴을 사용한 추론

하지만 위의 예시는 매개변수의 타입에 `any`를 사용했으므로 완전하지 않습니다. 

newValue의 타입이 속성의 타입이 되도록 다음과 같이 수정할 수 있습니다.

```typescript
type PropEventSource<Type> = {
    on<Key extends string & keyof Type>
        (eventName: `${Key}Changed`, callback: (newValue: Type[Key]) => void ): void;
};
```

```typescript
const person = makeWatchedObject({
  firstName: "Saoirse",
  lastName: "Ronan",
  age: 26
});
 
// (parameter) newName : string
person.on("firstNameChanged", newName => {
    console.log(`new name is ${newName.toUpperCase()}`);
});

// (parameter) newAge: number
person.on("ageChanged", newAge => {
    if (newAge < 0) {
        console.warn("warning! negative age");
    }
});
```

추론은 다양한 방식으로 결합될 수 있으며, 종종 문자열을 분해하고 다양한 방식으로 재구성할 수 있습니다.


## 고유 문자열 조작 타입

문자열 조작을 돕기 위해 TypeScript는 문자열 조작에 필요한 타입 집합이 포함되어 있습니다. 이러한 타입은 성능을 위해 컴파일러에 내장되어 있으며 `.d.ts` 파일로는 찾을 수 없습니다.

#### `Uppercase<StringType>`

대문자 문자열 리터럴 타입으로 변환합니다.

```typescript
type Greeting = "Hello, world"
// type ShoutyGreeting = "HELLO, WORLD"
type ShoutyGreeting = Uppercase<Greeting>
 
type ASCIICacheKey<Str extends string> = `ID-${Uppercase<Str>}`
// type MainID = "ID-MY_APP"
type MainID = ASCIICacheKey<"my_app">
```

#### `Lowercase<StringType>`

소문자 문자열 리터럴 타입으로 변환합니다.

```typescript
type Greeting = "Hello, world"
// type QuietGreeting = "hello, world"
type QuietGreeting = Lowercase<Greeting>
 
type ASCIICacheKey<Str extends string> = `id-${Lowercase<Str>}`
// type MainID = "id-my_app"
type MainID = ASCIICacheKey<"MY_APP">
```

#### `Capitalize<StringType>`

문자열 리터럴 타입의 첫번째 문자를 대문자로 변환합니다.

```typescript
type LowercaseGreeting = "hello, world";
// type Greeting = "Hello, world"
type Greeting = Capitalize<LowercaseGreeting>;
```

#### `Uncapitalize<StringType>`

문자열 리터럴 타입의 첫번째 문자를 소문자로 변환합니다.

```typescript
type UppercaseGreeting = "HELLO WORLD";
// type UncomfortableGreeting = "hELLO WORLD"
type UncomfortableGreeting = Uncapitalize<UppercaseGreeting>;
```
