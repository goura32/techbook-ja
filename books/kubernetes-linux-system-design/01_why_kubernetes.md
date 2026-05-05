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
### コンテナ時代の到来

インフラストラクチャの進化を追うことは、Kubernetesの必要性を理解するのに役立ちます。

**2006年** - Amazon Web Services (AWS) のEC2/ECSサービスを開始。仮想化からクラウドへ移行。

**2013年** - Dockerがオープンソース化。コンテナ技術が実用的なデプロイ単位として確立。

**2014年** - GoogleがBorgをOSSのKubernetesとして公開。10年以上の運用実績を持つ内部スケジューラの知見を集約。

**2015年** - CNCF（Cloud Native Computing Foundation）が設立。Kubernetesが最初の傘下プロジェクトに。

**2017年** - GKE/EKS/ACKがクラウド上でK8sをマネージドサービスとして提供開始。

**2023年以降** - AI/MLワークロード向けにK8sが標準基盤として定着。NVIDIA GPU、Wasmランタイム、eBPF連携が加速。

## クラウドネイティブの概念

### Twelve-Factor App

Kubernetesを適切に利用するには、クラウドネイティブな設計原則を理解する必要があります。

| Factor | 内容 | K8sでの対応 |
|--|--|--|
| 1. ベースコード | Gitでの一元管理 | git操作とCI/CDパイプライン |
| 2. 依存関係 | 明示的に宣言 | Container imageでの管理 |
| 3. 構成 | 環境変数で管理 | ConfigMapとSecret |
| 4. バックエンドサービス | アトミックなアタッチ | ServiceとIngress |
| 5. ビルド/リリース/実行 | 厳密に分離 | CI/CD + kubectl rollout |
| 6. プロセス | ステートレスな実行 | DeploymentとReplicaSet |

### DevOpsからプラットフォームエンジニアリングへ

DevOpsの次の進化として、プラットフォームエンジニアリングが注目されています。

| 比較項目 | DevOps | プラットフォームエンジニアリング |
|--|--|--|
| 焦点 | プロセスと自動化 | Internal Developer Portal |
| チーム | グロスに分割 | Internal Platformチーム |
| ツール | 分散 | 統合ポータル |
| 成果物 | パイプライン | Internal Developer Platform |

## K8sの代替技術との比較

| 技術 | 長所 | 短所 | 使用ケース |
|--|--|--|
| Docker Swarm | シンプル | エコシステム小 | 小規模デプロイ |
| Nomad | 柔軟なスケジューラ | 制約が薄い | 非K8sワークロード |
| ECS | AWS統合 | AWS固有 | AWSオンリー |
| OpenShift | 完全なパッケージ | やや重い | エンタープライズ |
| K8s | 最大のエコシステム | 複雑 | 汎用 |

## K8sのアーキテクチャ概要

Kubernetesはマスターノードとワーカーノードで構成されます。

### マスターノードの役割

| コンポーネント | 説明 |
|--|--|
| kube-apiserver | REST APIでK8sの全操作を統一 |
| etcd | クラスタの状態を保存する一貫したストア |
| kube-scheduler | 新規Podのノード割り当てを決定 |
| controller-manager | デーモンコントローラを管理 |
| cloud-controller-manager | クラウドプロバイダー固有のロジック |

### マスターノードの自己管理方法

マスターノードの各コンポーネントは以下のように動作しています。

1. APIサーバーからのリクエストがetcdに反映される
2. スケジューラがetcdのPending状態のPodを確認し、リソースが豊富なノードを割り当て
3. ノード上のkubeletがPodを起動
4. kube-proxyがServiceのポートフォワーディングを処理

### アーキテクチャの詳細図解

Kubernetesのアーキテクチャを図解してみましょう。

```
                    etcd（状態ストア）
                   /       \
              kube-apiserver
              /     |    \
    kube-scheduler  controller-manager  cloud-controller-manager
           |               |                    |
    +------v------+  +------v------+   +--------v-------+
    |  ワーカーNode 1 |  ワーカーNode 2 |   |  ワーカーNode 3  |
    |              |  |              |   |                |
    |  kubelet +   |  |  kubelet +   |   |  kubelet +     |
    |  kube-proxy  |  |  kube-proxy  |   |  kube-proxy    |
    +--------------+  +--------------+   +----------------+
```

### マスターノードの自己管理方法

1. APIサーバーからのリクエストがetcdに反映される
2. スケジューラがetcdのPending状態のPodを確認し、リソースが豊富なノードを割り当て
3. ノード上のkubeletがPodを起動
4. kube-proxyがServiceのポートフォワーディングを処理

### コンテナ時代のインフラストラクチャ進化

**2006年**: Amazon EC2の開始。仮想マシンベースのクラウド時代。

**2013年**: Dockerのオープン ソース化。コンテナがデファクトスタンダードに。

**2014年**: GoogleがBorgをKubernetesとして公開。10年以上の運用知見をOSS化。

**2017年**: AWS EKSの開始。クラウドマネージドK8sが本格化。

**2023年以降**: AI/ML基盤としてK8sが再評価。GPU/TPU連携、Wasmランタイムの統合。

## まとめ

本章をまとめます。

- Kubernetesはクラウドネイティブの標準基盤
- コンテナ技術の進化が土台
- マスターとワーカーの二層アーキテクチャ
- Docker SwarmやNomadなどの代替も存在
## プラットフォームエンジニアリングの重要性

### プラットフォームとは

プラットフォームエンジニアリングは、内部開発者体験（IDP）を向上させるための新しいDevOpsの進化的段階です。

### プラットフォームの3つの層

1. **Infrastructure Layer**: クラスタ、ネットワーク、ストレージ
2. **Platform Layer**: Service Mesh、CI/CD、GitOps
3. **Developer Experience Layer**: Internal Developer Portal、Self-Service

### プラットフォームチームの役割

従来のDevOpsチームとプラットフォームエンジニアリングチームの比較：

| 課題 | DevOpsチーム | プラットフォームチーム |
|--|--|--|
| デプロイ頻度 | 週1〜2回 | 数回/日 |
| メトリー・オブ・デプロイ | 数分 | 数秒 |
| エンジナーの作業負荷 | 多くの手作業 | ほぼ自動 |
| チームの成果物 | ツールの組み合わせ | 内部プラットフォーム |

## K8sの将来展望

### 短期展望（1年以内）

- **Wasmランタイムの標準化**: Wasmer、WasmEdgeなどのWebAssemblyランタイムがK8s上で実行可能に
- **Edge Computing**: K8sのミニマル版K3s/K0sがエッジで普及
- **AI統合**: KServ、Kubeflowの統合が深化

### 長期展望（3〜5年）

| トレンド | 説明 | 影響 |
|--|--|--|
| AIネイティブK8s | AIワークロードがK8sのファーストクラス | AI基盤としてK8sが必須に |
| 自律的な運用 | AIOpsによる自動スケーリング、自動障害復旧 | 運用負荷の大幅低下 |
| 量子コンピューティング | QPUのK8s統合 | 量子クラウド時代到来 |
| 環境対応 | 炭素意識オーケストレーション | 環境配慮型デプロイ |