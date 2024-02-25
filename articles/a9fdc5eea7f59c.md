---
title: "Hono + Cloudflare Workersでいい感じにpost cacheする"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hono", "cloudflare"]
published: true
---

この記事は、Hono Advent Calendar 2023 の 19 日目の記事です。

https://qiita.com/advent-calendar/2023/hono

最近、Cloudflare Workers や Fastly Compute@Edge などの Edge Computing のサービスで、より自由で柔軟なキャッシュができるようになりました、ね。
SSG そもそもいらないんじゃないかななんて思っています。

今回は、Next.js の文脈でよく聞くような ISR というキャッシュ戦略に似たようなものを Hono + Cloudflare Workers で実現してみます。

ISR の代表的な挙動として、

- 有効期限が切れても、キャッシュがある場合はそれを返す
- アクセスをトリガーに、有効期限が切れている場合はキャッシュを再生成する

というものがあります。
これらを模倣して、アクセス後にキャッシュを再生成する「post cache」を実現してみます。

イメージとしては、こんな感じです。

![flow](/images/a9fdc5eea7f59c/flow.png)

キャッシュが裏で再生成され、それまでの間古いキャッシュを返すことになるので、少しの不整合を許容する必要があります。
しかしそのかわり、どのタイミングでアクセスしてもキャッシュが返されるようになります。

## `c.executionCtx.waitUntil` を使う

Cloudflare Workers は基本 `fetch` ハンドラが終了すると (response が返されると) その後の処理が行われません。
（後述の比較動画を見ていただくとわかります）

`context.waitUntil` を使うことで、`fetch` ハンドラが終了してもその後の処理を 30 秒まで実行できます。

https://developers.cloudflare.com/workers/platform/limits/#duration

Hono ではこの API を `c.executionCtx.waitUntil` から使えます。

```ts
app.get("/no-wait-until", async (c) => {
  (async () => {
    await new Promise((resolve) => setTimeout(resolve, 1000));
    console.log("Waited 1 second");
  })();
  return c.text("Hello Hono!");
});

app.get("/wait-until", async (c) => {
  c.executionCtx.waitUntil(
    (async () => {
      await new Promise((resolve) => setTimeout(resolve, 1000));
      console.log("Waited 1 second");
    })()
  );
  return c.text("Hello Hono!");
});
```

![Wait Untilを使わない場合](/images/a9fdc5eea7f59c/no-wait-until.gif)

![Wait Untilを使った場合](/images/a9fdc5eea7f59c/wait-until.gif)

実際にこの動画のように、`waitUntil` を使わない場合は `fetch` ハンドラが終了するとその後の処理が行われません。
一方で `waitUntil` を使うと `fetch` ハンドラが終了しても渡された Promise が resolve されるまで続きます。

(Gif に変換した関係で 1 秒以上待っているように見えますが、実際は 1 秒です)

## キャッシュをKVに保存する

Cloudflare Workers でのキャッシュ先として

- `fetch` での `cf` パラメータからのキャッシュ
- Cache API でのキャッシュ
- `KV` へのキャッシュ

今回はエッジ間で共有できることが必要なので、エッジ間結果整合なストレージである `KV` へ保存することにします。

まずはキャッシュを定義しましょう。

```ts
interface CacheMetadata {
  expiresAt: number;
}

interface CacheResult {
  cache: string | null;
  isExpired: boolean;
}
```

