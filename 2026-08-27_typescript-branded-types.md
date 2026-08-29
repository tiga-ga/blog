---
title: 【TypeScript】型エイリアスで防げない同一プリミティブ型の引数取り違えを「ブランド型（Branded Types）」でコンパイル時に防ぐ
tags: TypeScript, 設計, Architecture, 初心者向け, OReilly
author: tiga-ga (https://qiita.com/tiga-ga)
---

# 【TypeScript】型エイリアスで防げない同一プリミティブ型の引数取り違えを「ブランド型（Branded Types）」でコンパイル時に防ぐ

---

## はじめに：なぜこれを学ぼうと思ったのか？

* **きっかけ**: オライリー書籍『初めてのTypeScript』を読んでいて、TypeScript の型システムが「構造的部分型（Structural Typing）」であることを再認識しました。
* **直面した課題**: `type UserId = string;` と `type OrderId = string;` のように丁寧に名前をつけた型エイリアスを定義しても、**中身が同じ `string` なら引数を逆に渡してもコンパイルが普通に通ってしまう** という落とし穴があります。
* **今回作った検証コード**: 為替計算で「30万円のノートPC」と「50ドルのマウス」の引数を逆にしてしまい、**4,500万円の超高額誤請求が発生する事故コード**を作成。これを実行時コスト完全ゼロ（0ms / 0byte）で、コンパイル時に100%確実に弾く**「ブランド型（Branded Types / Nominal Typing）」**を検証しました。

> **本記事の動作確認サンプルコードは GitHub で公開しています**
> [GitHub - sample-code (2026-08-26_typescript-branded-types)](https://github.com/tiga-ga/sample-code/tree/main/2026-08-26_typescript-branded-types)

---

## なぜ `type UserId = string` では防げないのか？

多くの現場で、可読性を上げるために以下のような「型エイリアス」が使われています。

```typescript
type UserId = string;
type OrderId = string;

// 注文キャンセル関数
function cancelOrder(userId: UserId, orderId: OrderId) { ... }
```

一見すると、「`userId` には `UserId` 型を、`orderId` には `OrderId` 型しか渡せない安全な関数」に見えます。

しかし、呼び出し側で以下のように**引数の順序を逆にしてしまったらどうなるでしょうか？**

```typescript
const userId: UserId = "USR-ALICE";
const orderId: OrderId = "ORD-1001";

// 逆に渡してしまった！
cancelOrder(orderId, userId); 
```

**なんと、TypeScript は何のエラーも出さず、平然とコンパイルを通してしまいます。**

### 原因：TypeScript の「構造的部分型（Structural Typing）」
TypeScript は「型の名前」ではなく**「型の構造（中身）」**を見て互換性を判断します。
`UserId` も `OrderId` も中身はただの `string` なので、TypeScript から見れば**「どちらも同じ string だから代入可能」**と判断されてしまうのです。

---

## Bad: 通常の型エイリアスでは引数の順序逆転を検知できない (bad.ts)

実際にどれだけ危険なバグにつながるかを実感するために、通貨（USD と JPY）の計算コードを例に見てみましょう。

* 該当コード: [bad.ts (GitHub)](https://github.com/tiga-ga/sample-code/blob/main/2026-08-26_typescript-branded-types/bad.ts)

```typescript
// bad.ts
type UserId = string;
type OrderId = string;
type USD = number;
type JPY = number;

const ordersDB: Record<string, string> = { "ORD-1001": "USR-ALICE" };

// 関数の引数に UserId と OrderId を指定
function cancelOrder(userId: UserId, orderId: OrderId): string {
  if (ordersDB[orderId] !== userId) {
    return `不正アクセス！注文者(${ordersDB[orderId] ?? "なし"}) と 要求者(${userId}) が不一致です`;
  }
  return `注文 ${orderId} を正常にキャンセルしました`;
}

// 関数の引数に USD と JPY を指定
function calculateTotal(priceUSD: USD, priceJPY: JPY, rate = 150): number {
  return priceUSD * rate + priceJPY;
}

// 呼び出し側
const userId: UserId = "USR-ALICE";
const orderId: OrderId = "ORD-1001";
const mouseUSD: USD = 50;        // 50ドル (約7,500円)
const laptopJPY: JPY = 300000;   // 30万円

// 1. 引数を逆に渡してしまった（UserId と OrderId を指定しているのに素通り！）
console.log(cancelOrder(orderId, userId));

// 2. 通貨の引数を逆に渡してしまった（USD と JPY を指定しているのに素通り！）
const total = calculateTotal(laptopJPY, mouseUSD);
console.log(`誤請求金額: ¥${total.toLocaleString()}`);
// -> 誤請求金額: ¥45,000,050
```

### 何が起きたのか？
関数 `calculateTotal` は引数に `(priceUSD: USD, priceJPY: JPY)` と指定しており、`priceUSD * 150 + priceJPY` を計算します。
しかし、引数を逆にして `calculateTotal(laptopJPY, mouseUSD)` と呼んでしまったため、**30万円に150倍が掛けられて「4,500万円」が請求される大インシデント**が発生してしまいました。

関数側に `USD` や `JPY` と型を指定しているにもかかわらず、どちらも中身が `number` なので、TypeScript はこの致命的なミスを一切警告してくれません。

---

## 解決策：ゼロコストで型を区別する「ブランド型（Branded Types）」

そこで登場するのが、TypeScript 上で「公称型（Nominal Typing: 名前の違いで型を区別する仕組み）」を再現する**ブランド型（Branded Types）**です。

### ブランド型の基本形
```typescript
// 型空間にのみ存在する一意なシンボル（値は生成しない）
declare const __brand: unique symbol;

// Base型（stringやnumber）に「見えないタグ」を付与するヘルパー
export type Brand<Base, Tag extends string> = Base & {
  readonly [__brand]: Tag;
};
```

これを使って、ドメイン固有の型を定義します。

```typescript
export type BrandUserId = Brand<string, "UserId">;
export type BrandOrderId = Brand<string, "OrderId">;
export type BrandUSD = Brand<number, "USD">;
export type BrandJPY = Brand<number, "JPY">;
```

### なぜこれで防げるのか？
* `BrandUserId` は `{ readonly [__brand]: "UserId" }` というタグを持っています。
* `BrandOrderId` は `{ readonly [__brand]: "OrderId" }` というタグを持っています。
* TypeScript は構造を見るため、タグの名前（`"UserId"` と `"OrderId"`）が一致しない限り、**「互換性のない全く別の型」として厳密に扱ってくれます**。

### 実行時オーバーヘッドは「完全ゼロ（0ms / 0byte）」
`declare const __brand` は TypeScript のコンパイラに「このシンボルが存在する」と認識させる宣言です。
コンパイル後の JavaScript には、このプロパティもコードも **1 文字も出力されません**。
余計なオブジェクト生成やメモリ消費、実行時オーバーヘッドが完全にゼロです。

---

## Good: ブランド型によるコンパイルエラーでバグを未然に防ぐ (good.ts)

ブランド型を適用した推奨コードがこちらです。

* 該当コード: [good.ts (GitHub)](https://github.com/tiga-ga/sample-code/blob/main/2026-08-26_typescript-branded-types/good.ts)

```typescript
// good.ts
declare const __brand: unique symbol;
type Brand<Base, Tag extends string> = Base & { readonly [__brand]: Tag };

type BrandUserId = Brand<string, "UserId">;
type BrandOrderId = Brand<string, "OrderId">;
type BrandUSD = Brand<number, "USD">;
type BrandJPY = Brand<number, "JPY">;

const ordersDB: Record<string, string> = { "ORD-1001": "USR-ALICE" };

// 関数の引数に BrandUserId と BrandOrderId を要求
function cancelOrder(userId: BrandUserId, orderId: BrandOrderId): string {
  if (ordersDB[orderId as string] !== (userId as string)) {
    return `不正アクセス！注文者(${ordersDB[orderId as string] ?? "なし"}) と 要求者(${userId as string}) が不一致です`;
  }
  return `注文 ${orderId} を正常にキャンセルしました`;
}

// 関数の引数に BrandUSD と BrandJPY を要求
function calculateTotal(priceUSD: BrandUSD, priceJPY: BrandJPY, rate = 150): number {
  return (priceUSD as number) * rate + (priceJPY as number);
}

// 呼び出し側（変数名・値は Bad と完全一致）
const userId = "USR-ALICE" as BrandUserId;
const orderId = "ORD-1001" as BrandOrderId;
const mouseUSD = 50 as BrandUSD;
const laptopJPY = 300000 as BrandJPY;

// 引数を逆に渡すと、即座にコンパイルエラー！
cancelOrder(orderId, userId);
calculateTotal(laptopJPY, mouseUSD);
```

このコードを書いた瞬間、エディタ上には赤波線が表示され、コンパイルエラーとして検出されます。

---

## 検証：`npm run check` で違いを確認する

サンプルコードのリポジトリで、型チェックを実行してみます。

```bash
npm run check
# (内部で tsc が実行されます)
```

### 出力結果
```text
good.ts(43,13): error TS2345: Argument of type 'BrandOrderId' is not assignable to parameter of type 'BrandUserId'.
  Type 'BrandOrderId' is not assignable to type '{ readonly [__brand]: "UserId"; }'.
    Types of property '[__brand]' are incompatible.
      Type '"OrderId"' is not assignable to type '"UserId"'.

good.ts(44,16): error TS2345: Argument of type 'BrandJPY' is not assignable to parameter of type 'BrandUSD'.
  Type 'BrandJPY' is not assignable to type '{ readonly [__brand]: "USD"; }'.
    Types of property '[__brand]' are incompatible.
      Type '"JPY"' is not assignable to type '"USD"'.
```

### ここがポイント
* **`bad.ts`**: まったく同じように引数を逆に渡しているのに、**エラー 0 件で完全に素通り**（本番に流出して大事故に）。
* **`good.ts`**: ブランド型を付与した行だけが、**ピンポイントでコンパイルエラーを叩き出し、ビルドを停止**。

「コンパイルエラーになるからこそ安心できる」——事故コードを本番にデプロイさせない強力な防壁が完成します。

---

## Bad vs Good 比較表

| 項目 | Bad (`bad.ts`) | Good (`good.ts`) |
| :--- | :--- | :--- |
| **型定義** | `type UserId = string;`<br>`type USD = number;` | `type BrandUserId = Brand<string, "UserId">;`<br>`type BrandUSD = Brand<number, "USD">;` |
| **型付けの性質** | 構造的部分型（実質ただの string/number） | **公称型（Nominal Typing）のシミュレーション** |
| **引数の順序逆転** | **素通り**（`cancelOrder(orderId, userId)` が通過） | **即座にコンパイルエラー** |
| **通貨の順序逆転** | **素通り**（円に150倍が掛かり4,500万円の誤計算） | **即座にコンパイルエラー** |
| **型チェック (`tsc`)** | **エラー 0 件（素通り）** | **コンパイルエラーでビルドを強制終了** |
| **実行時オーバーヘッド** | ゼロ | **完全ゼロ（0ms / 0byte）** |

---

## 実務でブランド型を使うべきシーン

ブランド型は、以下のような「同じプリミティブ型だが、意味がまったく異なる値」を扱う場面で役立ちます。

1. **ID の取り違え防止**:
   - `UserId`, `OrderId`, `TeamId`, `TenantId`（すべて `string` だが混ざると危険）
2. **通貨・単位の混同防止**:
   - `USD`, `JPY`, `EUR`
   - `Meters`, `Kilometers`, `Milliseconds`, `Seconds`
3. **セキュリティ境界の表現**:
   - `RawPassword` vs `HashedPassword`
   - `UnsanitizedHtml` vs `SanitizedHtml`

---

## ブランド型の使いすぎに注意：他にもある解決アプローチ

ブランド型は非常に強力ですが、「便利だからといって、あらゆるプリミティブ型にブランドをつける」のはアンチパターンです。

### 使いすぎによる3つのデメリット
1. **ボイラープレートの増加**: あらゆる変数に `as BrandXxx` のキャストが必要になり、コードの記述量が増えて可読性が低下します。
2. **外部ライブラリとの摩擦**: React や ORM（Prisma など）、ユーティリティ関数は普通の `string` や `number` を期待しているため、型変換の手間が爆発します。
3. **そもそも型が違えば不要**: 例えば `function register(name: string, age: number)` のように、最初から引数の型が異なっていれば、ブランド型を使わなくても TypeScript が引数逆転を 100% 防いでくれます。

### 引数の取り違えを防ぐ「他のやり方」

実は、引数の取り違えを防ぐ手段はブランド型だけではありません。実務では以下の代替アプローチも広く使われており、状況に応じて使い分けるのがベストプラクティスです。

#### 1. オブジェクト（名前付き引数）にまとめる
もっとも手軽で、多くの現場で採用されている方法です。引数を1つのオブジェクトにまとめます。

```typescript
// 引数をオブジェクトにする
function cancelOrder(params: { userId: UserId; orderId: OrderId }) { ... }

// 呼び出し側：プロパティ名で渡すため、順序の概念がなくなる
cancelOrder({
  userId: "USR-ALICE",
  orderId: "ORD-1001",
});
```

* **メリット**: ブランド型のような特別な型定義が不要で、直感的に誰でも書ける。
* **デメリット**: 関数呼び出しごとにオブジェクトが生成されるため、極限のパフォーマンスが求められるループ処理などではわずかにオーバーヘッドがある。

#### 2. スマートコンストラクタ（バリデーション関数）を通す
直接 `as BrandUserId` と書くのではなく、「バリデーションを通過したものだけにブランドを授ける関数」を用意する方法です。

```typescript
function parseUserId(raw: string): BrandUserId {
  if (!raw.startsWith("USR-")) {
    throw new Error(`不正なユーザーIDです: ${raw}`);
  }
  return raw as BrandUserId; // この関数の中でのみ安全にキャスト！
}
```

* **メリット**: 「引数の取り違え防止」だけでなく、「不正なフォーマットのデータ侵入」も同時に防ぐことができる（Parse, don't validate の原則）。

#### 3. スキーマバリデーションライブラリ（Zod など）を活用する
実務の Web アプリケーションでは、Zod などのバリデーションライブラリに組み込まれているブランド型機能を使うケースも増えています。

```typescript
import { z } from "zod";

const UserIdSchema = z.string().startsWith("USR-").brand<"UserId">();
type UserId = z.infer<typeof UserIdSchema>;

// バリデーションとブランド型付与を同時に行う
const userId = UserIdSchema.parse(req.params.userId);
```

### アプローチ比較表

| アプローチ | 実行時コスト | 実装の手軽さ | バリデーション | 推奨シーン |
| :--- | :--- | :--- | :--- | :--- |
| **名前付き引数 (オブジェクト)** | わずか（オブジェクト生成） | 高い | 別途必要 | 通常の関数設計、オプション引数が多い場合 |
| **ブランド型（手動定義）** | 完全ゼロ (0ms / 0byte) | 中 | 別途必要 | パフォーマンス重視、システム横断のIDや通貨 |
| **スマートコンストラクタ / Zod** | バリデーション処理のみ | 中〜高 | 同時に達成 | 外部APIやユーザー入力の受取口（BFF層など） |

---

## まとめ：明日から使えるアクションプラン

1. **型エイリアスで満足しない**:
   - `type UserId = string;` は可読性を上げるだけで、引数の取り違えバグは防げません。
2. **取り違えると危険な値にはブランド型を導入する**:
   - `declare const __brand: unique symbol;` をプロジェクトの共通型定義に 1 つ置くだけで、ゼロコストに堅牢な型安全性が手に入ります。
3. **コンパイル時にバグを落とす**:
   - 人間の「気をつける」というレビューに頼るのではなく、型システムに機械的に検知させる設計を心がけましょう。

---

## 番外編：記事を書いていて自分が一番疑問に思ったこと

実は、今回のサンプルコードを実装していて、自分自身が一番「あれ？」と疑問に引っかかったポイントがありました。

それは、**「なぜ Bad は型注釈（`: UserId`）なのに、Good は型アサーション（`as BrandUserId`）で書く必要があるのか？」** という点です。

```typescript
// Bad: 自然な型注釈
const userId: UserId = "USR-ALICE";

// Good: なぜか as（型アサーション）を使っている
const userId = "USR-ALICE" as BrandUserId;
```

### 1. 型注釈と型アサーションの決定的な違い

一言で言うと、「誰が責任を持つか（コンパイラにお願いするか、人間がねじ伏せるか）」の違いでした。

* **型注釈（`: Type`）**: 「TypeScriptさん、この値が型に合っているか厳密にチェックしてください」
* **型アサーション（`as Type`）**: 「TypeScriptさん、口出ししないで。私が責任を持つから、この型として扱って」

### 2. コードで見る安全性の違い

```typescript
type User = {
  name: string;
  age: number;
};

// 型注釈（: Type）の場合：漏れをコンパイラが怒ってくれる（安全）
const alice: User = {
  name: "Alice",
  // age を書き忘れた！ -> エラー！「プロパティ 'age' がありません」と守ってくれる
};

// 型アサーション（as Type）の場合：型チェックを黙らせてしまう（危険）
const bob = {
  name: "Bob",
  // age を書き忘れたのにエラーにならない！
  // 実行時に bob.age を参照すると undefined になりバグの原因になる
} as User;
```

実務で「`as` は原則使うな」と言われるのは、このようにコンパイラの型チェックをねじ伏せてプロパティ不足などのバグを隠してしまうからです。

### 3. 型注釈 vs 型アサーション 比較表

| 項目 | 型注釈 (`: Type`) | 型アサーション (`as Type`) |
| :--- | :--- | :--- |
| **意味** | 型に適合するか検査をお願いする | この型だと信じてくれと上書きする |
| **主導権** | TypeScript コンパイラ | プログラマー（人間） |
| **安全性** | 高い（タイポやプロパティ不足を即検知） | 低い（型チェックを素通りしてバグになりやすい） |
| **実行時** | JSコンパイル時に消滅（ゼロコスト） | JSコンパイル時に消滅（ゼロコスト） |

### 4. なぜブランド型でだけは as が必要なのか？

ブランド型（`BrandUserId`）は、型空間にのみ存在する架空のプロパティ（`[__brand]: "UserId"`）を持っています。
そのため、生の文字列 `"USR-ALICE"` にはそのプロパティが存在しないため、型注釈（`: BrandUserId`）で直接代入しようとすると、コンパイラに拒絶されてしまいます。

```typescript
// 型注釈だと、架空のプロパティがないためコンパイルエラーになる
const userId: BrandUserId = "USR-ALICE";
// -> エラー: プロパティ '[__brand]' が型 'string' にありません
```

つまり、ブランド型を使うときは「この文字列は正当なユーザーIDであることを私が保証する」という**ブランド付与の儀式**として、あえて `as BrandUserId`（型アサーション）を使用しなければならなかったのです。

「普段は避けるべき `as` だからこそ、ブランド型を生成する境界（外部APIレスポンスのパース時など）でだけ限定的に使う」という設計思想が理解でき、非常にスッキリしました。

### 5. ブランド型の使いすぎ（乱用）はNG：どこに使うべきか？

もう一つ、調べていて痛感したのが「便利だからといって、何でもかんでもブランド型にしてはいけない」ということです。

もしすべての文字列や数値にブランド型をつけてしまうと、以下のような問題が発生します：

* **コードがボイラープレートだらけになる**: あらゆる場所で `as BrandUserName` のようなキャストを書く必要があり、記述量が激増して可読性が落ちる。
* **外部ライブラリとの摩擦**: React や ORM などは普通の `string` や `number` を期待しているため、型変換の手間が爆発する。
* **そもそも型が違えば不要**: 例えば `function register(name: string, age: number)` のように、最初から引数の型が異なっていれば、ブランド型を使わなくても TypeScript が引数逆転を 100% 防いでくれる。

結論として、ブランド型は「基本は使わない」のが健全であり、**「同じプリミティブ型が複数並び、取り違えると致命的な事故（金銭被害・セキュリティ・データ破壊）になる急所」にだけ限定して使う**のが、実務における最もバランスの良いベストプラクティスだと分かりました。

---

> **検証コードを動かしてみたい方はこちら**
> [https://github.com/tiga-ga/sample-code/tree/main/2026-08-26_typescript-branded-types](https://github.com/tiga-ga/sample-code/tree/main/2026-08-26_typescript-branded-types)
> `npm run check` を叩くだけで、ブランド型のエラー検知を即座に確認できます。
