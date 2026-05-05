# 第5章 並行性と非同期

## スレッドとメッセージパッシング

Rustの並行性は、スレッドとメッセージパッシングの2つのアプローチを提供します。

### スレッドの基本的な使い方

```rust
use std::thread;
use std::sync::mpsc;
use std::sync::{Arc, Mutex};

fn main() {
    // basic
    let handles: Vec<_> = (0..10)
        .map(|i| {
            thread::spawn(move || {
                println!("スレッド {} 開始", i);
                i * 2
            })
        })
        .collect();

    for handle in handles {
        println!("結果: {}", handle.join().expect("スレッドの失敗"));
    }
}
```

### チャネルによるメッセージパッシング

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    // 複数のスレッドで同じtxを送信
    let mut threads = vec![];
    for i in 0..3 {
        let tx = tx.clone();
        threads.push(thread::spawn(move || {
            tx.send(i * 10).unwrap();
        }));
    }

    // rxで受信
    drop(tx);  // senderをdropしないとrecvがブロックする
    rx.try_iter().for_each(|msg| {
        println!("受信: {}", msg);
    });
}
```

## async/await

### Tokioによる非同期プログラミング

```bash
# Cargo.tomlに追加
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### asyncの基本

```rust
use std::future::Future;
use std::pin::Pin;

#[tokio::main]
async fn main() {
    println!("非同期プログラム開始");
    
    let f1 = async { 42 };
    println!("f1 = {}", f1.await);  // 42
    
    let f2 = async { "Hello, Rust!" };
    println!("f2 = {}", f2.await);  // Hello, Rust!
}
```

### チャネルの非同期版

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = mpsc::channel(32);

    let mut tasks = vec![];
    for i in 0..3 {
        let tx = tx.clone();
        tasks.push(tokio::spawn(async move {
            tx.send(i * 10).await.unwrap();
        }));
    }

    drop(tx);  // senderをdrop

    while let Some(msg) = rx.recv().await {
        println!("受信: {}", msg);
    }
}
```

## まとめ

- thread::spawnでスレッド
- mpsc::channelでスレッド間通信
- tokio::spawnで非同期タスク
- async/awaitで非同期関数
