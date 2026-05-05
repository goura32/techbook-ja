# 第2章 Linux基礎を再確認

## コンテナのLinux実装

本章では、コンテナがLinux上でどのように実装されているかを理解し、Kubernetesの設計を深く理解するための基盤を築きます。

### Linuxカーネルの重要な機能

#### cgroupsの仕組み

```bash
# cgroupsの構造を確認
ls /sys/fs/cgroup/

# 現在のプロセスの制限を確認
cat /sys/fs/cgroup/memory/current.memory.usage_in_bytes
cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us

# cgroup v2の構造
ls /sys/fs/cgroup/system.slice/
```

#### namespacesの仕組み

```bash
# プロセスのnamespace IDを確認
ls -l /proc/1/ns/

# 出力例：
# mount -> [4026531840]
# net   -> [4026531851]
# pid   -> [4026531836]
# uts   -> [4026531838]
```

### Dockerの仕組み

```bash
# Dockerの内部構造を確認
docker info

# イメージのファイルシステムを確認
docker inspect <container_id> --format='{{json .GraphDriver}}' | jq '.'
```

### コンテナのオーバーヘッド

```
+-----+-----------+------------+-------+------|
| 構成  | メモリ   | CPU使用率  | ストレ| 起動 |
+-----+-|---|--|-----+--------------+-------+-----+
| VM    | 512MB   | 5-10%     | 1-5GB | 数分 |
| Cont.   | <10MB   | 0-1%     | 10-500MB | 数秒 |
+-----+-|---|--|-----+--------------+-------+-----+
```

## Kubernetesのアーキテクチャ

### クラスタの構成

```
+------------------------------------------+
| Control Plane (マスター)                   |
|                                           |
| +-----+  +----+---+   +-----+    +-----+ |
| |API  |  |  Controller  | |   |Scheduler| | |
| +-----+  +----+-----+   +-----+    +-----+ |
|  +--|----+                                |
| +-----+                                  |
| |  etcd  | (Key-Value Store)              |
| +-----+                                  |
+------------------------------------------+
    |      |      |
    |      |      |
+----+  +----+  +----+
| Node 1 | | Node 2 | | Node 3 |
|        | |        | |        |
| API    | | API    | | API    |
| Server  | | Server | | Server |
| kubelet | | kubelet| | kubelet|
|    +----+  |    +--+  |    |
|    |       |    |     |    |
|  +-----+   |  +-----+ |  +-----+
|  |- Pod 1|  |  |- Pod 1| |  |- Pod 1|
|  +-----+   |  +-----+ |  +-----+
|  |  Pod 2 |  |  |  Pod 2 |
|  +-----+   +-----+ +-----+
+-----+
```

### コアイメージの解説

| コンポーネント | 役割 |
|---|-|
| API Server | クラスタのAPIエンドポート
| Controller Manager | 状態マージナー
| Scheduler | ワークロードのスケジューラ |
| etcd | 設定と状態のストレージ（クラスタのデータベース） |
| kubelet | ノードのエージェント |
| Container runtime | コンテナの実行（containerd、CRI-O） |
| CNI | コンテナネットワークインターフェース |
| CoreDNS | サービスディスカバリ |
| kube-proxy | サービスとネットワーク |

## etcdを深く理解する

etcdはK8sクラスタのブレースです。

```bash
# etcdctlの基本的な操作
etcdctl get /
etcdctl put /mykey "myvalue"
etcdctl get /mykey --print-value-only

# キーの階層構造
etcdctl get / --prefix --keys-only

etcdctl get /registry/ --prefix --keys-only
# 出力例：
# /registry/deployments/kube-system
# /registry/replicasets/kube-system
# /registry/pods/kube-system

# イベントの確認
etcdctl get /registry/events --prefix --print-value-only
```

###.etcdの設計

etcdはRaftプロトコルで分散合意を実現します：

```
Leader   | --- 1 --> | Follower 2
          | --- 2 --> | Follower 3
          | --- 3 --> | Follower 4

Follower 2 | <-- 4 --- | Follower 3
           | <-- 5 --- | Leader
           | <-- 6 --- | Follower 4
```

## Linuxの名前空間とcgroupの実践

### CNIプラグインの構成

CNI（Container Network Interface）は、Podのネットワークインターフェースを動的に作成・設定します。

```
Pod A          Pod B
eth0          eth0
veth pair    veth pair
|             |
v             v
CNI Bridge   CNI Bridge
|             |
v             v
Physical NIC |
```

### CNIプラグインの例（Calico）

```bash
# Calicoのセットアップ
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

# CNIの確認
kubectl get pods -n kube-system | grep calico
```

### Cilium（eBPFベースCNI）

eBPFによる高速ネットワークの例：

