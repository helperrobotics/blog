---
title: [Next 14] Router Cache 무효화
author: Seungwon
date: 2024-11-06
category: front, next
layout: post
---

# Router Cache 무효화

## 문제점

현재 개발하고 있는 서비스에서 로그인 관리를 JWT를 활용하고 있습니다. 로그인을 하면 쿠키에 토큰을 저장하고, 로그아웃하면 쿠키에서 토큰을 삭제하도록 구현하였습니다.

모든 요청은 middleware에서 쿠키를 통해 accessToken이 있는지 검사하고, 이후 Next에서 제공하는 API Route에서도 쿠키를 통해 검사하고 있습니다.

하지만 로그아웃을 하고 뒤로 가기를 하게 되면 이전 페이지로 이동이 가능한 버그를 발견하게 되었습니다.

## BFCache

BFCache는 브라우저에서 이전/다음 페이지 전체를 메모리에 캐싱하는 메커니즘입니다. 이를 통해 뒤로 가기 버튼을 누르게 되면 이전 페이지를 빠르게 불러올 수 있습니다. 하지만 저희 서비스에서는 로그아웃 이후에는 이 기능이 활성화 되어 있으면 안되기 때문에 이를 무효화할 수 있는 방향으로 찾아보았습니다.

BFCache는 간단히 HTML에 meta 태그에 http-equiv 속성을 통해 무효화할 수 있습니다. 이 내용을 바탕으로 Next 14 App Router 방식에서 어떻게 meta 태그를 만들 수 있는 지 확인해보았습니다.

하지만 Next에서 meta 태그의 http-equiv 속성을 지원하지 않는다는 사실을 알게되었습니다.

```
Use appropriate HTTP Headers via redirect(), Middleware, Security Headers
```

이렇게 지원하지 않는다면 이 문제점이 BFCache 때문은 아닐 것이라고 생각했고, Next 14 App Router에서는 어떻게 캐싱을 하고 있는지 찾아보았습니다.

Next는 총 네가지 방식[(Request Memoization, Data Cache, Full Route Cache, Router Cache)](https://nextjs.org/docs/app/building-your-application/caching#overview)으로 캐싱을 하고 있는 것을 확인했습니다. 이중에서 오직 Router Cache만 클라이언트에서 동작하는 것에 주목했습니다.

## Router Cache

```
Next.js has an in-memory client-side router cache that stores the RSC payload of route segments, split by layouts, loading states, and pages.
```

이 문장을 통하여 Next가 in-memory client-side router cache를 가지고 있으며, 페이지를 대상으로 캐싱을 하고 있다는 것을 확인할 수 있습니다.

이 router cache는 간단한 방법으로 클라이언트에서 무효화할 수 있는데, 바로 'router.refresh'를 사용하는 것입니다. 이는 새로고침과는 다르게 동작하며 페이지를 새로 렌더링 하는 것이 아니라 현재 경로에 대하여 새로운 요청을 서버한테 보내고 router cache를 초기화 합니다.

## 해결

위이 내용을 토대로 로그아웃 로직에서 현재 경로에 대해 'router.refresh'를 동작하게 하여, 로그인 페이지로 이동 후, 이전 페이지로 접근하면 새로운 데이터를 서버한테 요청하게 됩니다. 그 과정에서 쿠키에서 토큰을 확인하여 비정상적인 접근임을 확인할 수 있게됩니다.

```javascript
const logout = async () => {
  const { status } = await axios.get("/logout");
  if (status === 200) {
    router.refresh();
    router.push("/login");
  }
};
```
