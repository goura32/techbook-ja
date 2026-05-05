# 第10章 実践プロジェクト

## 実践プロジェクト - CLIツールとTCPサーバー

本章では、本書で学んだすべてを総動員して、**CLIファイル検索ツールとTCPサーバーを同時に構築します**。

### システムアーキテクチャ

```
┌─────────────┐    ┌────────────┐    ┌───────┐
│  CLIツール   │    │ TCPサーバー │    │ ファイル│
│ (Rust CLI)  │──▶│ (Rust Server)│──▶│ システム│
└─────────────┘    └────────────┘    └───────┘
                           │
                    ┌───────┐
                    │  Redis │
                    │(メモリ)│
                    └───────┘
```

## プロジェクトセットアップ

### プロジェクト作成

```bash
cargo new rust-chatbot --bin
cd rust-chatbot
```

### Cargo.toml

```toml
[package]
name = "rust-chatbot"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1.0"

[dev-dependencies]
assert_cmd = "2.0"
```

## CLIツール部分

### main.rs

```rust
use std::io::{self, Write, BufRead};
use std::sync::Arc;
use std::collections::HashMap;
use std::fs;
use std::path::Path;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct SearchResult {
    filename: String,
    content: String,
    path: String,
}

/// 指定されたディレクトリからキーワードを含むファイルを再帰的に検索する
fn search_files(root: &Path, keyword: &str) -> Vec<SearchResult> {
    let mut results = vec![];
    search_dir(root, keyword, &mut results);
    results
}

fn search_dir(dir: &Path, keyword: &str, results: &mut Vec<SearchResult>) {
    if let Ok(entries) = fs::read_dir(dir) {
        for entry in entries.flatten() {
            let path = entry.path();
            if path.is_dir() {
                search_dir(&path, keyword, results);
            } else if let Ok(content) = fs::read_to_string(&path) {
                if content.contains(keyword) {
                    results.push(SearchResult {
                        filename: path.file_name()
                            .unwrap_or_default()
                            .to_string_lossy()
                            .to_string(),
                        content: content
                            .lines()
                            .find(|l| l.contains(keyword))
                            .unwrap_or("")
                            .to_string(),
                        path: path.to_string_lossy().to_string(),
                    });
                }
            }
        }
    }
}

fn main() -> anyhow::Result<()> {
    let args: Vec<String> = std::env::args().collect();
    match args.len() {
        1 => print_help(),
        2 => search_files_for_keyword(&args[1]),
        _ => print_help(),
    }
    Ok(())
}

fn print_help() {
    println!("使用方法: rust-chatbot <キーワード> [<検索ディレクトリ>]");
}

fn search_files_for_keyword(keyword: &str) {
    let root = Path::new(".");
    let files = search_files(root, keyword);
    for file in files {
        println!("{}: {}", file.path, file.content.lines().next().unwrap_or(""));
    }
}
```

## TCPサーバー部分

### TCPサーバーの実装（main.rsに追加）

```rust
use tokio::net::TcpListener;
use tokio::net::TcpStream;
use tokio::io::{AsyncWriteExt, AsyncReadExt};
use std::sync::Arc;
use tokio::sync::Mutex;
use std::collections::HashMap;
use std::time::Instant;

/// クライアント管理
struct ChatState {
    // username → { last_seen, message_count }
    clients: HashMap<String, ClientInfo>,
    history: Vec<String>,
}

struct ClientInfo {
    last_seen: Instant,
    message_count: usize,
}

impl ChatState {
    fn new() -> Self {
        Self {
            clients: HashMap::new(),
            history: vec![],
        }
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let state = Arc::new(Mutex::new(ChatState::new());
    
    // TCPサーバーを初期化
    let listener = TcpListener::bind("0.0.0.0:8080").await?;
    
    println!("TCPサーバーを8080番ポートで開始します...");
    
    loop {
        let (mut stream, addr) = listener.accept().await?;
        println!("接続されました: {}", addr);
        
        let state = state.clone();
        tokio::spawn(async move {
            let mut buf = vec![0u8; 1024];
            let len = stream.read(&mut buf).await.unwrap_or(0);
            if len > 0 {
                let msg = String::from_utf8_lossy(&buf[..len]);
                let response = format!("Echo: {}", addr);
                stream.write_all(response.as_bytes()).await.unwrap();
            }
        });
    }
}
```

## Dockerでのデプロイ

### Dockerfile

```dockerfile
FROM rust:latest AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/rust-chatbot /usr/local/bin/
ENTRYPOINT ["rust-chatbot"]
```

## まとめ

本書で学んだこと：

- Rustの基本構文
- 所有権とライフタイム
- パターンマッチング
- ジェネリクス
- 非同期プログラミング
- パフォーマンス最適化
- エラーハンドリング

これで「Rustシステムプログラミング入門」は完結です。
おめでとうございます！

## 今後のステップ

これらのリソースでさらに学んでください：

- [The Rust Programming Language (公式ドキュメント)](https://doc.rust-lang.org/book/)
- [Rust by Example (例題集)](https://doc.rust-lang.org/rust-by-example/)
- [crates.io (パッケージマネージャー)](https://crates.io/)
- [Rust Cookbook (レシピ集)](https://rust-lang.github.io/rust-cookbook/)

### 推奨の次ステップ

1. **Tokio非同期フレームワークを深く学ぶ**
   - TCP/UDPソケット
   - HTTPクライアント/サーバー

2. **Webサーバーを構築**
   - Actix-web
   - Axum

3. **組込みプログラミング**
   - stm32
   - ESP32

4. **OSカーネルの開発**
   - xv6でカーネルをRustで書き直
