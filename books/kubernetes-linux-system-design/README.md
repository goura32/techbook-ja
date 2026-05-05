# Kubernetes × Linux システム設計

> クラウドネイティブの中心であるKubernetesとLinuxを、両輪として学ぶ実践書。

## 📖 目次

| 章 | タイトル | 内容 |
|---|---|-|
| [第0章](books/kubernetes-linux-system-design/ch00/chapter00.md) | はじめに | 本書の目的・対象読者・前提知識 |
| [第1章](books/kubernetes-linux-system-design/ch01/chapter01.md) | なぜ今、Kubernetesなのか | クラウドネイティブのトレンド・AI/MLワークロード |
| [第2章](books/kubernetes-linux-system-design/ch02/chapter02.md) | Linux基礎を再確認 | コンテナのLinux実装・cgroups・namespaces |
| [第3章](books/kubernetes-linux-system-design/ch03/chapter03.md) | Kubernetesの核心 | コアイメージの役割、APIオブジェクト |
| [第4章](books/kubernetes-linux-system-design/ch04/chapter04.md) | ポッドとサービスの設計 | Deployment、StatefulSet、Job、CronJob |
| [第5章](books/kubernetes-linux-system-design/ch05/chapter05.md) | ネットワークとeBPF | CNI、Cilium、Hubbleによる可視化 |
| [第6章](books/kubernetes-linux-system-design/ch06/chapter06.md) | GitOpsとArgo CD | ArgoCDのApp of Appsパターン |
| [第7章](books/kubernetes-linux-system-design/ch07/chapter07.md) | AI/MLワークロード | KServe、Kubeflow、GPUの扱い |
| [第8章](books/kubernetes-linux-system-design/ch08/chapter08.md) | FinOpsとコスト管理 | OpenCost、リソース最適化 |
| [第9章](books/kubernetes-linux-system-design/ch09/chapter09.md) | オブザーバビリティ | Prometheus、Grafana、OpenTelemetry |
| [第10章](books/kubernetes-linux-system-design/ch10/chapter10.md) | 実践プロジェクト | K8sプラットフォーム設計の実装 |

## 🚀 始め方

```bash
# 1. K8sのインストール
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# 2. Minikubeでの環境構築
minikube start --cpus=4 --memory=8192 --driver=docker

# 3. ArgoCDのインストール
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 4. Prometheusのインストール
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

## 📚 構成（3フェーズ）

| フェーズ | 範囲 | テーマ |
|---|---|-|
| フェーズ1：基盤 | ch01〜ch03 | K8sトレンド、Linuxの仕組み、K8sの核心 |
| フェーズ2：パターン | ch04〜ch06 | 設計パターン、eBPF、GitOps |
| フェーズ3：実運用 | ch07〜ch10 | AI/ML、FinOps、オブザーバビリティ、プラットフォーム設計 |

## 🛠 テクノロジースタック

| カテゴリ | テクノロジー |
|---|-|
| コンテナオーケストレーター | Kubernetes |
| CNI | Cilium |
| GitOps | Argo CD/Flux |
| メトリクス | Prometheus |
| 可視化 | Grafana |
| Distributed Tracing | OpenTelemetry |
| Model Serving | KServe |
| MLOps | Kubeflow |
| コスト管理 | OpenCost |
| AI Workload | NVIDIA/Docker |
| AI Workload | NVIDIA AI |
| AI Workload | NVIDIA |

## LICENSE

MIT License
