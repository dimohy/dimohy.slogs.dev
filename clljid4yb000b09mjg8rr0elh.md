---
title: "url 파라미터 받기"
datePublished: Sun Aug 20 2023 13:52:58 GMT+0000 (Coordinated Universal Time)
cuid: clljid4yb000b09mjg8rr0elh
slug: vue-url-params
tags: vuejs, typescript

---

url 주소의 특정 값을 취해야 할 때가 있습니다. 다음의 예에서

`/page/1?id=test#abcd`

`1`, `id`, `#abcd`를 취하는 방법입니다.

## 파라미터

`/page` 경로의 뷰가 있다고 할 때 `/page/1`일 경우 `1`은 어떻게 취해야 할까요?

먼저 특별한 설정을 하지 않으면 `/page`와 `/page/1`은 다른 경로이므로 같은 뷰를 바라보게 라우트 설정을 해야 합니다.

| index.ts

```typescript
...
    {
      path: '/page/:id?',
      name: 'page',
      component: () => import('../views/PageView.vue')
    },
```

이제 vue에서 `route.params`로 `id`값에 접근할 수 있게 됩니다.

| PageView.vue

```xml
<script setup lang="ts">
import { useRoute } from 'vue-router'

const route = useRoute()

const { id } = route.params
...
```

## 쿼리

`/page?id=test` 에서 `id`값은 `route.query` 에서 얻을 수 있습니다.

```javascript
const { test } = route.query
```

## 해시

`/page#abcd`에서 `#abcd`는 `route.hash`에 저장 됩니다.