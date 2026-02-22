---
title: 'Hono and React realtime app'
emoji: '🔥'
type: 'tech'
topics:
        - 'react'
        - 'cloudflare'
        - 'tailwind'
        - 'sqlite'
        - 'hono'
published: true
published_at: '2026-02-22 21:35'
---

# Hono and React realtime app

https://tsei.jp/articles/2026/02/20/note/

↑ english ver

世界で最も満足度が高い [₍₁₎](https://x.com/yusukebe/status/2018862080099864706) とされる Web 標準フレームワーク Hono と React で、リアルタイムアプリを構築します。

また、サービスに不可欠な認証や永続化を、なるべく 1 円も払わずに公開します。
以下コマンドから、Hono のテンプレートを生成してプロジェクトを開始します。

```ts
% npm create hono@latest

> npx
> create-hono

create-hono version 0.19.4
✔ Target directory app
✔ Which template do you want to use?
cloudflare-workers+vite
✔ Do you want to install project dependencies? No
✔ Cloning the template
```

:::message

HonoX のファイルベースルーティングは今回使用しません。
つかいたい場合は[前掲の記事](https://zenn.dev/jp/articles/bfd5996bc430f4)をご確認ください。

https://zenn.dev/jp/articles/bfd5996bc430f4

:::

ここまでの差分:

https://github.com/tseijp/voxelizer/pull/17/changes

## 1. setup react and hono

### 1.1. 開発前の準備

以下のコマンドで、必要なパッケージをインストールします。

```rb
npm i react react-dom swr tailwindcss
npm i -D @auth/core @auth/drizzle-adapter @hono/auth-js @tailwindcss/vite @types/react @types/react-dom @vitejs/plugin-react drizzle-kit drizzle-orm partyserver partysocket
```

"tsconfig.json" を修正し、`"jsxImportSource": "react"` を指定して JSX が React で処理されるようにします。

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "skipLibCheck": true,
-    "lib": ["ESNext"],
+    "lib": ["ESNext", "DOM"],
-    "types": ["vite/client"],
+    "types": ["vite/client", "@cloudflare/workers-types"],
    "jsx": "react-jsx",
-    "jsxImportSource": "hono/jsx"
+    "jsxImportSource": "react"
  }
}
```

:::message

他の設定はお好みに合わせてください。

- "lib" に "DOM" を追加すると、ブラウザ API の型補完が有効になります。
- "types" に "@cloudflare/workers-types" を追加すると、Workers Bindings の型が使えます。

:::

### 1.2. Backend の修正

バックエンドから SSR 関連のコードを削除します。（"src/renderer.tsx" も不要なので削除して大丈夫です。）
また vite の plugin 設定を react と tailwind に差し替えます。

1. ```ts
   // src/index.tsx
   import { Hono } from 'hono'
   export default new Hono().get('/api/res', (c) => c.text('ok'))
   // src/renderer.ts REMOVE
   ```
2. ```ts
   // vite.config.ts
   import { cloudflare } from '@cloudflare/vite-plugin'
   import { defineConfig } from 'vite'
   -import ssrPlugin from 'vite-ssr-components/plugin'
   +import react from '@vitejs/plugin-react'
   +import tailwindcss from '@tailwindcss/vite'
   export default defineConfig({
   -  plugins: [cloudflare(), ssrPlugin()],
   +  plugins: [cloudflare(), react(), tailwindcss()],
   })
   ```

### 1.3. Frontend の修正

フロントエンド側に React と Tailwind CSS の最小構成を追加します。
"src/style.css・src/client.tsx・index.html" の 3 ファイルをいつも通り作成します。

1. ```tsx
    /* src/style.css */
    @import 'tailwindcss';
   ```
2. ```tsx
   // src/client.tsx
   import './style.css'
   import { createRoot } from 'react-dom/client'
   createRoot(document.getElementById('root')!).render('ok')
   ```
3. ```html
   <!-- index.html -->
   <script src="/src/client.tsx" type="module"></script>
   <div id="root" />
   ```

以上で、"npm run dev" でサーバーを起動し、以下の 2 ページで ok と表示されれば ok です。

- [localhost:5173](http://localhost:5173)
- [localhost:5173/api/res](http://localhost:5173/api/res)

ここまでの差分:

https://github.com/tseijp/voxelizer/pull/18/changes

## 2. setup infra and auth

### 2.1 oauth 認証のセットアップ

Google OAuth を使った認証基盤を構築します。

[1](https://r.tsei.jp/note/2026-02-20/1.jpg)
[2](https://r.tsei.jp/note/2026-02-20/2.jpg)
[3](https://r.tsei.jp/note/2026-02-20/3.jpg)
[4](https://r.tsei.jp/note/2026-02-20/4.jpg)
[5](https://r.tsei.jp/note/2026-02-20/5.jpg)
[6](https://r.tsei.jp/note/2026-02-20/6.jpg)
[7](https://r.tsei.jp/note/2026-02-20/7.jpg)

1. [Google Cloud コンソールの新しいプロジェクト](https://console.cloud.google.com/projectcreate) の作成ボタンを押します
2. "Google Auth Platform Clients" で検索して、でてきた "Clients" ページを開きます
3. ブランディングが未作成の場合、 "Get started" ボタンが表示されるので押します
4. なんでもいいのでぽちぽちして、ブランディングの作成ボタンを押します
5. ブランディングの作成が完了したら、"Create client" ボタンを押します
6. Web application を選択し、以下の値を入力します（デプロイ後に本番 URL の追加が必要です）
      ```rb
      承認済みの JavaScript 生成元: http://localhost:5173
      承認済みのリダイレクト URI: http://localhost:5173/api/auth/callback/google
      ```
7. 表示されるモーダルの Download JSON ボタンを押して認証情報をダウンロードします（以下のような構造です）
      ```json
      {
        "web": {
          "client_id": "xxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.apps.googleusercontent.com",
          ...
          "client_secret": "GOCSPX-xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
        }
      }
      ```
8. ".dev.vars" ファイルを作成し、ダウンロードした JSON の値をもとに環境変数を設定します
      ```
      AUTH_URL = "http://localhost:5173/api/auth"
      AUTH_SECRET = "random"
      GOOGLE_CLIENT_ID = "xxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.apps.googleusercontent.com"
      GOOGLE_CLIENT_SECRET = "GOCSPX-xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
      ```

:::message

- "AUTH_URL" はローカル開発時に localhost を指定し、本番では deploy 先の URL に差し替えます
- "AUTH_SECRET" は "openssl rand -base64 32" で生成したランダム文字列をつかいます
- "GOOGLE_CLIENT_ID" と "GOOGLE_CLIENT_SECRET" はダウンロードした JSON の "web.client_id" と "web.client_secret" から取得します
- 本番デプロイ後は Cloudflare Console 上でも同じ環境変数をぽちぽち設定してください
- ".dev.vars" に ";" セミコロンが混入すると正しい環境変数が読み込まれません（".vscode/settings.json" で `"editor.formatOnSave": true` を設定していると "Ctrl+S" でついてしまいます）

:::

### 2.2 database schema の追加

ユーザーがサインアップしたとき、認証情報が永続化されるようにします。
Auth.js と Drizzle ORM の公式ドキュメントのコードをそのままもってきます。

<!-- prettier-ignore -->
1. ```ts
   // drizzle.config.ts
   import type { Config } from 'drizzle-kit'
   export default {
     out: './migrations',
     schema: './src/schema.ts',
     dialect: 'sqlite',
   } satisfies Config
   ```
2. ```ts
   // src/schema.ts
   import { integer, sqliteTable, text, primaryKey } from 'drizzle-orm/sqlite-core'
   import type { AdapterAccountType } from '@auth/core/adapters'
   export const users = sqliteTable('user', {
     id: text('id')
       .primaryKey()
       .$defaultFn(() => crypto.randomUUID()),
     name: text('name'),
     email: text('email').unique(),
     emailVerified: integer('emailVerified', { mode: 'timestamp_ms' }),
     image: text('image'),
   })
   export const accounts = sqliteTable(
     'account',
     {
       userId: text('userId')
         .notNull()
         .references(() => users.id, { onDelete: 'cascade' }),
       type: text('type').$type<AdapterAccountType>().notNull(),
       provider: text('provider').notNull(),
       providerAccountId: text('providerAccountId').notNull(),
       refresh_token: text('refresh_token'),
       access_token: text('access_token'),
       expires_at: integer('expires_at'),
       token_type: text('token_type'),
       scope: text('scope'),
       id_token: text('id_token'),
       session_state: text('session_state'),
     },
     (account) => [primaryKey({ columns: [account.provider, account.providerAccountId] })]
   )
   export const sessions = sqliteTable('session', {
     sessionToken: text('sessionToken').primaryKey(),
     userId: text('userId')
       .notNull()
       .references(() => users.id, { onDelete: 'cascade' }),
     expires: integer('expires', { mode: 'timestamp_ms' }).notNull(),
   })
   export const verificationTokens = sqliteTable(
     'verificationToken',
     {
       identifier: text('identifier').notNull(),
       token: text('token').notNull(),
       expires: integer('expires', { mode: 'timestamp_ms' }).notNull(),
     },
     (verificationToken) => [primaryKey({ columns: [verificationToken.identifier, verificationToken.token] })]
   )
   export const authenticators = sqliteTable(
     'authenticator',
     {
       credentialID: text('credentialID').notNull().unique(),
       userId: text('userId')
         .notNull()
         .references(() => users.id, { onDelete: 'cascade' }),
       providerAccountId: text('providerAccountId').notNull(),
       credentialPublicKey: text('credentialPublicKey').notNull(),
       counter: integer('counter').notNull(),
       credentialDeviceType: text('credentialDeviceType').notNull(),
       credentialBackedUp: integer('credentialBackedUp', {
         mode: 'boolean',
       }).notNull(),
       transports: text('transports'),
     },
     (authenticator) => [primaryKey({ columns: [authenticator.userId, authenticator.credentialID] })]
   )
   ```

:::message

- 公式ドキュメントの複合主キーの書き方が古いので、"sqliteTable" の第 3 引数はオブジェクトではなく配列を返すように修正してください。でないと以下の deprecation warning が発生します。
     > ```json
     > The signature '(name: "account", columns: { userId: NotNull<SQLiteTextBuilderInitial<"userId", [string, ...string[]], number | undefined>>; type: NotNull<$Type<SQLiteTextBuilderInitial<"type", [...], number | undefined>, AdapterAccountType>>; ... 8 more ...; session_state: SQLiteTextBuilderInitial<...>; }, extraConfig?: ((self: { ...; }) => SQLiteTableExtraConfig) | undefined): SQLiteTableWithColumns<...>' of 'sqliteTable' is deprecated.
     > ```
- ref
     - [Hono Auth.js Integration - Hono](https://hono.dev/examples/hono-authjs#step-2-database-setup)
     - [Auth.js | Drizzle](https://authjs.dev/getting-started/adapters/drizzle#schemas)

:::

### 2.3 Cloudflare Binding の作成

- 以下のコマンドで Cloudflare への認証ステータスを確認しておきます。
     ```rb
     npx wrangler login
     npx wrangler whoami
     ```
- 以下のコマンドで cloudflare の設定コードを生成します。出力されたテキストを "wrangler.jsonc" に記載します。
     ```rb
     npx wrangler d1 create my-d1-xxx
     npx wrangler r2 bucket create my-r2-xxx
     ```
- 以下のコマンドで migrations を生成し、ローカルとリモートの両方に適用します。（CDN Edge 上の SQLite なので無料で試せるはずです。）
     ```rb
     npx drizzle-kit generate
     npx wrangler d1 migrations apply --local my-d1-xxx
     npx wrangler d1 migrations apply --remote my-d1-xxx
     ```
- "wrangler.jsonc" に Durable Objects の設定を直接追記します。（CLI からの生成コマンドが存在しないためです。）
     ```json
     {
       ...
       "durable_objects": {
         "bindings": [
           {
             "name": "v1",
             "class_name": "PartyServer"
           }
         ]
       },
       "migrations": [
         {
           "tag": "v1",
           "new_sqlite_classes": ["PartyServer"]
         }
       ],
       ...
     }
     ```

:::message

- "migrations_dir" の値はデフォルトで "migrations" です。"drizzle.config.ts" の `out: './migrations'` と対応しています。（[Migrations · Cloudflare D1 docs](https://developers.cloudflare.com/d1/reference/migrations/#wrangler-customizations)）
- 自動生成時に `"remote": true` がついていたら削除してください。（ローカル開発時にリモートの D1 を参照しようとして失敗します。）
- `"$schema": "node_modules/wrangler/config-schema.json"` はモノレポ構成だとパスが変わります。
- "observability" を enabled にすると Workers の監視ダッシュボードが有効化されます。
- Console 上の環境変数がデプロイのたびに消えないよう "keep_vars" を true にしています。
- "wrangler.jsonc" は以下を参考にしてください。

<!-- prettier-ignore -->
```json
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "party",
  "compatibility_date": "2025-08-03",
  "main": "./src/index.tsx",
  // ↓↓↓ my created ↓↓↓
  "durable_objects": {
    "bindings": [
      {
        "name": "v1",
        "class_name": "PartyServer"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["PartyServer"]
    }
  ],
  // ↓↓↓ generated by `npx wrangler d1 create my-d1-party` ↓↓↓
  "d1_databases": [
    {
      "binding": "my_d1_xxx",
      "database_name": "my-d1-xxx",
      "database_id": "9571c20b-357e-40ec-83fb-068584ca7f52",
      "migrations_dir": "migrations"
    }
  ],
  // ↓↓↓ generated by `npx wrangler r2 bucket create my-r2-xxx` ↓↓↓
  "r2_buckets": [
    {
      "bucket_name": "my-r2-xxx",
      "binding": "my_r2_xxx"
    }
  ],
  // ↓↓↓ recommend ↓↓↓
  "observability": {
    "enabled": true
  },
  "keep_vars": true
}
```

:::

ここまでの差分:

https://github.com/tseijp/voxelizer/pull/19/changes

## 3. reatitime app

### 3.1 fix Backend

Backend には Google OAuth 認証と、partyserver（Durable Objects ベースの WebSocket ライブラリ）への中継ミドルウェアを追加します。

<!-- prettier-ignore -->
```ts
// index.tsx
import { users } from './schema'
import Google from '@auth/core/providers/google'
import { DrizzleAdapter } from '@auth/drizzle-adapter'
import { authHandler, initAuthConfig, verifyAuth } from '@hono/auth-js'
import { eq } from 'drizzle-orm'
import { drizzle } from 'drizzle-orm/d1'
import { Hono } from 'hono'
import { env } from 'hono/adapter'
import { createMiddleware } from 'hono/factory'
import { routePartykitRequest, Server } from 'partyserver'
import type { Connection, ConnectionContext } from 'partyserver'

