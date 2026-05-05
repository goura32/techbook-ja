# 第10章 実践プロジェクト：K8sプラットフォーム設計

## はじめに

本章では、これまでに学んだすべてを総動員して、本番環境のK8sプラットフォームを設計・構築します。

本章で計画するもの：
- 3ノード構成のクラスター設計
- ハイブリッドノードグループ（CPU/GPU）
- ネットワーク（Cilium）
- GitOps（Argo CD）
- モニタリング（Prometheus + Grafana）
- FinOps（OpenCost）
- AI/MLプラットフォーム（KServe + Kubeflow）

## プラットフォーム設計

### Phase 1: Base Platform

```bash
# 1. ノードの準備（3ノード構成）
# Control Plane x3
kubectl get nodes

# 2. ネットワークのインストール
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.16.3/install/kubernetes/manifests/cilium-install.yaml

# 3. GPU Device Plugin
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/deployments/static/nvidia-device-plugin.yml
```

### Phase 2: GitOps

```yaml
# ArgoCDのインストール
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myrepo/k8s-manifests.git
    path: base
    targetReleases: main
  destination:
    server: https://kubernetes.default.svc
EOF
```

### Phase 3: Monitoring

```bash
# Prometheusのインストール
helm install prometheus-stack grafana/prometheus-kube-prometheus \
  --namespace monitoring --set alertmanager.enabled=true
```

### Phase 4: FinOps

```yaml
# OpenCostのインストール
kubectl apply -f https://raw.githubusercontent.com/opencost/opencost/develop/manifests/kubecost.yaml
```

### Phase 5: AI/ML Platform

```yaml
# KServe + Kubeflowの統合
kubectl apply -k "github.com/kserve/kserve/config/overlays/stable?ref=v0.15.0"
```

## まとめ

本章で学んだこと：
- K8sプラットフォームの全体設計
- Node Group（CPU/GPU）の設計
- CiliumによるCilium
- Argo CDによるGitOps
- Prometheus/Grafanaによるモニタリング
- OpenCostによるFinOps
- KServe + KubeflowによるAI/ML

---

K8sは進化を続ける。最新のCNCFプロジェクトの動向をフォローしてください。
## Infrastructure as Code（IaC）

### TerraformでのK8sクラスター構築

```hcl
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.22"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

# クラスタのリソース定義
resource "kubernetes_namespace" "platform" {
  metadata {
    name = "platform"
  }
}

resource "helm_release" "argocd" {
  name       = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  namespace  = kubernetes_namespace.platform.metadata[0].name
  version    = "5.48.0"

  set {
    name  = "server.extraArgs"
    value = "--insecure"
  }
}
```

## プラットフォームの運用

### クラスタのメンテナンス手順

1. ノードのローリング更新
   ```bash
   # K8sバージョンのアップグレード
   kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
   # 該当ノードのK8sをアップグレード
   kubectl uncordon <node-name>
   ```

2. メンテナンスウィンドウの設定
   ```yaml
   apiVersion: node.k8s.io/v1
   kind: RuntimeClass
   metadata:
     name: maintenance
   handler: maintenance
   scheduling:
     nodeSelector:
       maintenance-window: "true"
   ```

3. バックアップの自動化
   ```bash
   # etcdのバックアップ（マスターノード上で）
   ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key
   ```

### 運用チェックリスト

| 項目 | 頻度 | 内容 |
|--|--|--|
| クラスタヘルスチェック | 毎日 | ノードの準備状況、Podのステータス |
| リソース使用状況確認 | 毎日 | CPU、メモリ、ストレージの利用率 |
| コストレポート | 毎週 | OpenCost/Grafanaダッシュボード |
| セキュリティパッチ | 毎月 | コンテナランタイム/OSの更新 |
| バックアップ検証 | 毎週 | etcdスナップからの復元テスト |
| パフォーマンスチューニング | 四半期 | リソースリクエストの再評価 |

## まとめ

本章で計画したプラットフォームの要点：

- コントロールプレーは3ノードで高可用性
- ホイットリストされたGPUノードでAI/MLワークロード
- Ciliumによる高速ネットワーク
- Argo CDによるGitOps
- Prometheus + Grafanaによる完全な可視性
- OpenCostによるFinOps
- KServe/KubeflowによるAIプラットフォーム
## プラットフォームの構成まとめ

### 全体のアーキテクチャ図

```
                        +------------------+
                        |   External API   |
                        |   / Monitoring   |
                        +--------+---------+
                                 |
                        +--------v---------+
                         |  Ingress/Cilium  |
                        +--------+---------+
                                 |
   +-----------------+----------+----------+------------------+
   |                 |                    |                   |
+--+--+          +--+--+              +--+--+             +--+--+
| App |          | App |              | App |             | App |
| A   |          | B   |              | C   |             | D   |
+--+--+          +--+--+              +--+--+             +--+--+
   |                 |                    |                   |
   +-----------------+--------+-----------+------------------+
                            |
                  +---------v---------+
                  |   Data Storage     |
                  |   (Persistent     |
                  |   Volumes)        |
                  +-------------------+
```

### 主要コンポーネントの構成

| コンポーネント | バージョン | 設定ファイル |
|--|--|--|
| Kubernetes | v1.31+ | kubeadm config |
| Cilium | v1.16+ | cilium-install.yaml |
| Argo CD | v2.12+ | argocd-install.yaml |
| Prometheus | v2.50+ | prometheus-operator.yaml |
| Grafana | v11.0+ | grafana-dashboard.json |
| OpenCost | v1.3+ | opencost-install.yaml |
| KServe | v0.15+ | kserve-install.yaml |
| Kubectl CLI | v1.31+ | kubectl-standalone |

## クラスタ運用のベストプラクティス

### ノード管理のベストプラクティス

1. **ノードタイプ別のラベル付け**
   ```bash
   kubectl label node worker-gpu-001 node-type=gpu --overwrite
   kubectl label node worker-cpu-001 node-type=cpu --overwrite
   ```

2. ** taintedノードの管理
   ```yaml
   # GPUワークロードのみを許容
   taints:
   - key: nvidia.com/gpu
     value: "present"
     effect: NoSchedule
   ```

3. **自動スケーリング**
   ```yaml
   apiVersion: addons.k8s.io/v1
   kind: ClusterAutoscaler
   spec:
     minNodes: 3
     maxNodes: 10
     targetUtilization: 0.7
   ```

## コスト管理の実践

### OpenCostによるコスト最適化

```bash
# コストレポの取得
curl http://opencost-opencost:9090/api/ve/cost?window=7d > cost-report.json

# 最コストのネームスペースを特定
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory

# 未使用リソースの特定
kubectl get pods --all-namespaces --sort-by=.spec.containers[0].resources.requests.cpu
```

### コスト最適化の具体案

| 項目 | 現在 | 目標 | 削減率 |
|--|--|--|--|
| CPUリクエスト | 500m | 250m | 50% |
| メモリリクエスト | 512Mi | 256Mi | 50% |
| GPU予約 | 40%% | 60%% | 使用率向上 |
| Storage | on-demand | 予約 | 30% |

## まとめ

本章で学んだプラットフォーム設計の要点をまとめます。

- 3ノードのコントロールプレーで高可用性を確保
- CPU/GPUノードを分離してリソース管理
- Ciliumによるネットワークとセキュリティ
- Argo CDによるGitOpsで自動化
- Prometheus + Grafanaで完全な可視性
- OpenCostでFinOpsを実践
- KServe/KubeflowでAI/ML基盤