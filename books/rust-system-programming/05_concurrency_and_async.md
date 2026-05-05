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


### チャネル（Channel）によるデータ共有

複数スレッド間でデータをやり取りする場合、チャネル（channel）が最も安全な方法です。Rustのチャネルはミューテックスよりも安全性が高く、デッドロックが起きません。

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    // 複数のスレッドがデータを送信
    for i in 0..5 {
        let tx = tx.clone();
        thread::spawn(move || {
            tx.send(i * 2).unwrap();
            println!("スレッド{}を送信", i);
        });
    }

    // 受信側で集約
    drop(tx);  // 送信側をドロップ（複製）
    let mut results = Vec::new();
    for val in rx {
        results.push(val);
        println!("受信: {}", val);
    }

    println!("合計値: {}", results.iter().sum::<i32>());
}
```

### async/awaitのパターン

Tokioランナーでasync/awaitを使った非同期処理を実現します。

```rust
use tokio::time::{sleep, Duration};
use tokio::sync::Mutex;
use std::collections::HashMap;

#[tokio::main]
async fn main() {
    // 並列の非同期タスク
    let handle1 = tokio::spawn(async {
        sleep(Duration::from_millis(100)).await;
        "Task 1"
    });

    let handle2 = tokio::spawn(async {
        sleep(Duration::from_millis(200)).await;
        "Task 2"
    });

    // タスクの完了を待つ
    let result1 = handle1.await.unwrap();
    let result2 = handle2.await.unwrap();
    println!("結果: {}, {}", result1, result2);
}
```

### スレッドセーフな状態の共有

RustのMutexはスレッドセーフな状態の共有に使用されます。

```rust
use std::sync::Arc;
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("合計: {}", *counter.lock().unwrap());
}
```

## FutureとExecutorの詳細

### Futureの仕組み

RustのFutureは非同期操作を抽象化する仕組みで、Pollとして状態を返します。

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// 簡易的なDelayed futureの実装
struct DelayedFuture {
    delayed_by: std::time::Duration,
    start: Option<std::time::Instant>,
}

impl DelayedFuture {
    fn new(delayed_by: std::time::Duration) -> Self {
        DelayedFuture {
            delayed_by,
            start: None,
        }
    }
}

impl Future for DelayedFuture {
    type Output = ();
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        let this = self.get_mut();
        
        if this.start.is_none() {
            this.start = Some(std::time::Instant::now());
        }
        
        let elapsed = this.start.unwrap().elapsed();
        if elapsed >= this.delayed_by {
            Poll::Ready(())
        } else {
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

async fn wait_delayed(duration: u64) {
    DelayedFuture::new(std::time::Duration::from_millis(duration)).await;
    println!("Done waiting {}ms", duration);
}

#[tokio::main]
async fn main() {
    tokio::join!(
        wait_delayed(1000),
        wait_delayed(500),
        wait_delayed(1500),
    );
    println!("All tasks completed");
}
```

### チャンネルによる並行通信

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(10);
    
    // 生産者タスク
    let producer = tokio::spawn(async move {
        for i in 0..5 {
            tx.send(i).await.unwrap();
            println!("Sent: {}", i);
        }
        drop(tx); // 送信側を閉じる
    });
    
    // 消費者タスク
    let consumer = tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            println!("Received: {}", msg);
        }
    });
    
    producer.await.unwrap();
    consumer.await.unwrap();
}
```

## 非同期エラーハンドリング

非同期コードではエラーハンドリングが特に重要です。

```rust
use tokio::time::{timeout, Duration};

async fn fetch_data(id: u64) -> Result<String, tokio::io::Error> {
    let id_str = id.to_string();
    let result = timeout(
        Duration::from_secs(5),
        tokio::fs::read_to_string(format!("/tmp/data/{}.txt", id_str))
    ).await;
    
    match result {
        Ok(Ok(content)) => Ok(content),
        Ok(Err(e)) => Err(e),
        Err(_) => Err(tokio::io::Error::new(
            tokio::io::ErrorKind::TimedOut,
            "Fetch timeout"
        )),
    }
}

#[tokio::main]
async fn main() {
    match fetch_data(1).await {
        Ok(data) => println!("Got data: {}", data),
        Err(e) => eprintln!("Error: {}", e),
    }
}
```

## 並行処理の実践パターン

### 並列処理スループットの最適化

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    // 並列タスクの管理
    let mut handles = vec![];
    
    for i in 0..10 {
        let handle = task::spawn(async move {
            println!("Task {} started", i);
            // 何かの計算
            tokio::time::sleep(std::time::Duration::from_millis(100)).await;
            println!("Task {} done", i);
            i * 2
        });
        handles.push(handle);
    }
    
    // 全タスクの完了を待つ
    let results: Vec<i32> = futures::future::join_all(handles)
        .await
        .into_iter()
        .filter_map(|r| r.ok())
        .collect();
    
    println!("All results: {:?}", results);
}
```
