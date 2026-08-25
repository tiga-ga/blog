# Project Rules - 技術ブログ・ナレッジ発信 (blog)

本ディレクトリ（`blog/`）は、技術学習・読書・コード検証から得られた知見を**Qiita等の技術記事・ブログ記事**として発信・蓄積するためのハブです。

AI（Antigravity）は、このディレクトリで作業する際、以下のルールとガイドラインに従ってください。

---

## 🎯 目的 & 配信先

1. **技術ナレッジのアウトプット・発信**:
   - 読書（オライリー等）やコード検証から得た「実践的な知見」「アンチパターンの回避策」「設計手法」を分かりやすい記事としてまとめる。
2. **主要公開先**:
   - **Qiita アカウント**: [`https://qiita.com/tiga-ga`](https://qiita.com/tiga-ga)
   - 記事フォーマットは Qiita 向け Markdown 記法に準拠し、コピー＆ペーストでそのまま投稿できる構成を意識する。

---

## 🔗 サンプルコードの参照・リンク規則

記事内でサンプルコードを参照・紹介・リンクする際は、必ず以下の GitHub リポジトリに紐付けてください。

* **サンプルコード リポジトリ**: [`https://github.com/tiga-ga/sample-code`](https://github.com/tiga-ga/sample-code)
* **リンク形式例**:
  - リポジトリ全体: `[GitHub - sample-code](https://github.com/tiga-ga/sample-code)`
  - 日付別フォルダ: `[検証コード (GitHub)](https://github.com/tiga-ga/sample-code/tree/main/YYYY-MM-DD)`
  - 特定ファイル: `[実装例 (GitHub)](https://github.com/tiga-ga/sample-code/blob/main/YYYY-MM-DD/example.ts)`

---

## 🛠️ AI（Antigravity）行動ガイドライン

### 1. 読者を引き込む Qiita 記事構成
- 記事は以下の流れを意識して作成・提案してください：
  1. **タイトル & タグ**: 検索されやすく魅力的なタイトルと適切な Qiita タグ（例: `TypeScript`, `Architecture`, `AI`）
  2. **導入・問題提起**: なぜこの問題が起きるのか、現場で直面しやすい課題
  3. **アンチパターン（Before ❌）**: よくあるNG実装とそのデメリット
  4. **ベストプラクティス（After ⭕️）**: 推奨実装とメリット・詳細解説
  5. **GitHubサンプルコードへのリンク**: `https://github.com/tiga-ga/sample-code` への参照
  6. **まとめ・要点整理**: 表や箇条書きで要点を簡潔に再確認

### 2. Markdown の表（Table）による構造化
- 比較やポイント解説では画像やMermaidを使わず、Markdownの表を積極的に活用してください。