const getUserBySub = (DB: D1Database, sub: string) => drizzle(DB).select().from(users).where(eq(users.id, sub)).limit(1)
const authMiddleware = initAuthConfig((c) => ({
  adapter: DrizzleAdapter(drizzle(c.env.my_d1_xxx)),
  providers: [Google({ clientId: c.env.GOOGLE_CLIENT_ID, clientSecret: c.env.GOOGLE_CLIENT_SECRET })],
  secret: c.env.AUTH_SECRET,
  session: { strategy: 'jwt' },
}))
const myMiddleware = createMiddleware(async (c) => {
  const headers = new Headers(c.req.raw.headers)
  headers.set('x-user-sub', c.get('authUser')?.token?.sub!)
  const req = new Request(c.req.raw, { headers })
  const res = await routePartykitRequest(req, env(c))
  return res ?? c.text('Not Found', 404)
})

type Env = { my_d1_xxx: D1Database; my_r2_xxx: R2Bucket }
type Conn = Connection<{ username: string }>

export class PartyServer extends Server<Env> {
  users = {} as Record<string, string>
  static options = { hibernate: true }
  async onConnect(conn: Conn, c: ConnectionContext) {
    const sub = c.request.headers.get('x-user-sub')!
    const [user] = await getUserBySub(this.env.my_d1_xxx, sub)
    conn.setState({ username: user.name! })
  }
  async onMessage(conn: Conn, message: string) {
    this.users[conn.state!.username] = message
    this.broadcast(JSON.stringify(this.users))
  }
  onClose(conn: Conn) {
    delete this.users[conn.state!.username]
    this.broadcast(JSON.stringify(this.users), [conn.id])
  }
}