```bash
# Ciliumのインストール
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system

# 可視化
kubectl get hubble-dest -n kube-system
kubectl get pods -n kube-system | grep cilium
```

## まとめ

本章で学んだこと：

- cgroupsとnamespacesがコンテナ技術の基盤
- Linuxのシステム設計をKubernetesに理解する
- etcdがK8sのブレースに
- CNIとCilium/eBPFがK8sネットワークを

次章では、K8sの核心を学びます。
## ユーザーネームスペース

Kubernetesはユーザーネームスペース間でリソースを分離します。

```bash
# ネームスペースの確認
kubectl get namespaces
# OUTPUT: default Active, kube-system Active, kube-public Active, production Active

# ネームスペースを作成
kubectl create namespace production

# リソースにネームスペースを設定
kubectl create deployment nginx --image=nginx -n production
kubectl get pods -n production
```

## Kubernetesとコンテナランタイム

KubernetesはCRI(Container Runtime Interface)経由でランタイムを管理します。

| コンテナランタイム | 特徴 | K8sでの使用例 |
|--|--|--|
| containerd | CNCFプロジェクト。Dockerが使用 | クラウドマネージドK8s |
| CRI-O | Kubernetes専用。軽量 | OpenShift |
| Docker shim | 廃止予定 | 古いK8sバージョン |
| Kata Containers | システムレベルの分離 | マルチテナント環境 |
## Linuxの名前空間の詳細

### プロセスの分離仕組み

```bash
# プロセスごとの名前空間IDを確認
ls -l /proc/1/ns/
# mount -> [4026531840]
# net   -> [4026531851]
# pid   -> [4026531836]
# uts   -> [4026531838]
# ipc   -> [4026531839]
# user  -> [4026531837]
# cgroup -> [4026531841]

# 新しい名前空間でコマンドを実行
unshare --fork --pid --mount-proc bash
# このシェルは新しいPID名前空間で実行される
```

### 6つの名前空間

| 種類 | 分離対象 | 用途 |
|--|--|--|
| PID | プロセスID | 他のプロセスから隔離 |
| Network | ネットワークインターフェース |独立したネットワークスタック |
| Mount | ファイルシステム | コンテナのルートfilesystem |
| UTS | ホスト名 | コンテナ内と外で別々の名前 |
| IPC | 共有メモリ | コンテナ間でも分離 |
| User | ユーザー/グループ | 権限管理 |

## cgroups v2の仕組み

### リソース制限の設定

```bash
# cgroup v2の構造を確認
tree /sys/fs/cgroup/

# CPUリソース制限
echo 50000 > /sys/fs/cgroup/myapp/cpu.max
# 50000 = 50%%（100000が100%）

# メモリリソース制限
echo 1073741824 > /sys/fs/cgroup/myapp/memory.max
# 1073741824 = 1GB

# I/O帯域制限
echo "8:0 reads=1024 writes=512" > /sys/fs/cgroup/myapp/io.max
```

### リージュームの仕組み

cgroups v2のリージョン管理により、優先度の高いプロセスにリソースを割り当てることができます。

```bash
# キューイングの仕組み
echo 1 > /sys/fs/cgroup/myapp/sched.weight
# 100がデフォルト。高いほど優先度が上がり、CPUタイムが割り当てられる。

# メモリの優先的リリース
echo 1 > /sys/fs/cgroup/myapp/memory.pressure
# メモリ逼迫時に優先的にメモリエントリをリリース
```

## K8sとLXCの関連

### K8sがLinuxコンテナを管理する仕組み

KubernetesはLXC（Linux Container）を直接操作せず、CRI（Container Runtime Interface）を介してコンテナランタイムを呼び出します。

### CRI（Container Runtime Interface）の概要

```
    +---------------------+
    |  Kubernetes Engine  |
    |  (kubelet, kubelet) |
    +----------+----------+
               | CRI (gRPC)
    +----------v----------+
    |  Container Runtime  |
    |  (containerd, CRI-O)|
    +----------+----------+
               | OCI (containerd)
    +----------v----------+
    |  Container Runtime  |
    |  (runc, crun)       |
    +---------------------+
```

### K8sのPodとコンテナの関係

1 Pod = 1または複数のコンテナ（共享リソース）

```yaml
# multi-container Podの例
apiVersion: v1
kind: Pod
metadata:
  name: log-processor
spec:
  containers:
  - name: log-reader
    image: busybox
    command: ["sh", "-c", "while true; do tail -f /var/log/system.log; done"]
    volumeMounts:
    - name: log-volume
      mountPath: /var/log
  - name: log-processor
    image: fluentd
    volumeMounts:
    - name: log-volume
      mountPath: /var/log
  volumes:
  - name: log-volume
    emptyDir: {}
```