---
title: "[PostMan] 환경변수 활용하기"
author: scorchedrice
date: 2024-12-30 18:00:00 +0900
categories: [Tools, PostMan]
tags: [postman, environment, token]
render_with_liquid: false
mermaid: true
---

postman을 사용하면서 이런 생각을 한 적 있다.

***언제까지 토큰을 복사하고 .. 넣고 .. 귀찮게 이렇게 써야만할까?***

이런 문제들을 환경변수를 활용해서 해결할 수 있다.

# 환경변수의 장점 (postman)

<img src="/assets/img/tools/postman/241230/postman_env_host.png" alt="postman-host_env">

<img src="/assets/img/tools/postman/241230/postman_env_ex1.png" alt="postman_ex_1">

postman에서 환경 변수를 설정하면 해당 주소가 변경된다면 내가 등록한 요청들의 주소를 모두 바꿀 필요가 없기에 유용한 기능이다.

# test를 활용한 토큰 환경변수화

postman에서 그냥 환경변수를 등록하는 것 그 자체도 매우 유용하지만, 테스트 기능을 활용한다면 더욱 유용하게 postman을 활용할 수 있다.
요청을 보내고 받은 응답을 활용하여 환경변수에 등록하는 것이 가능하기 때문이다.

매번 accessToken을 복사하고~ 이런 과정이 필요 없다는 것이다.

토큰을 환경변수에 등록하는 과정을 예로 이를 배워보자.

## 토큰을 환경변수 값으로 받아오는 스크립트 작성 과정

1. 토큰의 등록

일단, 빈 값을 가지는 환경변수를 등록해야한다. accessToken과 refreshToken을 등록해보자.

2. script작성

응답이 올 때 실행될 스크립트를 작성해야한다.

<img src="/assets/img/tools/postman/241230/postman_env_script.png" alt="postman_script">

```js
pm.test('Store access token', function() {
    pm.environment.set('accessToken', pm.response.json().accessToken);
});

pm.test('Store refresh token', function() {
    pm.environment.set('refreshToken', pm.response.json().refreshToken);
});
```

postman은 javaScript로 동작한다. 여기서 pm은 postman이 제공하는 모듈명이고 위에 작성된 코드는 환경변수 'accessToken', 'refreshToken'을 각각 json 형식 응답의 accessToken, refreshToken 값으로 설정하겠다는 것!

<img src="/assets/img/tools/postman/241230/postman_env_after_response.png" alt="postman_after_res">

로그인 요청을 보낸 이후 정상적으로 환경변수 값이 할당됨을 확인할 수 있다.

# Auth 활용

Auth 메뉴를 활용한다면 Base64로 변환해야하는 과정, 환경변수에 저장된 token 사용을 간단하게 할 수 있다.

## Basic

<img src="/assets/img/tools/postman/241230/postman_auth_basic.png" alt="postman_basic">

하단 console 창을 보면 Authorization: "Basic ..." 으로 요청이 갔다는 것을 알 수 있다.

Header에 직접 넣지 않아도 아이디와 비밀번호만 입력해둔다면 자동으로 전환되어 요청을 보낸다.

## Access & Refresh

바로 직전 테스트를 활용해 토큰을 환경변수에 저장하는 과정을 진행했다.

환경변수에 저장되어 있으니, 그대로 쓰기만하면 된다.

<img src="/assets/img/tools/postman/241230/postman_auth_bearer.png" alt="postman_bearer">

console 창에 정상적으로 요청이 갔음을 확인할 수 있다.

# 결론

왜 postman이 개발자들이 가장 많이 쓰는 개발도구라고 하는지 알거같다. 단순히 API요청을 보내는 프로그램으로 알고 있었으나, 유용한 기능들이 정말 많음을 깨달았다.

