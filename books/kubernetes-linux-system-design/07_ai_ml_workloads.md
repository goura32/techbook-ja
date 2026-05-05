# 第7章 AI/MLワークロード

## K8sでAI/MLを動かす

本章では、K8s上でAI/MLワークロードを動かす方法を学びます。

### AI/MLのアーキテクチャ

```
+-----+-----+-----+-----+-----+-----+
| Training   | Inference |  Monitoring |
| (GPU)      (CPU)      |  (CPU)      |
+-----+-----+-----+-----+-----+-----+
    |       |       |       |       |
    v       v       v       v       v
+-----+-----+-----+-----+-----+-----+
| Kubernetes Cluster                  |
| +--+--+--+--+--+--+--+--+--+--+--+--+|
| | Pod 1 | Pod 2 | Pod 3 | Pod 4 |  |
| +--+--+--+--+--+--+--+--+--+--+--+--+|
+-----+-----+-----+-----+-----+-----+
```

### GPUの扱い

K8s上でのGPUリソース管理は重要な課題です。

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

### KServe

KServeはK8s上のModel Servingの標準。

```bash
# KServeのインストール
kubectl apply -k "github.com/kserve/kserve/config/overlays/stable?ref=v0.13.0"

# KServeのModel Serving
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: iris-classifier
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: s3://my-bucket/models/sklearn/iris-model.pkl
```

### Kubeflow

KubeflowはK8s上のMLOpsプラットフォーム：

| メンバ | 役割 |
|---|-|
| Katib | ハイパーパラメータチューニング |
| KFP | パイプライン |
| NB | ノートブック |
| KServe | Model Serving |
| Pipelines | パイプライン |
| Notebook | ノートブック |

## まとめ

本章で学んだこと：

- K8sでのGPUリソース管理
- KServeがModel Servingのデファクト
- KubeflowがMLOpsプラットフォーム

次章では、K8sの FinOps を学びます。
