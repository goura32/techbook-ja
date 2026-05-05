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


### Rustupの設定ファイルの確認

Rustupとコンポーネントの設定ファイルを確認します。

```bash
# インストール先を確認
rustup show

# インストールするツールチェーンを確認
rustup default stable

# コンポーネントの確認
rustup component list --installed
```

### Cargo.tomlの詳細設定

Cargo.tomlの詳細な設定項目：

```toml
[package]
name = "my-project"
version = "0.1.0"
edition = "2021"
description = "Rustによるシステムプログラミング"

# ライセンス
license = "MIT OR Apache-2.0"
authors = ["Your Name <you@example.com>"]
homepage = "https://example.com"
repository = "https://github.com/user/repo"

# Documentation
documentation = "https://docs.rs/my-project"
readme = "README.md"
keywords = ["system", "networking", "tool"]
categories = ["command-line-utilities"]

# 実行ファイルの設定
[[bin]]
name = "my-app"
path = "src/main.rs"

# ライブラリ設定
[lib]
name = "my_lib"
path = "src/lib.rs"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }

[build-dependencies]
cc = "1.0"
```

### Rustのバージョン管理

Rustupで異なるバージョンのツールチェーンを使用できます。

```bash
# 安定版（stable）を使用
rustup default stable

# Nightly版の使用
rustup default nightly

# MSRV（Minimum Supported Rust Version）を指定
rustup install 1.70.0
rustup override set 1.70.0
```

### クロスコンパイル

Rustupとtargetを使えば、異なるOSやアーキテクチャにクロスコンパイルできます。

```bash
# 利用可能なターゲット一覧
rustup target list

# Windows(x86_64-pc-windows-gnu)にクロスコンパイル
rustup target add x86_64-pc-windows-gnu
cargo build --target x86_64-pc-windows-gnu

# aarch64にクロスコンパイル
rustup target add aarch64-unknown-linux-gnu
cargo build --target aarch64-unknown-linux-gnu
```

### Rustのエラーメッセージの読み方

Rustのエラーメッセージは非常に詳細です。以下のように理解しましょう。

```
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:2:9
  |
1 | let s = String::from("hello");
  |     - move occurs because `s` has type `String`, which does not implement the `Copy` trait
2 | let r = &s;
  |         ^^  value borrowed here
```

- `error[E0382]`：エラー番号
- `borrow of moved value`：説明（移動した値を借用した）
- `--> src/main.rs:2:9`：エラー位置
- 行番号付きのコードと`^`マーカーで、問題の箇所を示しています。
