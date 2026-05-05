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
