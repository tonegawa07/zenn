---
title: "OpenAPI定義からExpressのRequestに型をつける"
emoji: "🧩"
type: "tech"
topics: [TypeScript, Express, OpenAPI]
published: true
---

## Expressの Request 型

ExpressのRequestの型はデフォルトだと以下のようになっています。

```typescript
const getBook = async (req: Request, res: Response) => {
  const bookId = req.params.bookId; // string
  const body = req.body;            // any
};
```

`req.params` は `{ [key: string]: string }` なので存在しないパラメータにもアクセスできてしまいます。
`req.body` は `any` です。

もし自前で型定義をしようとするとこうなります。

```typescript
interface GetBookParams {
  bookId: string;
}

const getBook = async (
  req: Request<GetBookParams>,
  res: Response
) => {
  req.params.bookId; // string — OK!
};
```

型はつきますが、APIの仕様をOpenAPI等で管理していた場合、自前の型定義との二重管理になり、OpenAPIを更新したのにTypeScriptの型を直し忘れる、といったズレが起きやすくなります。

## やりたいこと

```
OpenAPI定義 (YAML)
    ↓ openapi-typescript
TypeScript型定義 (paths)
    ↓ TypedRequest（自作Utility Type）
Express Request<Params, ResBody, ReqBody, Query>
```

1. OpenAPIのYAMLから `openapi-typescript` で型を自動生成する
2. 生成された `paths` 型からパスパラメータ・リクエストボディを抽出するUtility Typeを書く
3. Expressの `Request` に注入する

## 例: 書籍管理API

ここでは以下のような書籍管理APIを例とします。

:::details openapi.yaml（全文）
```yaml
openapi: 3.0.0
info:
  title: Bookshelf API
  version: 1.0.0

paths:
  /books:
    get:
      summary: 書籍一覧取得
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [published, draft]
        - name: limit
          in: query
          schema:
            type: integer
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: "#/components/schemas/Book"
    post:
      summary: 書籍登録
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateBookRequest"
      responses:
        "201":
          description: Created

  /books/{bookId}:
    get:
      summary: 書籍詳細取得
      parameters:
        - name: bookId
          in: path
          required: true
          schema:
            type: string
      responses:
        "200":
          description: OK
    patch:
      summary: 書籍更新
      parameters:
        - name: bookId
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/UpdateBookRequest"
      responses:
        "200":
          description: OK

components:
  schemas:
    Book:
      type: object
      properties:
        id:
          type: string
        title:
          type: string
        author:
          type: string
        price:
          type: integer
        status:
          type: string
          enum: [published, draft]

    CreateBookRequest:
      type: object
      required: [title, author]
      properties:
        title:
          type: string
        author:
          type: string
        price:
          type: integer

    UpdateBookRequest:
      type: object
      properties:
        title:
          type: string
        author:
          type: string
        price:
          type: integer
        status:
          type: string
          enum: [published, draft]
```
:::

## openapi-typescriptでOpenAPI定義から型を生成する

