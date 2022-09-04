---
title: "【小ネタ】定数オブジェクトのプロパティを再起探索してユニオン型を作る・応用例紹介"
emoji: "🐱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

# 結論
  
## 原案

```ts
type ObjectKeyDeep<T> = T extends object
  ? { [K in keyof T]: ObjectKeyDeep<T[K]> }[keyof T]
  : T;
```

## 改良版

```ts
type ObjectKeyDeep<T> = T extends Record<string, infer U> ? ObjectKeyDeep<U> : T;
```

# 応用例

## RouterやStoreなどの文字列定数をキーとして使う関数を型安全にする

（Vueの例です）

- pushのラップ関数を作る

```ts:router.ts
const ROUTE_NAMES = {
  HOME: "home",
  ABOUT: "about",
  BLOG: {
    LIST: "Blog.list",
    DETAIL: "Blog.Detail",
  }
} as const;

new VueRouter({
  routes: [
    {
      path: "/",
      name: ROUTE_NAMES.HOME,
      component: Home,
    },
    {
      path: "/about",
      name: ROUTE_NAMES.ABOUT,
      component: About,
    },
    {
      path: "/blog",
      name: ROUTE_NAMES.BLOG.LIST,
      component: BlogList,
    },
    {
      path: "/blog/:id",
      name: ROUTE_NAMES.BLOG.DETAIL,
      component: BlogDetail,
    },
  ],
});

type RouteName = ObjectKeyDeep<typeof ROUTE_NAMES>;

const typedPush = (name: RouteName, params?: Record<string, string>) =>
  router.push({ name, params });

typedPush(ROUTE_NAMES.BLOG.DETAIL); // OK
typedPush("blog.list"); // OK
typedPush("hogehoge"); // NG (Type '"hogehoge"' is not assignable to type 'RouteName'.)
```

Routerの第一引数の`Location`型は`name`の型が`string`なので、`RouteName`でdeclareしたほうが賢明...？

以上しょぼすぎるけど割と実務で使えそうなやつでした。
