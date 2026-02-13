---
title: "TypeScriptでenum型を避けたほうがいい理由を改めて整理する"
emoji: "☺️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [TypeScript]
published: true
published_at: 2025-09-11 09:00
---

普段レビューで「enumは避けてUnionにしたいです」などと言っているのですが、そういえばなぜ避けたほうがいいのかちゃんと説明できないなと思ったので、改めて整理したいと思います。

## enumの例

```typescript
enum Status {
  Pending,
  InProgress,
  Done,
}
```

## なぜenumを避けるべきなのか？
### Tree Shakingが効かず、バンドルサイズが肥大化する

enumはJavaScriptにトランスパイルされる際、即時実行関数（IIFE）でラップされたオブジェクトを生成します。

TypeScriptコード

```typescript
enum Color {
  Red,
  Green,
  Blue,
}

console.log(Color.Red);
```

トランスパイル後のJavaScriptコード

```javascript
var Color;
(function (Color) {
    Color[Color["Red"] = 0] = "Red";
    Color[Color["Green"] = 1] = "Green";
    Color[Color["Blue"] = 2] = "Blue";
})(Color || (Color = {}));

console.log(Color.Red);
```

上記のようにColor.Redしか使っていなくても、Colorオブジェクト全体がバンドルファイルに含まれます。

その結果、不要なコードを削除する Tree Shaking の恩恵を受けられず、バンドルサイズが意図せず大きくなる原因になります。

### 数値enumの型の安全性が低い

数値enumはnumber型と互換性があり、意図しない値の代入が可能です。

```typescript
enum Status {
  Pending, // 0
  InProgress, // 1
  Done, // 2
}

let taskStatus: Status;

taskStatus = Status.InProgress; // OK
taskStatus = 1;                 // これもOK

// enumで定義されていない数値も代入できてしまう
taskStatus = 99;                // エラーにならない
```

上記のようにStatus型に定義されていない数値を代入してもTypeScriptコンパイラはエラーを検知できません。

### リバースマッピングによる混乱

数値enumは、キーから値へのマッピング（Color.Red → 0）だけでなく、値からキーへのリバースマッピング（Color[0] → "Red"）も自動的に生成します。

```typescript
enum Color {
  Red, // 0
  Green, // 1
}

console.log(Color.Red);      // 出力: 0
console.log(Color[0]);       // 出力: "Red"
console.log(Object.keys(Color)); // 出力: ["0", "1", "Red", "Green"]
```

上記のようにObject.keys() などで enum を反復処理しようとすると、キーと値の両方が含まれます。

## 代替手段
### 文字列リテラル型 (Union Types)

```typescript
type Status = 'Pending' | 'InProgress' | 'Done';
let taskStatus: Status;
taskStatus = 'InProgress'; // OK
taskStatus = 'Unknown';    // エラー
```

### as const を使ったオブジェクト

```typescript
// 値を定義
export const STATUS = {
  PENDING: 'pending',
  IN_PROGRESS: 'inProgress',
  DONE: 'done',
} as const;

// 値から型を生成 (Union Type)
export type Status = typeof STATUS[keyof typeof STATUS];
// Status 型は 'pending' | 'inProgress' | 'done' となる

// 使用例
let taskStatus: Status = STATUS.IN_PROGRESS;
```

## TypeScriptの思想

TypeScriptはあくまで "JavaScript + 静的型付け" です。

enum等のTypeScriptの独自機能は、上記のように元のコードからは想像しにくい複雑なJavaScriptコードを生成する場合があります。

enumのようなランタイムに影響を与える独自機能は、アプリケーションの挙動やパフォーマンスに影響を与えるためなるべく避けましょう。
