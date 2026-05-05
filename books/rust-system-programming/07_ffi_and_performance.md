# 第7章 FFIとC連携、パフォーマンス最適化

## はじめに

本章では、Rustの強力な機能である外部関数インターフェース（FFI）と、パフォーマンス最適化の技法を学びます。FFIを使えば、既存のC/C++ライブラリをRustから直接呼び出せ、パフォーマンス最適化の知識があれば、本番品質の高速コードが書けるようになります。

本章の目標：
- FFIの基本と実際の使い方
- C言語との相互運用
- 共有ライブラリとしての公開方法
- Rustのパフォーマンス最適化技法
- ベンチマークの実施方法

## C言語との連携（FFI）

### FFI（Foreign Function Interface）とは

FFIは、Rustのプログラムが外部のコード（通常はC言語）を呼び出す仕組みです。これにより、既存のCライブラリをRustから安全に利用できます。

### C関数をRustから呼び出す

最も基本的なFFI使用例です。Cの標準ライブラリ函数をRustから呼び出します。

```rust
use std::ffi::CString;
use std::os::raw::c_char;

extern "C" {
    fn strlen(s: *const c_char) -> usize;
    fn strcmp(one: *const c_char, another: *const c_char) -> i32;
}

fn main() {
    // CStringに変換してからFFI呼び出し
    let string = CString::new("Hello, Rust FFI!").expect("CString new failed");
    unsafe {
        let len = strlen(string.as_ptr());
        println!("文字列の長さ: {}", len);
    }
}
```

**ポイント：**
- `extern "C"` ブロックで外部函数を宣言
- Cの文字列は`CString`を使って安全に変換
- `unsafe` ブロック内でFFI呼び出しを実行

### Rust関数をCから呼び出す

Rustの函数をCから呼び出すことも可能です。

```rust
// lib.rs
#[no_mangle]
pub extern "C" fn add(x: i32, y: i32) -> i32 {
    x + y
}

#[no_mangle]
pub extern "C" fn rust_init() -> *mut std::os::raw::c_void {
    let vec = Box::new(vec![1, 2, 3]);
    Box::into_raw(vec) as *mut std::os::raw::c_void
}

#[no_mangle]
pub extern "C" fn rust_free(ptr: *mut std::os::raw::c_void) {
    unsafe {
        let _ = Box::from_raw(ptr as *mut Vec<i32>);
    }
}

fn main() {
    println!("add(3, 4) = {}", unsafe { add(3, 4) });
}
```

共有ライブラリとしてビルドするには、Cargo.tomlに以下を記載します。

```toml
[lib]
name = "rust_lib"
crate-type = ["cdylib", "staticlib"]
```

ビルド命令：

```bash
cargo build --release
# linuxの場合: librust_lib.so が生成される
# macOSの場合: librust_lib.dylib が生成される
# Windowsの場合: rust_lib.dll が生成される
```

### データ型の変換

RustとCの間で構造体を共有する場合、データレイアウトを適切に合わせる必要があります。

**C側 (types.h)：**

```c
#pragma once

typedef struct {
    double x;
    double y;
    int active;
} Point;

typedef struct {
    Point center;
    double radius;
} Circle;
```

**Rust側：**

```rust
use std::os::raw::c_double;
use std::os::raw::c_int;

#[repr(C)]
#[derive(Debug, Clone, Copy)]
struct Point {
    x: c_double,
    y: c_double,
    active: c_int,
}

#[repr(C)]
#[derive(Debug, Clone, Copy)]
struct Circle {
    center: Point,
    radius: c_double,
}

// 安全なラッパー
impl Point {
    fn new(x: f64, y: f64) -> Self {
        Point { x, y, active: 0 }
    }

    fn distance(&self, other: &Point) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }
}

impl Circle {
    fn new(center: Point, radius: f64) -> Self {
        Circle { center, radius }
    }

    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

fn main() {
    let center = Point::new(3.0, 4.0);
    let point = Point::new(7.0, 7.0);
    println!("距離: {:.2}", center.distance(&point));

    let circle = Circle::new(center, 5.0);
    println!("円の面積: {:.2}", circle.area());
}
```

