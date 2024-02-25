---
title: "Honoを使ってCloudflare Multi Workers環境を作る"
emoji: "🌐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Cloudflare", "Hono"]
published: true
---

## はじめに

複数の Cloudflare Workers 間を Cloudflare Network 内で通信できる Service Bindings の機能を使って、マイクロサービスの環境を作ります。
外部に露出させる gateway アプリが 1 つ、内部にある複数の microservice アプリが複数ある構成です。

先に成果物を貼っておきます。
https://github.com/sor4chi/workers-service-bindings-monorepo-sample

:::message
このコードは起動すると
"services" fields are experimental and may change or break at any time.
という警告が出ます。Experimental な機能を使っているので、使用する際は注意してください。

https://developers.cloudflare.com/workers/wrangler/configuration/#non-inheritable-keys
:::

## Monorepo環境を構築

今回は、複数の Cloudflare Workers を 1 つのリポジトリで管理するために、Monorepo を作ります。
今回は pnpm と turborepo を使いますが、yarn や npm を使ったり、lerna などのツールを使っても構いません。

パッケージ共通のツールはプロジェクトルートにインストールしておきます。

```json:package.json
{
  "devDependencies": {
    "turbo": "^1.10.9",
    "wrangler": "^3.1.2"
  },
  "scripts": {
    "dev": "turbo dev"
  }
}
```

dev でパッケージを平行に起動するために、turborepo を設定します。

```json:turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "dev": {
      "dependsOn": ["^dev"]
    }
  }
}
```

Workers のコードは、`workers` ディレクトリにパッケージとして配置します。

```yaml:pnpm-workspace.yaml
packages:
  - "workers/*"
```

```txt:.gitignore
node_modules
.turbo
```

```bash
pnpm install
```

## gatewayアプリを作成

このサービスは唯一外部に露出させ、他の内部プライベートサービスと通信するための gateway アプリです。
create hono という Hono のテンプレートプロジェクトを生成できる CLI ツールを使います。

`workers` ディレクトリに移動して

```bash
npm create hono@latest gateway
# cloudflare-workers を選択
```

とすると、`gateway` ディレクトリにプロジェクトが生成されます。
生成された `README.md` と `package.json` 内の `devDependencies` にある `wrangler` は root にあるので削除しても大丈夫です。

monorepo 管理のため、`package.json` の `name` フィールドを `gateway` としておきます。

```diff ts:workers/gateway/package.json
{
+  "name": "gateway",
...
}
```

```diff toml:workers/gateway/wrangler.toml
- name = "my-app"
+ name = "gateway"
  compatibility_date = "2023-01-01"

+ [dev]
+ port = 1234
```

`wrangler.toml` に workers アプリ名と、dev 時のポート番号を設定しておきましょう。

`pnpm install` で依存関係を Monorepo にインストールします。`pnpm dev` で指定された URL に Hello Hono! と表示されれば OK です。

## microserviceアプリを作成

本来は複数の microservice が生えますが、今回は 1 つだけ `private-service` という名前の worker を `workers` フォルダ以下に作成します。

gateway をコピペして、`private-service` ディレクトリを作成して `package.json` と `wangler.toml` を書きます。

```diff ts:workers/private-service/package.json
{
+  "name": "private-service",
...
}
```

```diff toml:workers/private-service/wrangler.toml
- name = "gateway"
+ name = "private-service"
  compatibility_date = "2023-01-01"

  [dev]
- port = 1234
+ port = 1235
```

あとは `/` にアクセスしたときのレスポンスをわかりやすいように変えておきます。

```ts:workers/private-service/src/index.ts
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.text('Hello Private Service!'))

export default app
```

`/` にアクセスすると Hello Private Service! と返すようにしました。

:::details tree

```txt
.
├── node_modules
├── package.json
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
├── turbo.json
└── workers
   ├── gateway
   └── private-service
```

ここまででこんな感じです
:::

## Service Binding

### Gateway設定

Service Binding を設定するために、gateway の `wrangler.toml` に `services` フィールドを追加します。

```diff toml:workers/gateway/wrangler.toml
  name = "gateway"
  compatibility_date = "2023-01-01"
+ services = [
+     { binding = "PRIVATE_SERVICE", service = "private-service" }
+ ]

  [dev]
  port = 1234
```

このとき、`service` にはバインド先サービスの `wrangler.toml` の `name` フィールドに設定した値を指定します。

### Hono設定

Hono では Service Binding を型安全に使うことができるので、それを使ってみます。
https://hono.dev/getting-started/cloudflare-workers#bindings

