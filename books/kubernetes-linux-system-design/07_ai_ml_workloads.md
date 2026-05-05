# 第7章 AI/MLワークロード

## はじめに

本章では、Kubernetes上で機械学習（ML）や大規模言語モデル（LLM）を動かし、推論（inference）とトレーニング（Training）を本番環境で運用する方法を学びます。

本章で学ぶ内容：
- KubernetesでのGPUリソース管理
- KServeによるモデルサーバーイング
- Kubeflow によるMLOpsパイプライン
- GPUのスケジュールリングとプロファイリング
- 推論時のスケーリングとローディング

## KubernetesのGPUサポート

### GPUリソースの管理

Kubernetesクラスター上でGPUを使うには、以下の手順が必要です。

1. **NVIDIAドライバのインストール**（ホストOSに）
2. **NVIDIA Device Pluginのデプロイ**（Kubernetesのアドオン）
3. **Podの資源としてGPUリクエスト**

### NVIDIA Device Pluginの設定

```bash
# NVIDIA Device Pluginのインストール
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/deployments/static/nvidia-device-plugin.yml
```

インストール後、以下の要でGPUリソースが確認できます。

```bash
# GPUの確認
kubectl describe node <node-name> | grep -i nvidia

# PodをGPU付きで起動
kubectl run test-gpu --image=nvidia/cuda:12.0.0-base-ubuntu22.04 -- nvidia-smi
```

### GPU Podの宣言（修正後）

```yaml
# GPU Podの宣言
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
  namespace: ai-workloads
spec:
  containers:
    - name: gpu-container
      image: nvidia/cuda:12.0.0-base-ubuntu22.04
      command: ["/bin/bash", "-c", "nvidia-smi && sleep infinity"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

### GPUノードのラベリング

GPU対応ノードには以下のラベルを付けます。

```bash
kubectl label nodes <gpu-node> nvidia.com/gpu=true
kubectl label nodes <gpu-node> nvidia.com/gpu.product=RTX-4090
```

## KServe: モデルサーバーイング

KServeはKubernetes上のモデルサーバーイングのデファクトスタンダードです。

### KServeのインストール

```bash
# KServeのインストール（Stable版）
kubectl apply -k "github.com/kserve/kserve/config/overlays/stable?ref=v0.15.0"
```

### モデルのデプロイ

```yaml
# InferenceService（Sklearnモデル）
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: iris-classifier
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
        version: "1"
      storageUri: s3://my-bucket/models/sklearn/iris-model.pkl
```

### OpenAI互換APIでのアクセス

```bash
# サービスのURLを取得
kubectl get inferenceservice
# iris-classifier.ai-workloads.example.com が返ります

# curlで推論
curl -X POST http://iris-classifier.ai-workloads.example.com/v1/classify \
  -H "Content-Type: application/json" \
  -d '{"data": [[1.0, 2.0, 3.0, 4.0]]}'
```

### ローカルのモデルサーバーとして

Kubeflow Pipelinesと連携することで、MLOpsの全体像をKubernetes上だけで実装できます。

| コンポーネント | 役割 |
|---|---|
| Katib | ハイパーパラメータチューニング |
| KFP | パイプライン実行 |
| NB | Jupyterノートブック |
| KServe | モデル サービング |
| Pipelines | パイプライン管理 |
| Notebook | ノートブック環境 |

## GPUとeBPFの連携（高度なトピック）

eBPF（Extended Berkeley Packet Filter）をK8sのネットワーク層として使うと、GPU間の通信パフォーマンスが大幅に改善されます。

```bash
# CiliumによるBPFベースのサービスリング
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.16.3/install/kubernetes/manifests/cilium-install.yaml
```

## まとめ

本章で学んだこと：

- GPUリソースの管理とNVIDIA Device Plugin
- KServeによるモデルサーバーイング
- KubeflowによるMLOパイプライン
- GPUノードのラベリングとスケジュールリング
- eBPFによるGPU間のネットワーク最適化

次章では、K8sの FinOps を学びます。
