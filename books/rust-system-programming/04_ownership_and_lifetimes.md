# 第4章 所有権とライフタイム

## 所有権の基本概念

本章では、Rustの最も特徴的な仕組みである「所有権」を学びます。これはRustのメモリ安全性を保証する核となる概念です。

### 所有権とは

所有権（Ownership）とは、**各値にはそれ唯一の「所有者」がいて、所有者がスコープを抜けると値が破棄される**というルールです。

```rust
fn main() {
    {
        let s = String::from("hello");  // sが"hello"の所有者
        println!("{}", s);
    }  // sのスコープ終了、"hello"が自動的に破棄
}  // sはもう使えない
```

### 3つのルール

1. **各値には必ず1人の所有者がいる
2. **同時に複数の所有者はいない**
3. **所有者がスコープを抜けると値が破棄される**

### 所有権の移動

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // 所有権がs1からs2に移動
    
    // println!("{}", s1);  // エラー！s1はもはや所有していない
    println!("{}", s2);   // OK
}
// s2が破棄される
```

### クローン

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();  // 明示的なコピー
    println!("s1 = {}, s2 = {}", s1, s2);
}
```

## パターンマッチング

### Enumの基礎

```rust
enum Color {
    Red,
    Green,
    Blue,
    Rgb { r: u8, g: u8, b: u8 },  // タプルバリアント
}

fn main() {
    let c = Color::Rgb { r: 255, g: 0, b: 0 };
    match c {
        Color::Red => println!("赤"),
        Color::Green => println!("緑"),
        Color::Blue => println!("青"),
        Color::Rgb { r, g, b } => println!("RGB({}, {}, {})", r, g, b),
    }
}
```

### OptionとResult

```rust
fn get_item(items: &[&str], index: usize) -> Option<&str> {
    items.get(index).copied()
}

fn parse_int(s: &str) -> Result<i32, String> {
    s.parse::<i32>().map_err(|e| e.to_string())
}

fn main() {
    let data = vec!["apple", "banana", "cherry"];
    
    // Optionの処理
    if let Some(item) = get_item(&data, 3) {
        println!("見つかった: {}", item);
    } else {
        println!("見つかりませんでした");
    }
    
    // 結果の処理
    let item = match parse_int("42") {
        Ok(n) => n,
        Err(e) => {
            println!("エラー: {}", e);
            0
        }
    };
    println!("{}", item);  // 42
}
```

## ライフタイム

### 借用（borrowing）のルール

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

// コンパイルエラーの可能性！なぜなら戻り値のライフタイムが不明
// 正しい書き方:
fn longest_with_lifetime<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

### ライフタイムのデコレータ

```rust
struct Text<'a> {
    content: &'a str,
    author: &'a str,
}

fn main() {
    let text = "Hello, Rust!";
    let t = Text {
        content: text,
        author: "Rustlang",
    };
    println!("{}", t.content);
}
```

## まとめ

本章で学んだこと：

- 所有権のルール
- 所有権の移動とクローン
- パターンマッチングの強力な機能

次章では、並行性と非同期を学びます。

## 借用と参照

### 借用（Borrowing）の概念

所有権を移動せずにデータをアクセスすることを「借用」と呼びます。

```rust
fn main() {
    let s = String::from("hello");
    
    // 不変参照（&）
    let r1 = &s;
    let r2 = &s;
    println!("{} {}", r1, r2);  // 複数の不変参照は可能
    
    // 可変参照（&mut）は1つのみ
    let mut s2 = String::from("world");
    let r3 = &mut s2;
    *r3 = "changed";
    // let r4 = &mut s2;  // エラー！
}
```

### 借用の規則

1. 参照はいつでも**一つ以下の可変参照**または**複数の不変参照**のどちらかを生成可能
2. 参照は常に有効で正しいデータを指す必要がある

```rust
fn main() {
    let mut data = String::from("hello");
    let r1 = &data;
    let r2 = &data;
    // data.push_str(" world");  // エラー！不変参照が存在中に可変参照は作れない
    println!("{} and {}", r1, r2);
}
```

## ライフタイムの詳細

### ライフタイム注釈の必要性

コンパイラが所有権の寿命を追跡できるよう、ライフタイム注釈が必要です。

```rust
// 注釈なしでもコンパイラが推論できる場合
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

// 明示的なライフタイム注釈
fn longest_with_lifetime<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let s1 = String::from("long string");
    let result;
    {
        let s2 = String::from("xyz");
        result = longest_with_lifetime(s1.as_str(), s2.as_str());
        // ここでs2の寿命が切れると、resultの寿命も切れる
    }
    println!("Longest string is {}", result);
}
```

### 構造体でのライフタイム

```rust
struct Text<'a> {
    content: &'a str,
    author: &'a str,
}

impl<'a> Text<'a> {
    fn display(&self) {
        println!("Author: {}, Content: {}", self.author, self.content);
    }
}

fn main() {
    let name = String::from("Alice");
    let text = Text {
        content: "Rust is great!",
        author: name.as_str(),
    };
    text.display();
}
```

## 所有権とパフォーマンス

所有権システムのオーバーヘッドはコンパイル時に解消されます。実行時への影響は最小限です。

### 所有権の検証はコンパイル時に行われる

```rust
// このコードはコンパイルエラーになる
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // 所有権が移動
    println!("s1 = {}", s1);  // コンパイルエラー！s1はもう所有していない
}
```

### 所有権を活用した安全なコードの例

```rust
struct Buffer {
    data: Vec<u8>,
    position: usize,
}

impl Buffer {
    fn new(size: usize) -> Self {
        Buffer {
            data: vec![0u8; size],
            position: 0,
        }
    }
    
    fn write(&mut self, bytes: &[u8]) -> Result<(), String> {
        if self.position + bytes.len() > self.data.len() {
            return Err("Buffer overflow!".to_string());
        }
        self.data[self.position..self.position + bytes.len()].copy_from_slice(bytes);
        self.position += bytes.len();
        Ok(())
    }
    
    fn read(&self, offset: usize, length: usize) -> Option<&[u8]> {
        if offset + length <= self.position {
            Some(&self.data[offset..offset + length])
        } else {
            None
        }
    }
}

fn main() {
    let mut buf = Buffer::new(1024);
    buf.write(b"Hello, Rust!").unwrap();
    let data = buf.read(0, 12).unwrap();
    println!("{}", String::from_utf8_lossy(data));
}
```