### FFIを使用する際の安全ガイドライン

1. **可能な限り`unsafe`を最小化** — FI呼び出し以外のコードは安全に
2. **ポインタの寿命を確認** — 無効なポインタは未定義動作
3. **文字列の変換は`CString`と`CStr`を使う** — メモリリーク防止
4. **エラー処理を明示** — Rustの`Result`でラップして安全に
5. **テストで検証** — インテgrーションテストは必須

## Rust言語のパフォーマンス最適化

### ゼロコスト抽象化

Rustは「ゼロコスト抽象化」を基本理念としています。つまり、高水準の抽象化を使っても実行時にオーバーヘッドがかからないということです。

```rust
fn main() {
    let data: Vec<i32> = (1..=1_000_000).collect();

    // イテレータでパイプライン処理（最適化されたコードが生成される）
    let sum_even_squares: i64 = data.iter()
        .filter(|x| *x % 2 == 0)
        .map(|x| (x * x) as i64)
        .sum();

    println!("偶数の2乗の和: {}", sum_even_squares);
}
```

このコードでは、中間ベクタを作らずにストリーミング処理が最適化されます。

### インライン化の活用

関数をインライン化することで、函数呼び出しのオフヘッドを消せます。

```rust
#[inline]
fn square(x: i32) -> i32 {
    x * x
}

#[inline(always)]
fn abs_diff(a: i32, b: i32) -> i32 {
    if a > b { a - b } else { b - a }
}

fn main() {
    let squares: Vec<i32> = (0..100).map(square).collect();
    println!("{}", abs_diff(10, 3));
}
```

### クロージャの活用と最適化

クロージャはRustで高速に関数を生成する強力な手段です。

```rust
fn main() {
    let multiplier = 2.5;

    // 計算重たい函数のクロージャ
    let heavy_compute = |x: f64| -> f64 {
        (0..10000).map(|i| (x * i as f64).sin()).sum::<f64>()
    };

    let compute_sine_table = |base: f64| -> Vec<f64> {
        (0..360).map(|deg| heavy_compute(base + deg as f64)).collect()
    };

    println!("値の数が: {}", heavy_compute(1.0));
}
```

### ベクトルと配列の最適化

ベクトル操作には適切なメモリ割り当てが重要です。

```rust
fn main() {
    let n = 1_000_000;
    let mut data = Vec::with_capacity(n);
    data.extend(0..n);

    let array = [0i32; 100];  // スタック上に配置（高速）
    let vec = vec![0i32; 100];  // ヒープ上に配置

    println!("配列の要素数: {}", array.len());
    println!("ベクタの要素数: {}", vec.len());
}
```

### ベンチマークの実行

ベンチマークを実行してボトルネックを特定します。

```bash
cargo test
cargo bench
cargo build --release
./target/release/myapp
perf record -g ./target/release/myapp
perf report
```

### Rustのパフォーマンス最適化チェックリスト

1. **アロケーションを最小化** — `Vec::with_capacity`を使おう
2. **文字列の結合** — `format!`より`&str`スライス
3. **ループの最適化** — `iter`は`for`ループと同速度
4. **関数呼び出し** — `#[inline]`で最適化
5. **メモリアクセス** — リージョン指向でキャッシュに優しく
6. **並行処理** — `rayon`で並列化を簡単に

## まとめ

本章で学んだこと：

- FFIでC/C++の既存ライブラリを活用できる
- `extern "C"`で外部函数を宣言
- 共有ライブラリとしてRust関数を公開可能
- Rustの抽象化はゼロコストで動作
- イテレータの遅延評価で最適化
- ベンチマークでボトルネックを特定可能
- `#[inline]`や`#[inline(always)]`で函数展開
- 性能最適化のチェックリストで安全に最適化

次章では、Rustのエラーハンドリングについて学びます。
