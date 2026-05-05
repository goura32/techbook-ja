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

## Rustの基本構文の詳細

### 制御構文

Rustには強力な制御構文が用意されています。

#### if式（式であり値を返す）

```rust
fn main() {
    let number = 7;
    
    // ifは式として使える
    let result = if number % 4 == 0 {
        "multiple of 4"
    } else if number % 3 == 0 {
        "multiple of 3"
    } else {
        "not a multiple of 3 or 4"
    };
    
    println!("The number is {}", result);
}
```

#### ループ構文

```rust
fn main() {
    // 無限ループ（明示的にbreakが必要）
    let mut count = 0u32;
    loop {
        count += 1;
        if count == 5 {
            break;
        }
    }
    
    // forループ（イテレータ経由）
    let arr = [10, 20, 30, 40, 50];
    for item in arr.iter() {
        println!("Value: {}", item);
    }
    
    // rangeによるループ
    for i in 1..=10 {
        if i % 3 == 0 {
            print!("{} ", i);
        }
    }
    println!(); // 4, 6, 8, 10 のように3の倍数を出力
}
```

#### matchの高度な使い方

```rust
fn main() {
    let score = 85;
    
    match score {
        0 => println!("Zero"),
        1..=10 => println!("Small number"),
        50..=90 => println!("Middle range"),
        91..=100 => println!("High score"),
        _ => println!("Out of range"),
    }
    
    // 値のバインディング
    let number = 42;
    match number {
        x if x % 2 == 0 => println!("{} is even", x),
        x => println!("{} is odd", x),
    }
    
    // タプルのパターンマッチング
    let point = (3, 4);
    match point {
        (0, y) => println!("On Y axis: {}", y),
        (x, 0) => println!("On X axis: {}", x),
        (x, y) => println!("At ({}, {})", x, y),
    }
}
```

### 構造体とメソッド

```rust
struct Point {
    x: f64,
    y: f64,
}

struct Circle {
    center: Point,
    radius: f64,
    color: String,
}

impl Circle {
    // 関連関数（コンストラクタ）
    fn new(x: f64, y: f64, radius: f64) -> Self {
        Circle {
            center: Point { x, y },
            radius,
            color: "black".to_string(),
        }
    }
    
    // インスタンスメソッド
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
    
    fn circumference(&self) -> f64 {
        2.0 * std::f64::consts::PI * self.radius
    }
    
    fn set_color(&mut self, color: &str) {
        self.color = color.to_string();
    }
}

fn main() {
    let c = Circle::new(0.0, 0.0, 5.0);
    println!("Area: {:.2}", c.area());
    println!("Circumference: {:.2}", c.circumference());
}
```

## コレクション型

### Vector（可変長配列）

```rust
fn main() {
    let mut vec: Vec<i32> = Vec::new();
    vec.push(10);
    vec.push(20);
    vec.push(30);
    
    // イテレーション
    for item in &vec {
        println!("Item: {}", item);
    }
    
    // 要素の変更
    let first = &mut vec[0];
    *first = 100;
    
    // collectとイテレータ
    let doubled: Vec<i32> = vec.iter().map(|x| x * 2).collect();
    println!("Doubled: {:?}", doubled);
}
```

### HashMap（連想配列）

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert("Alice".to_string(), 92);
    scores.insert("Bob".to_string(), 85);
    
    // 値の取得（Optionで返る）
    let alice_score = scores.get("Alice");
    match alice_score {
        Some(score) => println!("Alice: {}", score),
        None => println!("Not found"),
    }
    
    // 存在チェックと削除
    if scores.contains_key("Bob") {
        scores.remove("Bob");
    }
    
    // 更新（既存値のデフォルト値設定）
    scores.entry("Charlie".to_string()).or_insert(0);
}
```
