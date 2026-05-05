# 第1章 なぜ今、Kubernetesなのか

## なぜ今、Kubernetesなのか

本章では、クラウドネイティブ基盤としてのKubernetesの重要性を理解します。

| トレンド | 説明 |
|---|---|
| AI/MLワークロード | データパイプラインから「主要ユースケース」へ。AIワークロードがK8sのファーストクラス市民に |
| eBPF統合 | 高速ネットワークと可視化。Cilium 1.18+ |
| GitOps標準化 | Argo CDが「多数派採用」。Argo CDのApp of Appsパターンがデファクトに |
| FinOps | コスト管理の標準化。OpenCostがK8s上の標準コスト可視化ツールに |
| プラットフォームEngineering | DevOpsの次の進化段階として確立 |

## KubeCon Europeで注目されたポイント

KubeCon + CloudNativeCon Europe で注目されたのは：

1. **AI/MLの統合**: NVIDIAのGPUサポート、KServeの進化
2. **セキュリティ**: 強力なゼロトラスト、mTLS自動管理
3. **WebAssembly**: Wasmランタイム（WebAssembly）のK8sネイティブサポート
4. **FinOps**: AWS/GCP/Azureの統合、コスト配分の自動化
5. **eBPF**: ネットワークの自動生成、トラフィック可視化

### CNCFの成熟ロードマップ

CNCFの成熟ロードマップは現在以下のように定義：

```
Phase 1: 基盤（K8s未導入）
Phase 2: 可視化（オブザーバビリティ未整備）
Phase 3: プラットフォーム化（開発者体験の向上）
Phase 4: 最適化（コスト・パフォーマンス）
Phase 5: データの統合（データ管理・統合）
Phase 6: AI/ML（AIワークロードの標準化）
```

## KubernetesとLinuxの深い関係

Kubernetesは**Linuxの上で動いています**。理解すべきカーネルの仕組み：

### コンテナのLinux実装

```
+----+----+-----+-----+-----|-----|-----|-----+
| Linux Kernel: cgroups, namespaces, |
| seccomp, Capabilities, |
| network namespace          |
+----------+---------------+----------+----+
| Container Runtime (containerd/Runc)|
|     +-----+     +-----+     +-----+  |
|     | Pod 1 |     | Pod 2 |     | Pod 3 |  |
|     +-----+     +-----+     +-----+  |
+--------+---------------------------+----+
    API Server    Controller Manager
                    Scheduler
                    kubelet
+------------------|---------+
Kubelet    |         Scheduler
  kubectl  |         API Server
  Docker   |
  Docker   |
|   Docker   |
+------------|---------|
Linux Node (Server)
```

### cgroups（control groups）の役割

cgroupsは、以下のようなリソースを制御します：

| リソース | 制御内容 |
|---|---|
| CPU | CPU使用率制限 |
| Memory | メモリ使用量制限 |
| Disk I/O | ディスク入出力の制限 |
| Network | ネットワーク帯域制限 |

### namespaces（名前空間）の役割

namespacesは、以下のような独立したビューを提供：

| namespaceタイプ | 隔離内容 |
|---|---|
| pid | プロセスID |
| net | ネットワークスタック |
| mount | ファイルシステム |
| uts | ホスト名とドメイン |
| ipc | システムV IPC |
| user | ユーザーとグループ |

## Linuxの基礎知識（再確認）

### ファイルシステムヒエラルキー

```
/
├── bin      # 主要コマンド（bin）
├── etc      # コンフィグファイル
├── home     # ユーザーホーム
├── proc     # プロセス情報（proc）
├── sys      # カーネルパラメータ（sys）
├── var      # 可変データ
├── run      # ランタイムデータ
└── dev      | デバイスファイル
```

### systemdの基本

```bash
# サービスの管理
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl status nginx
```

### ネットワーク設定

```bash
# ネットワークインターフェースの確認
ip addr show

# ルーティングテーブル
ip route show

# DNSの参照
nslookup example.com
cat /etc/resolv.conf

# ポートの確認
ss -tlnp
netstat -tlnp
```

## まとめ

本章で学んだこと：

- KubernetesはAI/MLワークロード、GitOps、eBPFの統合
- Linuxのcgroupsとnamespacesがコンテナ技術の基盤
- CNCFロードマップで段階的にプラットフォームを成熟

次章では、Linuxのコンテナの仕組みを深く学びます。
