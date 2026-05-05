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
## AIワークロードのリソース管理

### GPUリソースのリクエスト

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: gpu-container
    image: nvidia/cuda:12.2.0-runtime-ubuntu22.04
    resources:
      limits:
        nvidia.com/gpu: 1
      requests:
        nvidia.com/gpu: 1
```

### CUDAとcuDNNのバージョン管理

```bash
# GPUのバージョン確認
nvidia-smi
# OUTPUT: NVIDIA-SMI 535.54.03, Driver Version: 535.54.03, CUDA Version: 12.2

# コンテナでのCUDAバージョン确认
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

### GPUのメトリクス監視

```bash
# GPUメトリクスの取得
nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total --format=csv

# Prometheus NVIDIA DCGM Exporterの設定
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/main/examples/kube-manifests/dcgm-exporter.yaml
```

## KServeによるモデルサービング

### KServeの基本的な構成

```yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: InferenceService
metadata:
  name: my-model
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      storageUri: "s3://my-bucket/models/my-model"
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
        nvidia.com/gpu: 1
```

### KServeのモデルフォーマット

| フォーマット | 説明 | KServeでの使用方法 |
|--|--|--|
| PyTorch | PyTorchモデルファイル | PyTorchPredictor |
| TensorFlow | SavedModel形式 | TensorFlowPredictor |
| ONNX | Open Neural Network Exchange | ONNXRuntimePredictor |
| Scikit-learn | pickle形式 | SklearnPredictor |

## Kubeflowとの比較

| ツール | 長所 | 短所 | 使用ケース |
|--|--|--|
| KServe | 軽量。K8sに統合 | 単一モデルに特化 | APIサービング |
| Kubeflow | 包括的なMLプラットフォーム | やや重い | 全MLパイプライン |
| Seldon Core | 多言語サポート | 設定が複雑 | 本番モデルサービング |
## トレーニングのためのK8s設定

### JobとCronJobによるバッチトレーニング

```yaml
# 並列分散トレーニング
apiVersion: batch/v1
kind: Job
metadata:
  name: distributed-training
spec:
  parallelism: 4
  template:
    spec:
      containers:
      - name: training
        image: pytorch/pytorch:2.3.0-cuda12.1-runtime
        command:
        - python
        - -m
        - torch.distributed.run
        - --nproc_per_node
        - "4"
        train.py
        resources:
          limits:
            nvidia.com/gpu: 4
      restartPolicy: Never
```

### Distributed Trainingのアーキテクチャ

```
                    Worker 0 (GPU x4)
                       |
    +---+---+---+
    |   |   |   |
    |   |   |   |
    |   |   |   |
    +---+---+---+
    Parameter Server (CPU)
    |   |   |   |
    +---+---+---+
                    Worker 1 (GPU x4)
```

## MLOpsパイプラインの自動化

### Argo Workflowsによるデータパイプライン

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: ml-pipeline
spec:
  entrypoint: pipeline
  templates:
  - name: pipeline
    steps:
    - - name: data-preprocessing
        template: data-preprocessing
    - - name: model-training
        template: model-training
        dependencies: [data-preprocessing]
    - - name: model-evaluation
        template: model-evaluation
        dependencies: [model-training]
    - - name: model-deployment
        template: model-deployment
        dependencies: [model-evaluation]
```

### ワークフローの実行例

```bash
# AIパイプラインの実行
kubectl apply -f ml-pipeline.yaml

# ワークフローの状態確認
argo list
argo get ml-pipeline

# ログ確認
argo logs ml-pipeline -f
```

## K8s上のモデルサービングパターン

### ローディングパターン

1. **ホットパス**: 起動時に全モデルをロード（低レイテンシ）
2. **コールドパス**: 必要に応じてモデルを動的にロード（リソース効率）
3. **マルチテナント**: 複数モデルを共有GPUで時間分割（コスト効率）

### モデルフォーマットの変換ツール

| 目的 | ツール | 変換元 | 変換先 |
|--|--|--|--|
| 圧縮 | ONNX Runtime | PyTorch/TensorFlow | ONNX |
| 最適化 | TensorRT | ONNX | TensorRT Engine |
| 量子化 | GGUF | フローティングポイント | 量子化モデル |
| 共有 | OpenVINO | 任意 | OpenVINO IR |

## AI/ML基盤の設計原則

### K8s AI/ML基盤の設計原則

1. **GPUリソースの抽象化**: device pluginによるGPUのK8sリソースとして管理
2. **自動スケーリング**: AIPGに基づいた自動スケーリング
3. **モデル版管理**: 全モデルをバージョン管理し、簡単にロールバック可能に
4. **マルチテナント**: チーム別のリソース制約と分離
5. **データパイプライン**: 学習データの効率的な処理と保存

### AI基盤設計のチェックリスト

| 項目 | 重要度 | 確認内容 |
|--|--|--|
| GPUリソースの予約 | 高 | 必要なGPU数とタイプの確保 |
| ストレージ性能 | 高 | データ読み書き速度の最適化 |
| モデルキャッシュ | 中 | よく使うモデルの事前ロード |
| メトリクス監視 | 高 | GPU利用率、メモリ使用量、推論レイテンシ |
| ドリフト検出 | 中 | モデル精度の経時変化 |