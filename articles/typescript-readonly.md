---
title: "TypeScriptのreadonlyとas constをちゃんと理解する"
emoji: "☺️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [TypeScript]
published: true
---

TypeScriptの`readonly` `as const` について、イマイチ良く分かっていなかったので整理したいと思います。

## readonlyとは

「一度値を代入したら、その後は変更できない」という制約をプロパティや配列に与える修飾子です。

## 基本的な使い方

### オブジェクトのプロパティを保護する

インターフェースや型エイリアスのプロパティに `readonly` をつけると、そのプロパティへの再代入がエラーになります。

```typescript
interface User {
  readonly id: number; // 変更不可
  name: string;        // 変更可能
}

const user: User = {
  id: 1,
  name: "Taro"
};

user.name = "Jiro"; // OK
user.id = 2;        // エラー: Cannot assign to 'id' because it is a read-only property
```

IDや設定値など、生成時から変わるべきではない値に使います。

### 配列を書き換え不可にする

配列に対して使うと、要素の追加・削除・変更ができなくなります。

```typescript
// 普通の配列
const list: number[] = [1, 2, 3];
list.push(4); // OK

// readonlyな配列
const readonlyList: readonly number[] = [1, 2, 3];

readonlyList.push(4); // エラー: pushが存在しない
readonlyList[0] = 9;  // エラー: インデックス署名がreadonlyのみ
```

上記のように`push`, `pop`, `splice` などの破壊的メソッドが使えなくなります。`map` や `filter` などの新しい配列を返すメソッドは使用可能です。

### Utility Type `Readonly<T>` を使う

既存の型をすべて `readonly` にしたい場合、`Readonly<T>` を使うと一括で変換できます。

```typescript
interface Config {
  host: string;
  port: number;
}

const appConfig: Readonly<Config> = {
  host: "localhost",
  port: 8080
};

appConfig.port = 3000; // エラー
```

### クラスのプロパティで使う

クラス内でも、コンストラクタでの初期化時以外に変更できないプロパティを作れます。

```typescript
class Car {
  readonly brand: string;

  constructor(brand: string) {
    this.brand = brand; // コンストラクタ内なら代入OK
  }

  changeBrand() {
    this.brand = "Toyota"; // エラー
  }
}
```

## 注意点

### `readonly` は「浅い (Shallow)」

`readonly` は「そのプロパティへの参照の変更」を防ぐだけで、「プロパティの中身（オブジェクト）」までは不変にしません。

```typescript
interface Student {
  readonly profile: {
    name: string;
  };
}

const student: Student = {
  profile: { name: "Hanako" }
};

student.profile = { name: "Yuki" }; // エラー（参照自体は変えられない）

student.profile.name = "Yuki"; // OK（中身は書き換えられてしまう）
```

中身まで完全に変更不可にしたい場合は、ネストされたオブジェクトにも `readonly` をつけるか、後述する `as const` を使います。

### `const` との違い

混同しそうですが、使い分けは明確です。

- `const`: 変数そのものの再代入を防ぐ
- `readonly`: プロパティの変更を防ぐ

```typescript
const user = { name: "Bob" };

user = { name: "Alice" }; // エラー（constの効果）
user.name = "Alice";      // OK（プロパティは守られていない）
```

オブジェクトの中身も守りたいなら、`readonly`や`as const`が必要です。

### ランタイムには影響しない

JavaScriptにコンパイルされると `readonly` は消えます。あくまで開発中の型チェック機能であり、実行時の動作制限はありません。

これはTypeScriptの「型は消去される」という設計思想に沿ったものです。enumのようにランタイムコードを生成する独自機能とは異なり、`readonly`は純粋な型レベルの機能です。

## `as const` を使う

`readonly` を手書きする代わりに `as const` を使うと、再帰的にすべてのプロパティを `readonly` にしてくれます。

```typescript
const config = {
  api: {
    endpoint: "https://api.example.com",
    timeout: 5000
  }
} as const;

config.api.timeout = 10000; // エラー（全てがreadonlyになる）
```
