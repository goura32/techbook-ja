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
