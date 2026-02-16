---
title: "esbuildでpnpmモノレポの共有パッケージをバンドルしてデプロイする"
emoji: "📦"
type: "tech"
topics: [esbuild, monorepo, pnpm, typescript]
published: true
---

## pnpmモノレポの構成

複数のサーバーレスFunctionsを持つTypeScriptプロジェクトで、pnpmワークスペースのモノレポを採用しています。`packages/` に共有ライブラリをまとめて、`apps/` 配下の各サービスから `workspace:*` で参照する構成です。

```
my-project/
├── apps/
│   ├── service-a/
│   ├── service-b/
│   └── service-c/
├── packages/
│   ├── shared-models/       # 共有データモデル
│   ├── shared-core/         # 共有コアロジック
│   └── shared-utils/        # 共有ユーティリティ
├── pnpm-workspace.yaml
└── package.json
```

```json
// apps/service-a/package.json
{
  "dependencies": {
    "@my-project/shared-models": "workspace:*",
    "@my-project/shared-core": "workspace:*",
    "zod": "^3.22.0"
  }
}
```

`pnpm install` するとシンボリックリンクが貼られ、npmレジストリに公開しなくてもローカルで依存が解決されます。ローカル環境では問題なく快適に開発を進めていたのですが、デプロイ時に問題が発生しました。

## tscでビルドしたらデプロイ時に落ちた

```
Error: Cannot find module '@my-project/shared-models'
```

`tsc` はTypeScriptをJavaScriptにコンパイルしますが、import文はそのまま残します。

```ts
// src/index.ts（コンパイル前）
import { User } from '@my-project/shared-models';
import { processData } from '@my-project/shared-core';
```

```js
// dist/index.js（tscコンパイル後）
"use strict";
const shared_models_1 = require("@my-project/shared-models"); // ← そのまま残る
const shared_core_1 = require("@my-project/shared-core");     // ← そのまま残る
```

ローカルでは `pnpm install` によってワークスペースパッケージが `node_modules` 内にシンボリックリンクとして配置されるので、`require("@my-project/shared-models")` はリンク経由で `packages/shared-models/` の実体を参照できます。

しかしデプロイ先では `pnpm install` ではなく通常の `npm install` が走ります。`"@my-project/shared-models": "workspace:*"` はnpmレジストリに存在しないので解決できず、シンボリックリンクも貼られません。結果として `require()` が参照先を見つけられずエラーになります。

## esbuildでバンドルする

