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
