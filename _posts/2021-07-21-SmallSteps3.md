---
title: "작은 발자국 프론트엔드 부분 만들기1"
excerpt: "vue.js, 컴포넌트랑 라우터 공부하기"
categories:
    - smallstep
tags :
    - smallstep
    - Vue.js
    - TIL
last-modified-at: 2021-07-21 T17:43
---

# frontend

### 0210721 라우터 기능 추가

- 프론트엔드를 업데이트했다.
- 컴포넌트에 사용 방법에 대해 공부했다.
- 라우터 기능을 활용하여 페이지 이동하는 방법을 익힘



__Intro__


![Intro](/assets/pagesModified/20210721Intro.PNG)



__Home__

![Home](/assets/pagesModified/20210721home.PNG)


__Intro__


![성장일지](/assets/pagesModified/20210721성장일지.PNG)


```html

<!-- App.vue -->
<template>
  <div id="app">
      <div id="content" class="content">
				<!-- 주소 매핑에 따라 다른 페이지를 띄운다 -->
        <router-view></router-view>
				<!-- Footer 컴포넌트는 항상 밑에 고정시킬 것이기 때문에 -- >
      <Footer msg="Copyright 2021. Hyunalee9"/>  
    </div>
  </div>
</template>
```

```js

//index.js
const routes = [
  {
    path: '/',
    component : Intro  // 이런식으로 컴포넌트와 맵핑

  },
  {
    path: '/home',
    component: Home
  },
  {
    path: '/about',
    component: About
  },

]


```
