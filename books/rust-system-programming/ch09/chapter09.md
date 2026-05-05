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
    let result = Err(app);
    
    match result {
        Err(e) => println!("エラー: {}", e),
        Ok(v) => println!("{}", v),
    }
    Ok(())
}
```

## まとめ

- Result<T, E>で成功値またはエラーを返す
- ?演算子でエラーを上位に伝播
- custom error type