export default new Hono<{ Bindings: Env }>()
  .get('/api/res', (c) => c.text('ok'))
  .use('*', authMiddleware)
  .use('/parties/*', verifyAuth())
  .use('/parties/*', myMiddleware)
  .use('/api/auth/*', authHandler())
  .use('/api/v1/*', verifyAuth())
  .get('/api/v1/me', async (c) => {
    const { token } = c.get('authUser')
    if (!token || !token.sub) return c.json(null, 401)
    const [user] = await getUserBySub(c.env.my_d1_xxx, token.sub)
    return c.json({ username: user.name || null })
  })
```

### 3.2 fix Frontend

まずは簡単な例として、Google アカウントのユーザー名とマウスカーソル位置を WebSocket で共有するアプリを実装します。

<!-- prettier-ignore -->
```tsx
// client.tsx
import './style.css'
import { signIn } from '@hono/auth-js/react'
import { usePartySocket } from 'partysocket/react'
import { useState } from 'react'
import { createRoot } from 'react-dom/client'
import useSWRImmutable from 'swr/immutable'
const Cursors = () => {
  const [users, set] = useState<[username: string, transform: string][]>([])
  const socket = usePartySocket({
    party: 'v1',
    room: 'my-room',
    onOpen: () => addEventListener('mousemove', (e) => socket.send(`translate(${e.clientX}px, ${e.clientY}px)`)),
    onMessage: (e) => set(Object.entries(JSON.parse(e.data))),
  })
  return users.map(([username, transform]) => (
    <div key={username} className="absolute" style={{ transform }}>
      {username}
    </div>
  ))
}
const fetcher = async () => {
  const res = await fetch('/api/v1/me')
  return await res.json()
}
const App = () => {
  const { data } = useSWRImmutable('me', fetcher)
  if (!data) return <button onClick={() => void signIn()}>Sign In</button>
  return <Cursors />
}

