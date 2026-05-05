# 第9章 エラーハンドリング

## Rustのエラーハンドリング

### Result型とOption型

```rust
fn main() {
    let x: Result<i32, String> = Err("error!".to_string());
    let y: Option<i32> = None;
    
    // Resultパターンマッチ
    if let Err(e) = x {
        println!("エラー: {}", e);
    }
    
    // Optionパターンマッチ
    if let Some(v) = y {
        println!("{}", v);
    } else {
        println!("None");
    }
}
```

### ?演算子

```rust
use std::fs::File;
use std::io::BufReader;
use std::io::BufRead;
use std::path::Path;

fn read_file(path: &Path) -> Result<Vec<String>, Box<dyn std::error::Error>> {
    let file = File::open(path)?;  // ファイルオープン
    let reader = BufReader::new(file);  // バファ
    let lines: Vec<String> = reader
        .lines()
        .collect::<Result<Vec<String>, _>>()?;  // 各行
    Ok(lines)
}

fn process_file(path: &Path) -> Result<Vec<String>, Box<dyn std::error::Error>> {
    read_file(path)?.lines()
        .filter(|line| line.contains("Rust"))
        .map(|line| line.to_string())
        .collect::<Vec<String>>()
}
```

### Errorトレイト

```rust
use std::fmt;

#[derive(Debug, PartialEq)]
enum MyError {
    NotFound(String),
    ParseError(String),
    Timeout,
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            MyError::NotFound(msg) => write!(f, "not found: {}", msg),
            MyError::ParseError(msg) => write!(f, "parse error: {}", msg),
            MyError::Timeout => write!(f, "timeout"),
        }
    }
}

impl std::error::Error for MyError {}
```

### anyhowとthiserror

```rust
#[derive(Debug)]
struct AppError {
    code: i32,
    message: String,
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "error {}: {}", self.code, self.message)
    }
}

impl std::error::Error for AppError {}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let err = AppError {
        code: 404,
        message: "not found".to_string(),
    };
    let result: Result<String, AppError> = Err(AppError {
        code: 404,
        message: "not found".to_string(),
    });

    match result {
        Err(e) => println!("エラー: {}", e),
        Ok(v) => println!("OK: {}", v),
    }
    Ok(())
}
```

## まとめ

- Result<T, E>で成功値またはエラーを返す
- ?演算子でエラーを上位に伝播
- custom error type


### errorとthiserrorクレート

errorクレートは、カスタムエラー型を定義するためのマクロを提供します。

```rust
use error::thiserror;

#[thiserror::Error]
enum NetworkError {
    #[error("接続エラー: {0}")]
    Connection(#[src] io::Error),
    
    #[error("タイムアウト: {0}秒")]
    Timeout(u64),
    
    #[error("DNS解決エラー: {0}")]
    Dns(String),
}

#[thiserror::Error]
enum AppError {
    #[error(transparent)]
    Network(#[from] NetworkError),
    
    #[error("パーサーエラー: {0}")]
    Parse(String),
}

fn fetch_data() -> Result<String, AppError> {
    // 接続処理
    Ok("データ取得成功".to_string())
}
```

### エラーチェーン（Error Cause）

Rustではエラーの原因をチェーンで表現できます。

```rust
use std::io::{self, Read};

fn read_config(path: &str) -> Result<String, anyhow::Error> {
    let content = std::fs::read_to_string(path)
        .map_err(|e| anyhow::anyhow!("ファイル読み込み失敗: {}", e))?;
    
    // JSONパース
    let config: serde_json::Value = serde_json::from_str(&content)
        .map_err(|e| anyhow::anyhow!("JSONパース失敗: {}", e))?;
    
    Ok(config["host"].as_str().unwrap_or("").to_string())
}
```

### エラーハンドリングパターン

代表的なエラーハンドリングパターンをまとめます。

1. ** Resultを使う** — functionの戻値をResult<T, E>にする
2. ** ?演算子** — エラーを上位に即座に伝播
3. **unwrap() / unwrap_or()** — デバッグ時に使用
4. **map_err()** — エラー型を変換
5. **ok_or()** — OptionをResultに変換
6. **expect()** — エラーメッセージ付きでunwrap

これらを適切に使い分けることが、堅牢なRustプログラムを書くために必要です。
