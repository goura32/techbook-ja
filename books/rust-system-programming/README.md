# Rustシステムプログラミング入門

> C/C++ の後、次はRustを学ぶ。システムプログラミングを体系的に学ぶ実践書。

## 📖 目次

| 章 | タイトル | 内容 |
|---|---|---|
| [第0章](00_introduction.md) | はじめに | 本書の目的・対象読者・必要環境 |
| [第1章](01_why_rust.md) | なぜRustか | TIOBE指数急上昇、Linuxカーネル採用の背景 |
| [第2章](02_development_environment.md) | 開発環境構築 | Rustup、Cargo、VS Codeのセットアップ |
| [第3章](03_rust_basics.md) | Rustの基礎 | 変数、関数、型システム、パターンマッチング |
| [第4章](04_ownership_and_lifetimes.md) | 所有権とライフタイム | Borrow checkingの仕組み、メモリ安全性 |
| [第5章](05_concurrency_and_async.md) | 並行性と非同期 | スレッド、メッセージパッシング、async/await |
| [第6章](06_pattern_matching_and_generics.md) | 型システム | Enum、Trait、ジェネリクス |
| [第7章](07_ffi_and_c_integration.md) | FFIとC連携 | 外部関数インターフェース、Cライブラリの利用 |
| [第8章](08_performance_optimization.md) | パフォーマンス最適化 | ベンチマーク、最適化レベル、プロファイリング |
| [第9章](09_error_handling.md) | エラーハンドリング | Result/Option、?演算子、カスタムエラー型 |
| [第10章](10_practical_cli_server_project.md) | 実践プロジェクト | CLIツールとTCPサーバーの構築 |

## 🚀 始め方

```bash
# 1. Rustツールチェーンをインストール
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# 2. プロジェクトを作成してビルド
cargo new rust-chatbot
cd rust-chatbot
cargo build --release

# 3. 実行
./target/release/rust-chatbot
```

## 📚 構成

本書は3フェーズで構成されます。

### フェーズ1: 基礎 (第0章〜第2章)
- なぜRustなのかの理解
- 開発環境のセットアップ

### フェーズ2: Rustの核心 (第3章〜第6章)
- 所有権とライフタイム
- 型システムの仕組み
- 非同期処理

### フェーズ3: 実践的な技術 (第7章〜第10章)
- C/Libの連携
- パフォーマンス最適化
- CLIツールとサーバーの構築

## 🛠 技術スタック

| カテゴリー | 技術 |
- 言語: Rust (Edition 2024)
- ビルド: Cargo
- 非同期: Tokio
- エラー処理: anyhow::std::error::Error
- シリアライズ: serde
- バイナリ: serde_json

## LICENSE

MIT License