[openapi-typescript](https://github.com/openapi-ts/openapi-typescript) で型を生成します。

```bash
npx openapi-typescript openapi.yaml -o src/types/api.ts
```

生成された `src/types/api.ts` の中身を一部抜粋すると以下のようになります。

```typescript
// src/types/api.ts（自動生成・抜粋）

export interface paths {
  "/books": {
    get: {
      parameters: {
        query: {
          status?: "published" | "draft";
          limit?: number;
        };
      };
      responses: {
        200: {
          content: {
            "application/json": components["schemas"]["Book"][];
          };
        };
      };
    };
    post: {
      requestBody: {
        content: {
          "application/json": components["schemas"]["CreateBookRequest"];
        };
      };
      // ...
    };
  };
  "/books/{bookId}": {
    get: {
      parameters: {
        path: {
          bookId: string;
        };
      };
      // ...
    };
    patch: {
      parameters: {
        path: {
          bookId: string;
        };
      };
      requestBody: {
        content: {
          "application/json": components["schemas"]["UpdateBookRequest"];
        };
      };
      // ...
    };
  };
}
```

OpenAPI定義がそのままTypeScriptの型になっています。`paths` という型にすべてのエンドポイント情報がまとまっています。

## TypedRequestを実装する

この `paths` 型から、パスパラメータやリクエストボディを抽出してExpressの `Request` に渡すUtility Typeを作ります。

```typescript
// src/types/apiTypes.ts

import type { Request } from "express";
import type { paths } from "@/types/api";

/**
 * APIエンドポイントの型を抽出するユーティリティ型
 */
type ApiEndpoint<
  Path extends keyof paths,
  Method extends keyof paths[Path],
> = paths[Path][Method];

/**
 * パスパラメータを抽出
 */
type ExtractPathParams<
  Path extends keyof paths,
  Method extends keyof paths[Path],
> = ApiEndpoint<Path, Method> extends { parameters: { path: infer P } }
  ? P
  : Record<string, never>;

/**
 * リクエストボディを抽出
 */
type ExtractRequestBody<
  Path extends keyof paths,
  Method extends keyof paths[Path],
> = ApiEndpoint<Path, Method> extends {
  requestBody: { content: { "application/json": infer B } };
}
  ? B
  : never;

/**
 * クエリパラメータを抽出
 */
type ExtractQueryParams<
  Path extends keyof paths,
  Method extends keyof paths[Path],
> = ApiEndpoint<Path, Method> extends { parameters: { query: infer Q } }
  ? Q
  : Record<string, never>;

/**
 * 型安全なExpressリクエスト型
 */
export type TypedRequest<
  Path extends keyof paths,
  Method extends keyof paths[Path],
> = Request<
  ExtractPathParams<Path, Method>,
  unknown,
  ExtractRequestBody<Path, Method>,
  ExtractQueryParams<Path, Method>
>;
```

### 何をやっているか

Expressの `Request` 型は4つのジェネリクスを受け取ります。

```typescript
Request<Params, ResBody, ReqBody, Query>
```

| 引数 | 対応するプロパティ | 意味 |
|------|----------------|------|
| `Params` | `req.params` | パスパラメータ (`/books/:bookId`) |
| `ResBody` | `res.json()` の型 | レスポンスボディ（今回は使わない） |
| `ReqBody` | `req.body` | リクエストボディ |
| `Query` | `req.query` | クエリパラメータ |

`TypedRequest` は、openapi-typescriptが生成した `paths` 型からこれらを抽出して `Request` に渡しています。

### Conditional Type と infer

`ExtractPathParams` を例に見てみます。

```typescript
type ExtractPathParams<Path, Method> =
  ApiEndpoint<Path, Method> extends { parameters: { path: infer P } }
    ? P
    : Record<string, never>;
```

`extends` で構造を判定し、`infer P` でその中の型を取り出しています。`ApiEndpoint<Path, Method>` が `{ parameters: { path: ??? } }` という構造を持っていたら `???` の部分が `P` に入り、持っていなければ空オブジェクトになります。

`GET /books` にはパスパラメータがないので `Record<string, never>`（空オブジェクト）、`GET /books/{bookId}` には `bookId` があるので `{ bookId: string }` が抽出されます。

### なぜ Record\<string, never> なのか

パスパラメータやクエリパラメータが存在しないエンドポイントでは、`req.params.something` のような存在しないプロパティへのアクセスをコンパイル時に弾きたいためです。

```typescript
// Record<string, never> の場合
req.params.bookId; // コンパイルエラー — 意図通り

// {} の場合
req.params.bookId; // エラーにならない — 危険
```

TypeScriptでは `{}` 型は「nullとundefined以外のすべて」を許容してしまうため、意図せずプロパティにアクセスできてしまいます。`Record<string, never>` を使うことで「キーはあるかもしれないが、値はnever（取り出せない）」となり、存在しないプロパティへのアクセスを防げます。

## コントローラーで使う

```typescript
// src/controllers/bookController.ts

import type { Response } from "express";
import type { TypedRequest } from "@/types/apiTypes";

/** 書籍登録 */
const create = async (
  req: TypedRequest<"/books", "post">,
  res: Response,
) => {
  const { title, author, price } = req.body;
  // title: string, author: string, price: number | undefined
};

/** 書籍詳細取得 */
const show = async (
  req: TypedRequest<"/books/{bookId}", "get">,
  res: Response,
) => {
  const bookId = req.params.bookId; // string
};

/** 書籍更新 */
const update = async (
  req: TypedRequest<"/books/{bookId}", "patch">,
  res: Response,
) => {
  const bookId = req.params.bookId;      // string
  const { title, author, price, status } = req.body;
  // status: "published" | "draft" | undefined — enumも効いている
};
```

`TypedRequest<"/books/{bookId}", "patch">` と書くだけで、`req.params`・`req.body` にOpenAPI定義どおりの型がつきます。
OpenAPI定義に存在しないパスやメソッドの組み合わせはコンパイル時に弾かれるので、定義と実装の乖離が構造的に起きなくなります。

## まとめ

- openapi-typescriptで生成される `paths` 型から、Conditional Type + `infer` でパスパラメータやリクエストボディを抽出できる
- それをExpressの `Request` に渡すUtility Typeを作ることで、OpenAPI定義と型定義の二重管理がなくなる
- OpenAPI定義に存在しないパスやメソッドはコンパイル時に弾かれるので、定義と実装の乖離を防げる

## 参考

- https://github.com/openapi-ts/openapi-typescript
- https://speakerdeck.com/tonegawa07/typespecdeshi-xian-suruxin-kunaiopenapisukimaqu-dong-kai-fa