対策としては、共有ライブラリをプライベートnpmレジストリに公開する方法や、ビルド成果物を `node_modules` にコピーするスクリプトを書く方法があります。
ただ、前者はレジストリの管理やバージョニングのオーバーヘッドが大きく、後者はパスの管理が複雑で脆いという問題があります。
全依存を1ファイルにまとめてしまえばデプロイ先でのモジュール解決自体が不要になるので、バンドラーを使うことにしました。バンドラーには設定がシンプルでビルドが速い [esbuild](https://esbuild.github.io/) を選びました。

esbuildはGo言語で書かれていて、`require()` や `import` を再帰的に辿り、すべてのコードを1つのファイルにまとめてくれます。

```js
// tscの出力 — require()がそのまま残る
const shared_models_1 = require("@my-project/shared-models");

// esbuildの出力 — @my-project/shared-models の中身がインライン展開される
// → require()自体がなくなり、デプロイ先でのモジュール解決が不要になる
```

これなら `@my-project/*` をnpmに公開しなくても、デプロイ先で動きます。

## esbuild.config.mjs

esbuildはビルドツールなので `devDependencies` にインストールし、各サービスに設定ファイルを置きます。

```js
// apps/service-a/esbuild.config.mjs
import esbuild from 'esbuild';

await esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  platform: 'node',
  target: 'node22',
  format: 'cjs',
  outfile: 'dist/index.js',

  // ランタイムが提供するパッケージはバンドルしない
  external: [
    '@google-cloud/*',
  ],

  sourcemap: true,
  minify: true,
  treeShaking: true,

  logLevel: 'info',
}).catch(() => process.exit(1));
```

| オプション | 説明 |
|---|---|
| `bundle` | `import`/`require` を辿って依存を1ファイルにまとめる |
| `platform` | `'node'` を指定するとNode.js向けの出力になる（`fs` 等の組み込みモジュールをバンドルしない） |
| `target` | 出力するJavaScriptの構文レベル。デプロイ先のランタイムに合わせる |
| `format` | `'cjs'`（CommonJS）か `'esm'`（ES Modules）か。既存コードに合わせて選ぶ |
| `external` | バンドルに含めないパッケージ |
| `minify` | 変数名の短縮や空白除去で出力サイズを削減する |
| `treeShaking` | importされているが実際には使われていないコードを除去する |

## `external`：バンドルしないものを決める

esbuildはデフォルトで全依存をバンドルに含めますが、externalにすべきものがあります。

まず、ネイティブバイナリ（`.node` ファイル）を含むパッケージです。`sharp`、`bcrypt`、`sqlite3` などがこれにあたります。esbuildはネイティブモジュールをバンドルできず、ビルド環境（CI）とデプロイ先でプラットフォームが異なる場合、仮にバンドルできても動きません。

また、実行環境がプリインストールしているパッケージもバンドルしない方がよいです。たとえばAWS Lambdaの `@aws-sdk/*` やCloud Functionsの `@google-cloud/*` がこれにあたります。バンドルに含めるとランタイム側のバージョンと競合したり、バンドルサイズが不必要に肥大化します。

```js
external: [
  '@google-cloud/*',   // ワイルドカードで一括指定
],
```

`external` に指定したパッケージは `require()` のまま出力に残り、実行時に解決されます。

`external` に何を入れるかは実行環境次第なので、デプロイ先のランタイムが何を提供しているか確認しておくとよいです。

## デプロイ用 package.json の生成

もう1つ対応が必要です。`package.json` にはまだ `@my-project/*` への依存が書かれています。

```json
{
  "dependencies": {
    "@my-project/shared-models": "workspace:*",  // ← これが残っている
    "@my-project/shared-core": "workspace:*",    // ← これも
    "zod": "^3.22.0"
  }
}
```

esbuildですでにバンドル済みなのでこれらは不要ですが、デプロイ先で `npm install` が走ると`Cannot find module` エラーになってしまうので、デプロイ前に除去するスクリプトを用意します。

```js
const fs = require('fs');
const path = require('path');

const pkgPath = process.argv[2]
  ? path.resolve(process.argv[2])
  : path.join(process.cwd(), 'package.json');

const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf8'));

pkg.dependencies = Object.fromEntries(
  Object.entries(pkg.dependencies || {})
    .filter(([name]) => !name.startsWith('@my-project/'))
);

fs.writeFileSync(pkgPath, JSON.stringify(pkg, null, 2));
```

`@my-project/*` を除去すると、`package.json` にはesbuildでバンドルしなかった外部パッケージ（`external` に指定したもの）だけが残ります。デプロイ先の `npm install` ではそれらだけがインストールされます。

## スクリプト構成

開発時と本番で使い分けます。

```json
{
  "scripts": {
    "build": "tsc",
    "build:prod": "node esbuild.config.mjs"
  }
}
```

- `build`：開発用。`tsc` で型チェック付きコンパイル。エディタのエラー表示と一致させる
- `build:prod`：デプロイ用。esbuildでバンドル

## CI/CD

GitHub Actionsでの構成例です。

```yaml
steps:
  - uses: actions/checkout@v5
  - uses: pnpm/action-setup@v4
  - uses: actions/setup-node@v4
    with:
      node-version: 22
      cache: 'pnpm'

  - run: pnpm install --frozen-lockfile

  # 1. 共有パッケージを先にビルド
  - name: Build shared packages
    run: pnpm --filter "./packages/*" build

  # 2. 各サービスをesbuildでバンドル＆package.json書き換え
  - name: Build service-a
    working-directory: apps/service-a
    run: |
      pnpm build:prod
      node ../../scripts/generate-deploy-package.js package.json

  # 3. デプロイ
  - name: Deploy
    run: # デプロイコマンド
```

ビルド順序に注意が必要です。共有パッケージの `package.json` で `"main": "dist/index.js"` のようにビルド成果物を指している場合、esbuildはそのパスを読みにいきます。`dist/index.js` は `tsc` で生成されるファイルなので、先に共有パッケージを `tsc` でビルドしてからサービスのesbuildを走らせる必要があります。逆にすると `Could not resolve` で落ちます。

## esbuildにしてどうなったか

- **ビルド高速化**：esbuildのバンドルは1秒未満で終わります。Go製で並列処理に最適化されているだけあって速いです。
- **デプロイサイズが小さい**：tree-shakingで実際に使われているコードだけがバンドルされるため、`node_modules` をまるごと持ち込んでいた頃とは比較になりません。サイズの削減はサーバーレスFunctionsのコールドスタート改善にもつながります。
- **型チェックは別途必要**：esbuildは型チェックをしないので、別途型チェックの仕組みが必要です。

## まとめ

- `tsc` はimportをそのまま残すので、モノレポのワークスペースパッケージがデプロイ先で解決できない。esbuildで1ファイルにバンドルすることでこの問題を解消できる
- `external` でランタイム提供パッケージやネイティブモジュールをバンドル対象から除外し、`package.json` からワークスペース依存を除去するスクリプトを組み合わせることで、デプロイ先に必要なものだけを持ち込める
- 副次的にビルド速度の向上、tree-shakingによるデプロイサイズの削減、コールドスタートの改善が得られる

## 参考

- https://esbuild.github.io/
- https://pnpm.io/workspaces