`KV` には metadata という付加情報を保存できるので、HTML 文字列と一緒に有効期限を保存します。そのために、`CacheMetadata` という型を定義します。
`KV` 自体にも [expiring keys](https://developers.cloudflare.com/kv/api/write-key-value-pairs/#expiring-keys) を設定する機能があるのですが、これを使うと `KV` に保存されたデータが有効期限切れとなったときに、`KV` からデータが削除されてしまいます。今回は古いキャッシュを返すという挙動を実現するために、独自で有効期限を管理します。
また、`KV` から取得したキャッシュが有効期限切れかどうかを判定するために、`CacheResult` という型を定義します。

次に、キャッシュを取得する関数を定義します。

```ts
const getCache = async (
  kv: KVNamespace,
  pathname: string
): Promise<CacheResult> => {
  const { value, metadata } = await kv.getWithMetadata<CacheMetadata>(
    pathname,
    "text"
  );
  if (!value || !metadata) {
    return {
      cache: null,
      isExpired: true,
    };
  }
  return {
    cache: value,
    isExpired: metadata.expiresAt < Date.now(),
  };
};
```

`KV` からキャッシュを取得し、キャッシュが存在しない場合は `null` を返します。
また、キャッシュが存在する場合は、`expiresAt` が現在時刻よりも過去の場合は `isExpired` を `true` にします。

次に、キャッシュを生成して `KV` に保存する関数を定義します。

```ts
const createCache = async (
  kv: KVNamespace,
  origin: string,
  pathname: string,
  ttl: number
): Promise<void> => {
  const res = await fetch(origin + pathname);
  const data = await res.text();
  const expiresAt = Date.now() + ttl;
  await kv.put(pathname, data, {
    metadata: {
      expiresAt,
    },
  });
};
```

`fetch` でオリジンからデータを取得し、`KV` に保存します。
`KV` へ保存する際に `metadata.expiresAt` を `ttl` 秒後の時刻にして保存します。

## プロキシを作る

これらを使って、プロキシを作ります。

```ts
app.get("/*", async (c) => {
  const { pathname } = new URL(c.req.url);
  const { cache, isExpired } = await getCache(c.env.KV, pathname);
  if (isExpired) {
    c.executionCtx.waitUntil(createCache(c.env.KV, c.env.ORIGIN, pathname, 60));
  }
  if (cache) {
    return c.html(cache);
  }
  return await fetch(pathname);
});
```

環境変数 `ORIGIN` には、キャッシュする対象の URL を指定します。
前述の `c.executionCtx.waitUntil` を使って、キャッシュが有効期限切れの場合は、`createCache` を裏で再生成するようにします。

これでどれだけ SSR に時間がかかるページでも、ほんとに初回のアクセス（キャッシュ初回生成）以外は完全にキャッシュが返されるようになります。
一切オリジンに手を加えずして爆速なプロキシを作ることができました。

:::details コード全体

```ts
import { Hono } from "hono";

const app = new Hono<{
  Bindings: {
    KV: KVNamespace;
    ORIGIN: string;
  };
}>();

interface CacheMetadata {
  expiresAt: number;
}

interface CacheResult {
  cache: string | null;
  isExpired: boolean;
}

const getCache = async (
  kv: KVNamespace,
  pathname: string
): Promise<CacheResult> => {
  const { value, metadata } = await kv.getWithMetadata<CacheMetadata>(
    pathname,
    "text"
  );
  if (!value || !metadata) {
    return {
      cache: null,
      isExpired: true,
    };
  }
  return {
    cache: value,
    isExpired: metadata.expiresAt < Date.now(),
  };
};

const createCache = async (
  kv: KVNamespace,
  origin: string,
  pathname: string,
  ttl: number
): Promise<void> => {
  const res = await fetch(pathname);
  const data = await res.text();
  const expiresAt = Date.now() + ttl;
  await kv.put(pathname, data, {
    metadata: {
      expiresAt,
    },
  });
};

app.get("/*", async (c) => {
  const { pathname } = new URL(c.req.url);
  const { cache, isExpired } = await getCache(c.env.KV, pathname);
  if (isExpired) {
    c.executionCtx.waitUntil(createCache(c.env.KV, c.env.ORIGIN, pathname, 60));
  }
  if (cache) {
    return c.html(cache);
  }
  return await fetch(pathname);
});

export default app;
```

:::

注意点として、キャッシュ再生成のトリガーがユーザーのアクセスなので、もしユーザーがあまり訪問しないようなページだと、キャッシュが生成されないままかなり古いキャッシュを返されることになります。

キャッシュの TTL を規模に合わせたり、Cron を使ってユーザーのアクセスとは別に定期的にキャッシュを再生成するようにするなどの対策をするといいと思います。

## 最後に

ここまでやっといてアレなんですが、これを執筆しながら色々探してたら yusukebe さんが全く同じことをやっていて被ってしまいました。
こちらもぜひご覧ください。

https://yusukebe.com/posts/2022/dcs/
