# 第2章 開発環境構築

## Rust開発環境の準備

本章では、Rustの開発環境を整えます。RustのツールチェーンはRustupでインストールします。

### Rustupのインストール

RustupはRustのバージョン管理ツールです。

```bash
# macOS/Linuxの場合
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# インストールを確認
rustup --version
rustc --version
cargo --version

# 例:
# rustup 1.27.1
# rustc 1.85.0
# cargo 1.85.0
```

### Windowsの場合

1. https://win.rustup.rs/x86_64 からインストール
2. インストーラーを実行
3. 「1」を押し、デフォルト設定でインストール

### VS Codeの設定

```bash
# VS CodeでRust Analyzerをインストール
# VS Code > Extensions > Rust Analyzerインストール

# またはコマンドラインから
# 推奨拡張機能:
# - Rust Analyzer
# - C/C++ Extension Pack (C/C++ライブラリとの連携に)
```

### Cargo.tomlの基本構造

```toml
[package]
name = "my-project"
version = "0.1.0"
edition = "2024"
description = "Rustによるシステムプログラミング"
license = "MIT OR Apache-2.0"
authors = ["Your Name <you@example.com>"]

[dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
anyhow = "1.0"

[dev-dependencies]
criterion = "0.5"

[profile.release]
opt-level = 3
lto = true
strip = "symbols"

```

### Cargoの主要コマンド

```bash
cargo init           # 新規プロジェクト作成
cargo build          # ビルド
cargo build --release  # 最適化ビルド（本番向け）
cargo run            # 実行
cargo clippy         # 静的解析
cargo fmt            # コードフォーマット
cargo doc            # ドキュメント生成
cargo test           # テスト実行
cargo check          # コンパイルチェック
cargo clippy --fix   # クリピーの自動修正
cargo --help         # ヘルプ
```

## クイックスタート

```bash
# プロジェクト作成
cargo new rust-sys-app
cd rust-sys-app

# 実行
cargo run

# エディターで開く
code .

# クロップのビルド
cargo build --release
ls target/release/rust-sys-app
```

## まとめ

本章で学んだこと：

- RustupでRustをインストール
- cargoの主要コマンド
- VS Code + Rust Analyzerのセットアップ

次章では、Rustの基本的な構文を学んでいきます。
