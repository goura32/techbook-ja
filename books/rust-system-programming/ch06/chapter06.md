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
