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
