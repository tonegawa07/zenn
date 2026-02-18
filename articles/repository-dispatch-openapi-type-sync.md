---
title: "repository_dispatchでリポジトリ間のOpenAPI型定義を自動同期する"
emoji: "🔄"
type: "tech"
topics: [githubactions, openapi, typescript]
published: true
---

## 型定義ファイルの同期問題

バックエンドとフロントエンドを別リポジトリで開発していると、API仕様の変更をフロントエンド側に反映する作業が地味に面倒です。

バックエンドリポジトリの `openapi.yaml` から `openapi-typescript` で型定義ファイルを生成していたのですが、フロントエンドリポジトリへの同期は以下のような流れで手動で行っていました。

1. バックエンドリポジトリで OpenAPI 定義を更新する
2. 開発者がローカルで `openapi-typescript` を実行して `api.ts` を再生成する
3. フロントエンドリポジトリにコミットしてPRを出す

一見シンプルですが、実際にはいくつかの問題を抱えていました。

- **同期忘れ**: API側の変更をマージした後に `api.ts` の更新を忘れ、フロントエンドが古い型定義のまま開発が進む
- **タイミングのズレ**: 誰がいつ型定義を更新するかが曖昧で、複数人が同時に更新して競合する

## repository_dispatch で自動同期する

API定義の変更に連動して型定義が自動更新されれば、これらの問題は解消できます。GitHub Actions の `repository_dispatch` イベントを使い、バックエンドリポジトリの OpenAPI 定義変更をトリガーにフロントエンドリポジトリで `api.ts` を再生成してPRを作成する仕組みを構築しました。

```
backend-api の openapi.yaml が develop にマージされる
  ↓
GitHub Actions が repository_dispatch を frontend-web に送信
  ↓
frontend-web 側の GitHub Actions が起動
  ↓
openapi.yaml を取得して openapi-typescript で api.ts を再生成
  ↓
差分があれば自動でPRを作成
```

## 認証方式の選定

この仕組みではリポジトリをまたいだ操作が必要ですが、`GITHUB_TOKEN` では権限が足りません。認証方式の候補は以下の2つで、今回は GitHub App を採用しました。

| 方式 | メリット | デメリット |
|------|---------|-----------|
| Personal Access Token (PAT) | 設定が簡単 | 個人アカウントに紐づく。退職・異動時にトークン再発行が必要 |
| GitHub App | 組織所有。短命トークン（1時間で失効）。権限を最小限に絞れる | 初期設定がやや手間 |

## GitHub App の準備

### 1. GitHub App を作成

Organization の Settings → Developer settings → GitHub Apps → New GitHub App から作成します。

設定のポイント：

- **Webhook**: 無効にする（Active のチェックを外す）
- **Repository permissions**:
  - Contents: Read and write
  - Pull requests: Read and write
- **インストール先**: 対象リポジトリ（backend-api, frontend-web）を選択

Contents は `Read-only` だと `repository_dispatch` 送信時に `Resource not accessible by integration` エラーになります。`repository_dispatch` は対象リポジトリへの書き込み操作にあたるため、**Read and write** が必要です。

### 2. Secrets を登録

Organization の Secrets に以下を登録し、両リポジトリからアクセスできるようにします。

- `APP_ID`: GitHub App の App ID
- `APP_PRIVATE_KEY`: 生成した Private Key（`.pem` ファイルの中身）

Private Key は生成時に一度だけダウンロードできます。紛失した場合は App 設定ページから再生成できます。

## ワークフローの実装

### backend-api 側：dispatch 送信

`openapi.yaml` の変更が `develop` にマージされたときに `repository_dispatch` を送信するワークフローです。

```yaml
# .github/workflows/dispatch-web-types.yml
name: Dispatch Web Types Update

on:
  push:
    branches: [develop]
    paths:
      - "openapi/openapi.yaml"

jobs:
  dispatch:
    runs-on: ubuntu-latest
    steps:
      - name: Generate GitHub App Token
        uses: actions/create-github-app-token@v1
        id: app-token
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          repositories: frontend-web

      - name: Dispatch to frontend-web
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ steps.app-token.outputs.token }}
          repository: your-owner/frontend-web
          event-type: update-api-types
```

`actions/create-github-app-token@v1` が GitHub App の短命トークンを生成し、`peter-evans/repository-dispatch@v3` がそのトークンを使って別リポジトリにイベントを送信します。

### frontend-web 側：型定義の自動更新PR作成

`repository_dispatch` を受けて `openapi.yaml` を GitHub API で取得し、`api.ts` を再生成してPRを作成します。

```yaml
# .github/workflows/update-api-types.yml
name: Update API Types

on:
  repository_dispatch:
    types: [update-api-types]

jobs:
  update-types:
    runs-on: ubuntu-latest
    steps:
      - name: Generate GitHub App Token
        uses: actions/create-github-app-token@v1
        id: app-token
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          repositories: frontend-web,backend-api

      - name: Checkout frontend-web
        uses: actions/checkout@v4
        with:
          ref: develop

      - name: Download openapi.yaml from backend-api
        run: |
          curl -H "Authorization: token ${{ steps.app-token.outputs.token }}" \
               -H "Accept: application/vnd.github.v3.raw" \
               -o openapi.yaml \
               "https://api.github.com/repos/your-owner/backend-api/contents/openapi/openapi.yaml?ref=develop"

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Generate api.ts
        run: npx openapi-typescript openapi.yaml -o src/types/api.ts

      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v7
        with:
          token: ${{ steps.app-token.outputs.token }}
          commit-message: "chore: OpenAPI型定義を自動更新"
          title: "chore: OpenAPI型定義を自動更新"
          branch: chore/update-api-types
          base: develop
          delete-branch: true
```

`peter-evans/create-pull-request@v7` は差分がない場合はPR作成をスキップしてくれるので、不要なPRが量産される心配はありません。

## まとめ

- 手動で行っていた OpenAPI 型定義の同期を、`repository_dispatch` + GitHub App で自動化した
- API定義の変更が自動でフロントエンドの型定義に反映されるようになり、同期忘れやタイミングのズレが解消された
- この仕組みは OpenAPI に限らず、GraphQL スキーマや Protocol Buffers など、バックエンドの定義からフロントエンドのコードを生成するケース全般に応用できる

## 参考

- https://github.com/actions/create-github-app-token
- https://github.com/peter-evans/repository-dispatch
- https://github.com/peter-evans/create-pull-request
- https://github.com/openapi-ts/openapi-typescript
