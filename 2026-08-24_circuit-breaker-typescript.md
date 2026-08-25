---
title: 外部API遅延で画面真っ白になってない？サーキットブレーカーで守る耐障害性設計入門
tags: TypeScript, SRE, Architecture, Node.js, 初心者向け
author: tiga-ga (https://qiita.com/tiga-ga)
---

# 外部API遅延で画面真っ白になってない？サーキットブレーカーで守る耐障害性設計入門

---

## 【著者メモ】はじめに：なぜこれを学ぼうと思ったのか？

* **きっかけ**: 外部APIが遅延・ダウンした時、画面全体が真っ白（500エラー）になって道連れになる「連鎖障害（Cascading Failure）」を防ぎたかったため。
* **今回のゴール**: 今回はサーキットブレーカーの概念を理解するために、実際に TypeScript でサーキットブレーカーを実装してみることにしました。その内容を、記事にまとめていきます！
* **参考**: [サーキット ブレーカー パターン](https://learn.microsoft.com/ja-jp/azure/architecture/patterns/circuit-breaker)にもまとまっています。

> **GitHub サンプルコード**:
> [sample-code (2026-08-24_circuit-breaker-resilience)](https://github.com/tiga-ga/sample-code/tree/main/2026-08-24_circuit-breaker-resilience)

---

## サーキットブレーカーとは？（仕組み & 3つの状態）

家庭の分電盤にあるブレーカーと同じで、**「外部APIの異常を検知したら回路を遮断し、0msで代替データ（フォールバック）を返す」** 仕組みです。

例：外部APIが1秒で応答しないとタイムアウトして遮断 → 2秒休養 → 1回お試し → 成功なら再稼働、失敗ならまた遮断

```text
[1. 通常稼働: CLOSED] ──(外部APIが2回連続失敗/タイムアウト)──> [2. 遮断: OPEN (0msで代替品を即返却)]
       ▲                                                              │
       │                                                         (2秒休養経過)
       │                                                              ▼
       └───(テスト通信成功)─────────────────────────── [3. お試し: HALF_OPEN]
```

| 状態（State） | モード | 動作 |
| :--- | :--- | :--- |
| **1. CLOSED** | 通常稼働 | 普通に外部APIへ通信する。外部APIの失敗（エラーやタイムアウト）が続くと OPEN へ。 |
| **2. OPEN** | 遮断中 | **通信を一切行わない**。0ms で即座に代替品（フォールバック）を返す。 |
| **3. HALF_OPEN** | お試し | 2秒後に1回だけ試験送信。成功なら CLOSED、失敗なら OPEN へ。 |

---

### サーキットブレーカーが防ぐ「4つの大惨事」

```text
【従来の地獄（ドミノ倒し）】
外部おすすめAPIが遅延 -> 自社サーバのスレッドが満杯 -> 決済・ログインまで全滅 -> サイト全体が真っ白

【サーキットブレーカー導入後】
外部おすすめAPIが遅延 -> 即座に遮断！ -> 0msで人気商品を代替表示 -> 決済・ログインは元気に継続
```

| 防げる大惨事 | 導入しないとどうなる？（地獄） | サーキットブレーカーが防ぐ仕組み |
| :--- | :--- | :--- |
| **1. 連鎖障害（ドミノ倒し）** | おすすめAPIが死んだだけで、ログインや決済まで全滅してサイト全体が落ちる。 | 異常検知で回路を切り離し、**障害をその場に閉じ込めて延焼を防ぐ**。 |
| **2. サーバのリソース枯渇** | 遅い相手を待ち続けるせいで接続枠が埋まり、新規ユーザー全員が門前払いになる。 | 通信をスキップし、**「0ms」で即座に応答してリソースを即座に解放**する。 |
| **3. 相手へのトドメ（リトライの嵐）** | 相手が過負荷で倒れかけているのに、全員がリトライを連打して完全に窒息死させる。 | 通信を遮断して相手に**回復のための息継ぎ時間（クールダウン）を与える**。 |
| **4. 最悪のユーザー体験** | 画面全体に「500エラー」が出て、ユーザーが離脱・購入を諦める。 | 画面を壊さず、**静的な代替品（フォールバック）を表示してサービスを継続（縮退運転）**。 |

---

## [Before] 外部API連携の4大アンチパターン

サーキットブレーカーがない現場でやりがちな「親切心でリトライして自滅するコード」の問題点です。

* 該当コード: [bad_example.ts (GitHub #L24-L56)](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/bad_example.ts#L24-L56)

| アンチパターン | 該当コード | 何が起きるか？ |
| :--- | :--- | :--- |
| **1. タイムアウトなし** | [bad_example.ts: L36](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/bad_example.ts#L36) | 相手が遅いと無限に待ち続け、サーバの接続枠を食いつぶす。 |
| **2. 相手が死んでるのに連打** | [bad_example.ts: L32-L52](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/bad_example.ts#L32-L52) | 相手が倒れているのに全員が毎回アクセスして自滅する。 |
| **3. 固定間隔リトライ** | [bad_example.ts: L50](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/bad_example.ts#L50) | 全員が 0.5秒後に一斉リトライして混雑を悪化させる。 |
| **4. 画面全体のクラッシュ** | [bad_example.ts: L45-L47](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/bad_example.ts#L45-L47) | 一部のエラーで画面全体を 500 エラーにして店を閉める。 |

---

### [After] 推奨される4大ベストプラクティス

| 観点 | Before (問題点) | After (解決策) | 該当コード / 解決策 |
| :--- | :--- | :--- | :--- |
| **待機時間** | タイムアウト未設定で無限に待つ。 | **短時間タイムアウト（例: 300ms）** でスパッと打ち切る。 | [circuit_breaker.ts: L53-L73](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/circuit_breaker.ts#L53-L73) (`Promise.race`) |
| **障害時の通信** | 相手が落ちていても全員がリクエストを連打。 | **サーキットブレーカー（OPEN）** で通信を遮断し、0ms で即答。 | [circuit_breaker.ts: L78-L107](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/circuit_breaker.ts#L78-L107) (Fast-Fail) |
| **リトライ制御** | 0.5秒ごとに固定リトライして混雑悪化。 | **指数バックオフ ＋ Jitter（ランダム揺らぎ）** で分散。 | [circuit_breaker.ts: L146-L155](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/circuit_breaker.ts#L146-L155) (`sleepWithBackoff`) |
| **エラー時の対応** | 例外を投げて画面全体を 500 エラーにする。 | **フォールバック関数（縮退運転）** で静的な代替品を表示。 | [good_example.ts: L21-L32](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/good_example.ts#L21-L32) (`defaultRecommendationsFallback`) |

---

## 実際に動かしてみて分かったこと・気づき

実際にベンチマークスクリプト（[`run_comparison.ts`](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/run_comparison.ts)）を動かしてみて、机上の理論では分からなかったリアルな学びがありました。

### 1. Badコードの恐ろしさ（雪だるま式の連鎖障害）
* **地獄のタイムライン**: 1人につき「1.5秒待機 -> 0.5秒リトライ -> 1.5秒待機 -> 0.5秒リトライ -> 1.5秒待機」で **計5.5秒（5508ms）** も拘束される。
* **学習しないシステム**: 相手がダウンしているのに状態を記憶しないため、2人目・3人目も全く同じ5.5秒ずつ待たされ、**わずか3人で16.5秒もサーバーがハングアップ** して全員500エラーになりました。
* **本番への恐怖**: もし1秒間に100人が訪れるECサイトなら、一瞬で接続プールがパンクして決済やログインまで道連れになる「連鎖障害（Cascading Failure）」の怖さを身をもって実感しました。

### 2. Goodコード（サーキットブレーカー）の圧倒的安心感
* **0msで即座に縮退運転**: 2回連続で失敗した瞬間にブレーカーが落ち（`OPEN`）、3人目以降は外部APIへの通信をスキップして **「0ms」で代替品（静的な人気商品リスト）を即座に返却** してくれました。
* **フォールバック用returnの重要性**: 外部APIが遅延・ダウンした際、本来のreturnを待たずにタイムアウトで切り上げ、身代わりとなるフォールバックのreturn（静的データ）を即座に返すことで、画面クラッシュを防ぎユーザー体験を死守できる重要性を実感できました。
* **自律的な自己修復**: 外部APIが復旧すると、2秒のクールダウン後に `HALF_OPEN`（お試し）で1回だけ通信を試み、成功したら自動で `CLOSED`（完全復旧）へと元通りに戻る動きが確認できました。

### 3. 設計上の大きな発見（関心の分離・高階関数）
* **Badの構造**: 「APIを叩く処理」と「リトライ・エラー制御」が1つのファイルにごちゃ混ぜ（密結合）。
* **Goodの構造**: 家電とコンセントのブレーカーのように、**「安全装置（`CircuitBreaker`）」の中に「実行したい関数（`flakyService`）」を引数として放り込む構造（関心の分離 / 高階関数）** になっています。
* **使い回しの良さ**: この設計のおかげで、レコメンドAPIだけでなく、決済APIや天気APIなど、プロジェクト内のあらゆる非同期処理に `breaker.execute(...)` の1行で耐障害性を付与できるのが素晴らしいと感じました。

---

## まとめ & 明日からのアクション

* [ ] 外部API連携には必ず短めのタイムアウトを設定する。
* [ ] 相手が倒れた時はリトライを連打せず、サーキットブレーカーで遮断して相手を休ませる。
* [ ] エラーで画面全体を落とさず、フォールバック（縮退運転）で主要機能を死守する。

---

## 番外編: ラッパー関数とは？ なぜジェネリクス `<T, TArgs>` を使うのか？

* 該当箇所: [circuit_breaker.ts: L23-L24](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/circuit_breaker.ts#L23-L24)

### 1. ラッパー関数とは？（スマホケースの例え）
ある関数を「包んで（ラップして）」、元の処理コードはいじらずに**新しい機能（タイムアウト、ログ計測、キャッシュ、サーキットブレーカー等）を外側から付け足す関数**のことです。
（スマホ本体に、衝撃から守る「スマホケース」を被せるイメージです）。

### 2. なぜジェネリクス `<T, TArgs extends unknown[]>` が必須なのか？
1. **`any` を使わないため**: 戻り値の型推論が消滅し、タイポ（`res.recomends` 等）をコンパイラが見逃して実行時バグになるのを完全防止。
2. **型を固定しないため**: レコメンド専用にせず、決済APIや天気APIでも同じブレーカーを 1 行で使い回すため。
3. **型の完全一致保証**: 「本命処理が返す型」と「フォールバック関数が返す型」が **完全一致していないとコンパイルエラー** にしてくれる。

### 3. サンプルコードで理解するラッパー関数

今回のサーキットブレーカー（[`circuit_breaker.ts: L78-L107`](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/circuit_breaker.ts#L78-L107)）の `execute` メソッドこそが、まさにこのラッパー関数です。

#### サーキットブレーカーの実装（型定義とラッパー構造）
```typescript
// 1. 任意の関数・引数・戻り値を受け付けるジェネリクス型
export type ActionFunction<T, TArgs extends unknown[]> = (...args: TArgs) => Promise<T>;
export type FallbackFunction<T, TArgs extends unknown[]> = (error: Error, ...args: TArgs) => Promise<T> | T;

// 2. 本命関数 (actionFn) と 身代わり (fallbackFn) を包むラッパーメソッド
async execute<T, TArgs extends unknown[]>(
  actionFn: ActionFunction<T, TArgs>,
  fallbackFn: FallbackFunction<T, TArgs>,
  ...args: TArgs
): Promise<T> {
  // 遮断中なら通信せず 0ms で fallbackFn を実行
  // 通常時は callWithTimeout でタイムアウト制限付きで実行
  // -> 本命が成功しても、身代わりが動いても、必ず型 T のデータが安全に返る！
}
```

#### 実際の使い方（[`good_example.ts: L56-L62`](https://github.com/tiga-ga/sample-code/blob/main/2026-08-24_circuit-breaker-resilience/good_example.ts#L56-L62)）
ジェネリクスのおかげで、元の関数の引数型・戻り値型が **100% 保持されたまま** 安全に実行されます。

```typescript
// 本命関数: (id: string) => Promise<UserRecommendationsResponse>
// 身代わり: (err: Error, id: string) => UserRecommendationsResponse
const res = await breaker.execute(
  flakyService,
  defaultRecommendationsFallback,
  "user-123" // 引数 (string) の型チェックが効く
);
// -> 戻り値の型も自動的に Promise<UserRecommendationsResponse> になる！
```
