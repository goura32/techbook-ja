# 第6章 パターンとジェネリクス

## Enumとパターンマッチング

```rust
enum Color {
    Rgb(u8, u8, u8),
    Hsv { h: f64, s: f64, v: f64 },
}

fn main() {
    let c = Color::Rgb(255, 0, 0);
    
    match c {
        // タプルバリアントのパターンマッチング
        Color::Rgb(r, g, b) => {
            println!("RGB({r}, {g}, {b})");
        }
        // 構造体バリアント
        Color::Hsv { h, s, v } => {
            println!("HSV({h}, {s}, {v})");
        }
    }
}
```

## Trait（トレイト）

```rust
// トレイトの定義
trait Drawable {
    fn draw(&self);
    fn area(&self) -> f64;
    fn name(&self) -> &str {
        "Drawable"
    }
}

struct Circle {
    radius: f64,
    color: String,
    name: String,
}

trait CircleTrait {
    fn draw(&self) {
        println!("円を描きます (半径: {})", self.radius);
    }
    fn area(&self) -> f64 {
        3.141592 * self.radius * self.radius
    }
    fn name(&self) -> &str {
        "Circle"
    }
}

impl CircleTrait for Circle {
    fn draw(&self) {
        println!("円を描きます (半径: {})", self.radius);
    }
    fn area(&self) -> f64 {
        3.141592 * self.radius * self.radius * self.radius
    }
    fn name(&self) -> &str {
        "Circle"
    }
}

fn draw_all(drawables: &[&dyn Drawable]) {
    for d in drawables {
        d.draw();
        println!("面積: {:.2}", d.area());
        println!("名前: {}", d.name());
        println!();
    }
}

fn main() {
    let circle = Circle {
        radius: 50.0,
        color: "赤".to_string(),
        name: "円1".to_string(),
    };
    
    // メソッド呼び出し
    println!("面積: {:.2}", circle.area());
}
```

## ジェネリクス

```rust
// ジェネリック関数
fn max<T: PartialOrd>(a: T, b: T) -> T {
    if a > b {
        a
    } else {
        b
    }
}

println!("{}", max(3, 5));      println!("{}", max(3.14, 2.71));  // 3.14

// ジェネリック型
struct Point<T> {
    x: T,
    y: T,
}

// メソッド
impl<T> Point<T> {
    fn distance(&self) -> f64 {
        let dx = (self.x as f64) - (self.x as f64);
        let dx = (self.y as f64) - (self.x as f64);
        (dx * dx + dy * dy).sqrt()
    }

impl<T: std::fmt::Display> Point<T> {
    fn print(&self) {
        println!("x: {}, y: {}", self.x, self.y);
    }

fn main() {
    let point32 = Point { x: 3, y: 4 };
    let pointf64 = Point { x: 3.14, y: 2.71 };
    point32.print();  // x: 3, y: 4
    pointf64.print(); // x: 3.14, y: 2.71
}
```

### trait Boundで抽象化

```rust
// シングルトンパターン
fn main() {
    let max_i32 = max(3, 5);        // i32
    let max_f64 = max(3.14, 2.71);  // f64
}
```

## まとめ

- enumとパターンマッチングで多様なデータを表現
- traitで振る舞いを抽象化
- ジネリックスで型引数を取る関数・型を作成

## 高度なパターンマッチング

### if letとwhile let

```rust
fn main() {
    let opt = Some(42);
    
    // if let：値がSomeの場合は処理
    if let Some(value) = opt {
        println!("Got value: {}", value);
    }
    
    // while let：Someがなくなるまでループ
    let mut values = vec![Some(1), Some(2), Some(3), None];
    while let Some(v) = values.pop() {
        println!("Value: {}", v);
    }
}
```

### ビットパターンマッチング

```rust
fn main() {
    let byte = 0b1101_0010u8;
    
    match byte {
        // ビットパターンにマッチ
        b if b & 0b0000_0001 != 0 => println!("最下位ビットが立っている"),
        _ => println!("最下位ビットが立っていない"),
        
        // マスクによるマッチ
        b if (b & 0b1111_0000) == 0b1101_0000 => println!("上位4ビットが1101"),
        _ => println!("それ以外"),
    }
}
```

## ジェネリクス

### ジェネリック関数

```rust
fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in &list[1..] {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let numbers = vec![34, 50, 25, 70, 19];
    let chars = vec!['z', 'a', 'm', 'x', 'b'];
    
    println!("Largest number: {}", largest(&numbers));
    println!("Largest char: {}", largest(&chars));
}
```

### ジェネリック構造体

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn mix<V, W>(self, other: Point<V, W>) -> Point<T, V> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 3.14 };
    let p2 = Point { x: 10, y: 2.71 };
    let p3 = p1.mix(p2);
    println!("p3.x = {}, p3.y = {}", p3.x, p3.y);
}
```

## トレイト境界と制約

トレイト境界によりジェネリック型の動作を制限できます。

```rust
// Displayトレイトを実装した型のみ受け付ける
fn print_info<T: std::fmt::Display>(item: &T) {
    println!("Item: {}", item);
}

// CloneとPartialEqを実装した型のみ受け付ける
fn compare_and_clone<T: std::fmt::Display + Clone + PartialEq>(a: T, b: T) {
    let a_copy = a.clone();
    if a == b {
        println!("{} and {} are equal", a, b);
    }
    println!("Clone copy: {}", a_copy);
}

fn main() {
    print_info(&42);
    print_info(&"hello");
    
    compare_and_clone(String::from("hello"), String::from("hello"));
}
