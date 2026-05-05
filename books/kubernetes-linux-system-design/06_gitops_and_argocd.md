# 第6章 GitOpsとArgo CD

## GitOpsとArgo CD

本章では、GitOps（Gitベースの運用）とArgo CDについて学びます。

### GitOpsとは

GitOpsは、**Gitリポジトリを唯一の信頼できるソース**として運用する手法です。

```
Gitリポジトリ（YAML）
    |
    |（デプロイ）
    |
+-----+-----+-----+-----+
| Argo CD          |
| App of Appsパターン |
+-----+-----+-----+-----+
    |
    |（自動管理）
    |
Kubernetes
Cluster
```

### Argo CDの基本概念

```
Application
  -> ApplicationSet
    -> GitRepo
      -> Helm Chart
        -> Kubernetes
```

### Argo CDのインストール

```bash
# ArgoCDのインストール
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 初期パスワード取得
argocd admin initial-password -n argocd

# Port-forwarding（ローカル）
kubectl port-forward service/argocd-server -n argocd 8080:443
```

### Argo CDのApplication

```yaml
# Applicationの定義
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
  namespace: argocd
spec:
  project: default
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  source:
    repoURL: https://github.com/myorg/web-app.git
    targetRevision: main
    path: k8s/
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### App of Appsパターン

```yaml
# parent-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-platform.git
    path: clusters/prod
    targetRevision: main
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
```

## GitOpsとCICD

### ArgoCD + Argo Events

```
GitHub (push)
    ->
Argo Events (event)
    ->
Argo Workflows (workflow)
    ->
Argo CD (sync)
    ->
Kubernetes (deploy)
```

### GitOpsのパターン

```
開発者(Git) -> プルリクエスト -> CI（テスト） -> マージ -> Argo CD（sync） -> クラスタ
```

## まとめ

本章で学んだこと：

- GitOpsの概念
- Argo CDのインストールと基本
- App of Appsパターン
- CI/CDとの統合

次章では、K8s上でのAI/MLワークロードを学びます。
