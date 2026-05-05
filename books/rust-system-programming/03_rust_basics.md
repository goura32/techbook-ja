# 第3章 Rustの基礎

## Rustの基礎構文

本章では、Rustの基本的な構文を学びます。

### Hello World

```rust
// hello.rs
fn main() {
    println!("Hello, Rustシステムプログラミング!");
}
```

### 変数と型

```rust
fn main() {
    // 変数はデフォルトで不変（immutable）
    let x = 42;
    println!("x = {}", x);
    
    // 可変変数にするにはmutキーワードを使用
    let mut y = 10;
    y += 5;
    
    // 型推論で型を省略可能
    let name: &str = "Rust";
    let price: f64 = 9.99;
    let is_active: bool = true;
    
    // 配列
    let arr: [i32; 5] = [1, 2, 3, 4, 5];
    
    // ベクトル（ヒープ上）
    let vec = vec![1, 2, 3, 4, 5];
    
    // ハッシュマップ
    use std::collections::HashMap;
    let mut map = HashMap::new();
    map.insert("key1", 100);
    map.insert("key2", 200);
    
    println!("{:?}", map);
}
```

### 関数

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 {
        None  // 0除算防止
    } else {
        Some(a / b)
    }
}

// 戻り値が式の場合はreturn;で戻り値を返す
// 式の場合はreturn;を省略可能
fn factorial(n: u64) -> u64 {
    if n <= 1 {
        1
    } else {
        n * factorial(n - 1)
    }
}

fn main() {
    println!("{}", add(1, 2));     // 3
    println!("{:?}", divide(10.0, 3.0)); // Some(3.333...)
    println!("{:?}", divide(10.0, 0.0)); // None
    println!("{}", factorial(5));   // 120
}
```

### 条件分岐

```rust
fn main() {
    // if-else
    let x = 5;
    if x < 0 {
        println!("負の数");
    } else if x == 0 {
        println!("ゼロ");
    } else {
        println!("正の数");
    }
    
    // match（パターンマッチング）
    match x {
        1 => println!("イチ"),
        5 => println!("ゴ"),
        10..=100 => println!("10以上100以下"),
        _ => println("_default"),
    }
}
```

## まとめ

本章で学んだこと：

- Rustの変数はデフォルト不変
- 型推論、関数、パターンマッチングの基本

次章では、所有権とライフタイムを深く学びます。
