# Miri（公式 README 日本語訳）

> **これは [rust-lang/miri](https://github.com/rust-lang/miri) の README の、非公式な日本語訳です。** Miri の開発チームや Rust Project による確認や公認を受けたものではありません。このリポジトリにツール本体のコードはありません。
>
> **読むのはこちら: https://rustsecjp.github.io/miri-ja/**
>
> This is an unofficial Japanese translation of the Miri README. It does not contain the tool itself.

Miri は、Rust の未定義動作（undefined behavior）を検出するツールです。cargo プロジェクトのバイナリやテストスイートを実行し、安全性の要件を守れていない unsafe コードを検出します。境界外のメモリアクセス、解放後の使用、未初期化データの不正な使用、データ競合などが対象です。

原文の README の訳は [guide.md](guide.md) にあります。この README は日本語訳の案内だけを載せています。

## 訳した版

| | |
|---|---|
| 原文の版 | 2026-11-21 時点（コミット [`f285a9d`](https://github.com/rust-lang/miri/commit/f285a9dc42f3dfd46f81de66cba5907c665478bb)） |
| 翻訳日 | 2026-11-22 |

原文がこの版より後に更新されている場合、訳は古い内容のままです。

## 出典とライセンス

- 原文: https://github.com/rust-lang/miri の `README.md`
- 原著者: Miri の貢献者（The Rust Project）
- ライセンス: MIT または Apache-2.0。全文は [LICENSE-MIT.txt](LICENSE-MIT.txt) と [LICENSE-APACHE.txt](LICENSE-APACHE.txt)。
- 訳した版: 2026-11-21 時点（コミット f285a9dc42f3dfd46f81de66cba5907c665478bb）
- 変更点: 日本語への翻訳。見出しに原文と同じリンク先の目印を付与。原文リポジトリ内のファイルへの相対リンクを、原文リポジトリの URL に変更。
- 図（`miri-sticker.png`）は原文のまま収録しています。原文の表記では、絵は Paige Losare-Lusby 作、Creative Commons Attribute-ShareAlike です。

出典の詳細は [NOTICE.md](NOTICE.md) にあります。誤訳の指摘や訳語の提案は、Issue か Pull Request でお願いします。

RustSecJP は非公式のコミュニティです。Miri の開発チーム、Rust Project、Rust Foundation とは提携していません。
