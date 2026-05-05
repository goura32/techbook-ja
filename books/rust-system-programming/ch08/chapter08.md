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
