# Miri（公式 README 日本語訳）

Rust の未定義動作（undefined behavior）を検出するツール「Miri」の公式 README の、非公式な日本語訳です。Miri は、cargo プロジェクトのバイナリやテストスイートを実行し、安全性の要件を守れていない unsafe コードを検出します。境界外のメモリアクセス、解放後の使用、未初期化データの不正な使用、データ競合などが対象です。

> **ご注意**: この翻訳は Miri の開発チームや Rust Project の確認や公認を受けたものではありません。コマンド、フラグ、環境変数、出力例は原文のまま残しています。実際に使うときは、必ず最新の原文を確かめてください。

## 文書

1. [Miri]({{ '/guide.html' | relative_url }})

## 出典

- 原文: [rust-lang/miri](https://github.com/rust-lang/miri) の `README.md`
- 原著者: Miri の貢献者（The Rust Project）
- ライセンス: MIT または Apache-2.0
- 訳した版: 2026-11-21 時点（コミット `f285a9d`）
