---
title: "엘리먼트 참조"
datePublished: Sun Aug 20 2023 04:04:47 GMT+0000 (Coordinated Universal Time)
cuid: cllixcqbk000609l574ye7qx3
slug: 7jey66as66i87yq4ioywuoyhsa
tags: vuejs, typescript

---

input 태그로 포커스는 주는 등 엘리먼트를 직접 참조해야 할 필요가 있습니다.

Vue3 + TypeScript에서는 `ref` 속성을 이용해서 가능합니다.

```xml
<input ref="code1" class="form-control" type="text" @input="onTextInput($event)" />
```

참조할 엘리먼트에 ref 속성을 부여한 후 ref 이름과 동일한 `ref`를 `script setup`에서 만들어 줍니다.

```typescript
<script setup lang="ts">
import { ref } from 'vue'

const code1 = ref<HTMLInputElement>()
...
```

`const code1 = ref(null)`로도 사용 가능하지만 TypeScript의 인텔리센스를 사용할 수 없습니다. 이후 이벤트 등에서 `code1.value?.focus()`등으로 사용 가능해 집니다.

```typescript
const onTextInput = function (event: any) {
    if (event.target.value != '')
        event.target.value = ''

    const c = event.data

    if (c >= 'a' && c <= 'z')
        event.target.value = c.toUpperCase()
    else if (c >= '0' && c <= '9')
        event.target.value = c

    if (event.target == code1.value)
        code2.value?.focus()
...
```