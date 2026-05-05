# 第8章 FinOpsとコスト管理

## FinOps

本章では、K8s上のFinOps（FinOps = financial ops）について学びます。

### FinOpsの概要

```
K8sリソース -> 予算 -> コスト配分 -> OpenCost -> Grafana
```

| 分野 | 用語 |
|---|-|
| FinOps | Financial Operations |
| FinOps | コスト最適化 |

```
+------+ +------+ | FinOps +------+ |
| Resource(VM) | Cost(GPU) +------+ |
+------+ +------+ +------+ +------+ +-----+
  K8s    AWS    GCP    Azure    OpenCost    Grafana
+-----+
```

### OpenCostのインストール

```bash
# OpenCostのインストール
kubectl apply -f https://raw.githubusercontent.com/opencost/opencost/develop/ksonnet/k9s.yaml
```

### OpenCostの詳細

| 項目 | 説明 |
|---|-|---|
| CPU | CPUのコスト |
| Memory | メモリの使用コスト|
| GPU | GPUの実行コスト |
| Storage | ストレージコスト |

### FinOpsのツール

| ツール | 特徴 |
|---|-|
| OpenCost | 標準。K8sネイティブ |
| Kubecost | 有料。UI |
| CloudHealth | 有料。多 cloud |
| Datadog | 有償。多 cloud |
| Prometheus | 無料。監視 |
| Grafana Labs | 有償。可視化 |

## まとめ

本章で学んだこと：

- FinOps：コスト管理のフレームワーク
- OpenCost：K8s上のコスト可視化
- リソースの最適化

次章では、K8sのオブザーバビリティについて学びます。

## GPUコスト管理とリソース最適化

### GPUインスタンスタイプの選択

クラウドプロバイダーごとにGPUインスタンスタイプが異なります。適切な選択がコスト削減につながります。

| インスタンスタイプ | GPU | メモリ | vCPU | 時間単位コスト(USD) | 用途 |
|---|--|-|-|------|-|-|
| g4dn.xlarge | T4 x1 | 16GB | 4 | ~0.53 | フロントエンド推論 |
| g4dn.2xlarge | T4 x1 | 32GB | 8 | ~1.06 | バッチ処理 |
| g5.xlarge | A10G x1 | 24GB | 4 | ~0.87 | テキスト生成推論 |
| p3.2xlarge | V100 x1 | 16GB | 8 | ~3.06 | トレーニング |
| p3.8xlarge | V100 x4 | 244GB | 32 | ~12.24 | 大規模トレーニング |
| p4d.24xlarge | A100 x8 | 320GB | 96 | ~32.77 | 超大規模LLM訓練 |

### リソース割り当ての最適化

#### RequestsとLimitsの適切な設定

```yaml
# リソース割り当ての例
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
    - name: inference
      image: my-inference:latest
      resources:
        requests:
          nvidia.com/gpu: 1          # 必ず確保するリソース
          cpu: "2"
          memory: "8Gi"
        limits:
          nvidia.com/gpu: 1          # 最大使用量
          cpu: "4"
          memory: "16Gi"
```

#### コンテナのサイズ調整

GPUコンテナを極端に大きくすると、他のワークロードとの競合が発生します。

```yaml
# 過剰なリソース割り当て（NG例）
limits:
  nvidia.com/gpu: 4
  cpu: "32"
  memory: "128Gi"
  # → 実際にはGPUの50%しか使われていない場合

# 適切なリソース割り当て（推奨）
limits:
  nvidia.com/gpu: 1
  cpu: "4"
  memory: "16Gi"
  # → タスクごとにGPUを分割
```

### 分散推論とバッチ推論

1つのGPUに複数のモデルを配置することを考えます。

```bash
# NVIDIA MPS（Multi-Process Service）でGPUを共有
nvidia-cuda-mps-control -d

# vLLMのバッチ推論設定
# batch_sizeごとにGPUメモリを効率的に使用できる

## Kubeflowとコストの統合

### Kubeflowパイプラインのコスト可視化

KubeflowのPipelineは、各ステップの実行コストを追跡できます。

```bash
# Kubeflowのインストール
kubectl create namespace kubeflow
kubectl apply -f https://raw.githubusercontent.com/kubeflow/manifests/v1.7.0/kubeflow-katib/katib-install/katib-install.yaml
kubectl apply -f https://raw.githubusercontent.com/kubeflow/manifests/v1.7.0/kubeflow-training/training-operator-install.yaml
```

```yaml
# KatibのHyperParameterチューニング（コスト最適化目的）
apiVersion: kubeflow.org/v1beta1
kind: Experiment
metadata:
  name: cost-optimized-training
spec:
  algorithm:
    algorithmName: bayesianoptimization
    maxCount: 20
    parallelTrialCount: 2        # コスト抑制のため同時実行数を制限
  objective:
    type: minimize
    goal: 0.001
  parameters:
    - name: learning_rate
      parameterType: double
      minValue: 0.0001
      maxValue: 0.01
  trialTemplate:
    baseTemplatePath: training-job.yaml
```

### エラーコスト管理

```bash
# KubeflowのErrorリソース管理
# GPU利用を制限するためにLimitRangeを設定
apiVersion: v1
kind: LimitRange
metadata:
  name: gpu-limit-range
  namespace: kubeflow
spec:
  limits:
    - type: Container
      max:
        nvidia.com/gpu: 2        # 1 Podあたり最大2GPU
      default:
        nvidia.com/gpu: 1        # デフォルトは1GPU
```

### Auto-scalingとコスト最適化

```bash
# K8s HPA（Horizontal Pod Autoscaler）の設定
kubectl autoscale deployment inference-server   --min=1   --max=10   --cpu-percent=70

# Vertical Pod Autoscaler（VPA)の設定
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: inference-vpa
spec:
  targetRef:
    name: inference-server
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    minAllowed:
      cpu: "1"
      memory: "2Gi"
    maxAllowed:
      cpu: "8"
      memory: "32Gi"
```

## まとめ

本章で学んだこと：

- FinOps：コスト管理のフレームワーク
- OpenCost：K8s上のコスト可視化
- GPUインスタンスタイプの選択がコストに影響
- リクエストとリミットの適切な設定
- Kubeflowでコスト最適化可能
- HPA/VPAで自動スケーリングによるコスト削減
- LimitRangeでリソース使用を制約

次章では、K8sのオブザーバビリティについて学びます。
