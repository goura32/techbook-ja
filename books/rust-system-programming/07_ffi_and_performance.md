# 第7章 FFI、C連携、パフォーマンス最適化

## はじめに

本章では、Rustの外部ライブラリとの連携（FFI）と、パフォーマンス最適化の技法を学びます。

# 第7章 FFIとC連携

## C言語との連携

### FFI（Foreign Function Interface）の基本

```rust
extern "C" {
    // strlenの外部宣言
    fn strlen(s: *const i8) -> usize;
    
    // printfの外部宣言
    fn printf(fmt: *const i8, ...);
}

fn main() {
    let msg = c"Hello from Rust!";
    unsafe {
        println!("Hello, RustからFFI!", strlen(msg) as i8)) != 0);
    }
}

pub extern "C" fn rust_function(x: i32, y: i32) -> i32 {
    x * y
}

fn main() {
    // Rust関数を共有ライブラリとしてリンク
    // Rust関数を呼び出す場合は安全
}
```

### 共有ライブラリへのExport

```toml
# Cargo.toml
[lib]
name = "rust_lib"
crate-type = ["cdylib"]
```

### データ型の変換

```c
// C側 (main.h)
#pragma once
typedef struct {
    int x;
    int y;
} Point;
```

```rust
// Rust側
use std::os::raw::c_int;

#[repr(C)]
struct Point {
    x: c_int,
    y: c_int,
}

extern "C" {
    fn rust_function_create() -> *mut Point;
    fn rust_function_destroy(p: *mut Point);
    fn rust_point_set(p: *mut Point, x: c_int, y: c_int);
    fn rust_point_get(p: *mut Point) -> *const Point;
}
```

## まとめ

- RustはC言語と相互に呼び出し可能
- extern "C"でFFI
- 共有ライブラリとして公開可能
- unsafeで安全に管理

次章では、パフォーマンスの最適化について学びます。


## パフォーマンス最適化

Rustはゼロコスト抽象化を目標としており、ほとんどの場合でオーバーヘッドゼロで高速なコードが生成されます。しかし、さらなる最適化のために以下の技法があります。

# 第8章 パフォーマンス最適化

## ベンチマーク

```bash
# ベンチマーク
cargo test --bench my_benchmark
```

## クロージャとイテレータの最適化

```rust
fn main() {
    // ベクトル
    let data: Vec<i32> = (1..=100).collect();
    
    // イテレータをパイプライン
    // let sum: i32 = data.iter()  .filter(|x| *x % 2 == 0)
        .map(|x| x * x)
        .sum();
}

// sum()が最適化されて、中間のベクトルを生成しない

fn main() {
    let data: Vec<String> = vec![
    let len = data.iter()
        .map(|s| s.len())
        .copied()
        .collect::<Vec<_>>();
    // 中間のベクトル
    println!("{:?}", len) // [5, 6, 6, 9]
}
```

## クロージャの最適化

```rust
fn main() {
    let multiplier = 2.0;
    
    // クロージャ
    let square = |x: i32| -> i32 {
        x * x * (multiplier as i32)
    };
    
    // クロージャの型はinferred
    let result = square(5);
    println!("{result}"); // 50
}
```

## まとめ

- ベクトルのイテレータは自動最適化
- クロージャの型は推論

### まとめ

本章で学んだこと：

- i32とf64の両方を処理できる型パラメータ
- トレイト境界による制約


## まとめ

本章で学んだこと：

- FFIでC/C++の既存ライブラリを活用できる
- 共有ライブラリとしてRust関数を公開可能
- Rustの抽象化はゼロコスト
- イテレータの遅延評価で最適化
- ベンチマークでボトルネックを特定可能
- クロージャの型推論はコンパイル時に解決
- num_gpuでGPUのリソース活用も重要