```diff ts:workers/gateway/src/index.ts
  import { Hono } from 'hono'

+ type Bindings = {
+   PRIVATE_SERVICE: Fetcher;
+ };

- const app = new Hono()
+ const app = new Hono<{ Bindings: Bindings }>()

  app.get('/', (c) => c.text('Hello Hono!'))

  export default app
```

`Bindings` という型を定義して、`Hono` の Generics に渡します。
このとき、`Bindings` のキーは `wrangler.toml` の `services` フィールドに設定した `binding` の値と同じにしておきます。

## Gatewayを書く

`/private` にアクセスしたら、`private-service` に転送させてレスポンスをパススルーするようにします。

```diff ts:workers/gateway/src/index.ts
  import { Hono } from 'hono'

  type Bindings = {
    PRIVATE_SERVICE: Fetcher;
  };

  const app = new Hono<{ Bindings: Bindings }>()

  app.get('/', (c) => c.text('Hello Hono!'))
+ app.get('/private/*', async (c) => {
+   const res = await c.env.PRIVATE_SERVICE.fetch(c.req.raw)
+   return res
+ })

  export default app
```

`/private` のリクエストがそのまま `private-service` に転送されるので、`/private` にアクセスすると Hello Private Service! と返ってくるようにサブパスに配置します。

```diff ts:workers/private-service/src/index.ts
  import { Hono } from 'hono'

- const app = new Hono()
+ const app = new Hono().basePath('/private')
```

これでサービス間の通信ができます。

## Privateにする

`private-service` は、`gateway` からのみアクセスできるようにします。
まずは内部通信用に TOKEN を設定しておきます。

両方のサービスに `.dev.vars` を作って、その中に `INTERNAL_TOKEN` を設定します。

```txt:.dev.vars
INTERNAL_TOKEN = "THIS_IS_A_SECRET_TOKEN_FOR_INTERNAL_REQUEST"
```

### Gateway側でトークンを付与する

`gateway` 側で `private-service` にアクセスするとき、このトークンを付与します。

```diff ts:workers/gateway/src/index.ts
  import { Hono } from 'hono'

  type Bindings = {
    PRIVATE_SERVICE: Fetcher;
+   INTERNAL_TOKEN: string;
  };

  const app = new Hono<{ Bindings: Bindings }>()

  app.get('/', (c) => c.text('Hello Hono!'))
  app.get('/private/*', async (c) => {
-   const res = await c.env.PRIVATE_SERVICE.fetch(c.req.raw)
+   const res = await c.env.PRIVATE_SERVICE.fetch(c.req.raw, {
+     headers: {
+       'x-custom-token': c.env.INTERNAL_TOKEN,
+     },
+   })
    return res
  })

  export default app
```

### Private Service側でトークンを検証する

`private-service` 側で、`gateway` からのアクセスであることを検証します。

```diff ts:workers/private-service/src/index.ts
  import { Hono } from 'hono'

+ type Bindings = {
+   INTERNAL_TOKEN: string;
+ };

- const app = new Hono().basePath('/private')
+ const app = new Hono<{ Bindings: Bindings }>().basePath('/private')

+ app.use('*', async (c, next) => {
+   const token = c.req.headers.get('x-custom-token')
+   if (token !== c.env.INTERNAL_TOKEN) {
+     return c.text('Unauthorized', 401)
+   }
+   await next()
+ })

...
```

これで、`gateway` からのアクセスでない場合は、`Unauthorized` というレスポンスを `401` で返すようになりました。

## Deployする

`gateway` と `private-service` をそれぞれ `pnpm run deploy` でデプロイします。
`prod` 環境には `INTERNAL_TOKEN` が設定されていないので、dashboard か wrangler cli で設定するのを忘れないようにしましょう。

`gateway` 側のデプロイされた URL から `/private` にアクセスすると、`Hello Private Service!` と返ってきます。
一方で直接 `private-service` 側の URL にアクセスすると `Unauthorized` と返ってくるようになってれば OK です。
そして `INTERNAL_TOKEN` が外側からは見えないようになっているはずです。

## おわりに

いかがだったでしょうか。これがある程度無料で動かせるのはすごいですよね...gateway 側で色々し放題なのが素晴らしいです。
例えば Binding 先の Workers に立てた Remix の SSR 結果を gateway 一定時間キャッシュするようにして ISR のようなことをしても面白そうです。
更新頻度の低い API を Cache + Purge 式にしても良さそう。

Hono の開発体験もめっちゃ良かったので Service Binding を使うときは是非使ってみてください。

## 参考

https://developers.cloudflare.com/workers/configuration/bindings/about-service-bindings/#authentication-workers-service
https://github.com/cloudflare/workers-sdk/pull/1503