createRoot(document.getElementById('root')!).render(<App />)
```

### 3.3. reatitime game

先週、国土交通省 project の PLATEAU AWARD にてイノベーション賞をいただきました 🎉
東京 23 区の都市モデルをボクセル化し、階層的経路探索 "HPA\*" でルート検索できます。

[navigator.glre.dev](https://navigator.glre.dev)

[![](https://r.tsei.jp/note/2026-02-20/20260212.gif =256x)](https://navigator.glre.dev)
[![](https://r.tsei.jp/note/2026-02-20/20260213.gif =256x)](https://navigator.glre.dev)
[![](https://r.tsei.jp/note/2026-02-20/0.jpg =512x)](https://www.youtube.com/live/7pkahO9tWFw)

> - service: [navigator.glre.dev](https://navigator.glre.dev)
> - require: [navigator.glre.dev/claude/ja](https://navigator.glre.dev/claude/ja)
> - proposal: [navigator.glre.dev/readme/ja](https://navigator.glre.dev/readme/ja)
> - schedule: [docs.google.com/spreadsheets](https://docs.google.com/spreadsheets/d/1HLuEUU5CTvMhOYZFNg4IE8dtlqicwMc-EzXwbWefWcU)

[voxelizer](https://github.com/tseijp/voxelizer) で今回の構成を使ってリアルタイム通信を試してみました。
"client.tsx" をボクセル都市用に書き換えると以下のようなかんじになります。

https://github.com/tseijp/voxelizer/blob/main/projects/app/src/client.tsx

ここまでの差分:

https://github.com/tseijp/voxelizer/pull/20/changes

## Conclusion

"npm run deploy" で、認証・DB・WebSocket を備えたリアルタイムアプリが Cloudflare 上にゼロコストで公開できます。
以下のコマンドで環境を削除できます。

```sh
# D1 データベースの削除
npx wrangler d1 delete my-d1-xxx

# R2 バケットの削除
npx wrangler r2 bucket delete my-r2-xxx

# Worker の削除（Durable Objects も同時に削除されます）
npx wrangler delete party
```
