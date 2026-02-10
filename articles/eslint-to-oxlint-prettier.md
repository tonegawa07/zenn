---
title: "ESLint v9 (Flat Config) への移行があまりにも大変そうなので、思い切ってoxlint + Prettierに移行した"
emoji: "⚓️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript, oxc, oxlint, prettier]
published: true
---

## ESLint 8系の構成

TypeScriptのバックエンドプロジェクトで、ESLint 8系を使っており、構成はこんな感じだった。

```
eslint
@typescript-eslint/eslint-plugin
@typescript-eslint/parser
eslint-config-google
eslint-plugin-import
eslint-plugin-jsdoc
eslint-import-resolver-typescript
```

ESLintがコード品質チェックとフォーマットの両方を担っていた。

1. **コード品質チェック**: 未使用変数、import制約、TypeScript固有のルールなど
2. **フォーマット**: `indent`、`quotes`、`max-len`、`import/order`（import文の並び順）など

## ESLint v9への移行がしんどい

ESLint 9ではFlat Configへの移行が必須になった。
調査してみると、以下のようにFlat Configへの移行に各プラグインが追従しきれておらず、この時点でESLint 9への移行は諦めることにした。

- `.eslintrc.js` → `eslint.config.mjs` への書き換えが必要
- `eslint-config-google` がESLint 9に非対応で、そのまま使えない
- `eslint-plugin-import` も非対応で、`eslint-plugin-import-x` への差し替えが必要
- `@typescript-eslint` も統合パッケージへの移行が推奨されている

## lintとformatを分離する

ここで立ち止まって、そもそもの構成を見直すことにした。ESLintにフォーマットまで任せていたのがよくなかったので、フォーマットは **Prettier** に分離し、lintは **oxlint** に置き換えることにした。

## なぜoxlintか

### Rust製で圧倒的に速い

oxlintはRust製のリンターで、ESLintと比較して50〜100倍高速とされている。
lintの高速化はCIの高速化にもつながる。

### プラグインが内蔵されている

oxlintは `typescript`、`import`、`unicorn`、`jest` などの主要なプラグインを内蔵している。`@typescript-eslint` のバージョンを気にしたり、プラグイン同士の互換性を調べたりする必要がなく、設定ファイル1つで完結する。

### oxfmtの存在

oxlintを開発しているOxcプロジェクトでは、Rust製フォーマッターの **oxfmt** の開発も進んでいる。将来的にPrettierをoxfmtに置き換えれば、lint + formatの両方がRust製ツールで統一される。Prettierは現時点での最善の選択だが、エコシステム全体の方向性としてOxcに寄せておくのは悪くないと判断した。

## 移行手順

### 1. Prettierの導入

まずフォーマットをPrettierに移した。ESLintのフォーマット系ルール（`indent`、`quotes`、`max-len`、`import/order` など）を削除し、Prettierに一任する。

```bash
npm install -D prettier
```

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

Prettierの高速化のため `@prettier/plugin-oxc`（パーサーをRust製のOxcに置き換えるプラグイン）も導入した。これだけで **24秒 → 16秒** に改善した。

### 2. ESLintをoxlintに置き換え

ESLint関連のパッケージを全て削除し、oxlintをインストールした。

```bash
# ESLint関連を削除
npm uninstall eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser \
  eslint-config-google eslint-plugin-import eslint-plugin-jsdoc \
  eslint-import-resolver-typescript

# oxlintをインストール
npm install -D oxlint
```

```json
{
  "scripts": {
    "lint": "oxlint .",
    "lint:fix": "oxlint --fix ."
  }
}
```

ESLintで使っていたカスタムルールは、oxlintでもほぼそのまま `.oxlintrc.json` に移植できた。

有効化したプラグインは `eslint`、`typescript`、`unicorn`、`oxc`、`import`、`jest` の6つ。いずれもoxlintに内蔵されているため、追加のnpmパッケージは不要になる。

### 3. CIパイプラインの更新

ESLint + reviewdogの構成から、oxlint + Prettierに変更した。

```yaml
# Before
- name: ESLint with reviewdog
  uses: reviewdog/action-eslint@v1
  with:
    reporter: github-pr-review
    filter_mode: file
    fail_level: error

# After
- name: Oxlint
  run: npm run lint

- name: Prettier
  run: npm run format:check
```

reviewdogによるPRインラインコメント機能は失われたが、oxlintのエラー出力は十分に見やすく、CIログから問題箇所を特定するのに困ることはなかった。
oxlintの恩恵によりCIのlintステップは1秒以下で終わるようになった。

### 4. 不要ファイルの削除

`.eslintrc.js` と、ESLintパーサー用の `tsconfig.eslint.json` を削除した。oxlintはtsconfigを参照しないため、パーサー用の設定ファイルが不要になる。

## 最終的な構成

| 役割 | ツール |
|---|---|
| リンティング | oxlint（eslint, typescript, unicorn, oxc, import, jest プラグイン） |
| フォーマット | Prettier + @prettier/plugin-oxc |

devDependenciesは **7パッケージ → 3パッケージ** に削減された。

```diff
- "@typescript-eslint/eslint-plugin"
- "@typescript-eslint/parser"
- "eslint"
- "eslint-config-google"
- "eslint-import-resolver-typescript"
- "eslint-plugin-import"
- "eslint-plugin-jsdoc"
+ "oxlint"
+ "prettier"
+ "@prettier/plugin-oxc"
```

## 注意点

- **ESLintのルールが全てサポートされているわけではない。** eslint-plugin-jsdocのように対応していないプラグインもある。カスタムルールを多用しているプロジェクトでは事前に互換性を確認しておきたい。
- **エコシステムはまだ発展途上。** ただし主要なルールは内蔵されており、実用上困ることは少ない。
