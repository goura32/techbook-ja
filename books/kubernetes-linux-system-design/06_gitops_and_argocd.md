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
### Applicationの構成例（ApplicationSet）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-apps
spec:
  generators:
  - clusters: {}
  template:
    metadata:
      name: 'cluster-{{name}}-web'
    spec:
      project: default
      source:
        repoURL: https://github.com/example/deployments.git
        targetRevision: main
        path: 'staging'
      destination:
        server: '{{server}}'
        namespace: web-apps
```

### ApplicationSetのジェネレータ

| ジェネレータ | 用途 |
|--|--|
| List | 手動でアプリケーションをリスト |
| Git | リポジトリのリストから自動生成 |
| Clusters | クラスタごとのデプロイ |
| SCM | Pull Requestから動的生成 |

## GitOpsのベストプラクティス

### 環境別の構成管理

```
git-repo/
|-- base/
|   |-- namespace.yaml
|   |-- common-labels.yaml
|-- environments/
|   |-- staging/
|   |   |-- kustomization.yaml
|   |   |-- patches/
|   |-- production/
|       |-- kustomization.yaml
|       |-- patches/
```

### Kustomizeでの設定管理

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../base
namespace: staging
commonLabels:
  env: staging
patches:
- target:
    kind: Deployment
    name: web-app
  patch: |
    - op: replace
      path: /spec/replicas
      value: 2
```

## GitOpsの利点と課題

### GitOpsの利点

| 利点 | 説明 |
|--|--|
| 変更追跡 | 全変更がGitで追跡可能 |
| 変更管理 | PRレビュープロセスで品質管理 |
| 自動化 | マニュアルなデプロイ作業から解放 |
| バックアップ | リポジトリがリソースの状態のバックアップ |
| ドキュメント | リポジトリが実際のコードのドキュメント |

### GitOpsの課題と対策

| 課題 | 対策 |
|--|--|
| Secrets管理 | Sealed Secrets、External Secretsを使用 |
| シンクロ問題 | Argo CDのDrift Detection |
| ロールバック | Gitでのバージョン管理で簡単ロールバック |
| 同時更新 | Argo CDのSync Waveで順序制御 |
## Argo Rolloutsによるデプロイ戦略

### Canaryデプロイメント

Argo CDのExtensionとしてArgo RolloutsがCanaryデプロイメントを提供します。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: hello-world
spec:
  replicas: 5
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: argoproj/rollout-demo:blue
        ports:
        - name: http
          containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: 128Mi
          limits:
            cpu: "200m"
            memory: 256Mi
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 10s}      # 10%%で10秒待機
      - setWeight: 20
      - pause: {duration: 30s}     # 20%%で30秒待機
      - analysis:
          templates:
          - template: success-rate
        pause: {duration: 120s}
      - setWeight: 30
      - pause: {duration: 60s}
      - setWeight: 50
      - pause: {duration: 300s}    # デプロイ完了まで5分待機
```

### Blue-Greenデプロイメント

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    blueGreen:
      activeService: blue-green-active
      previewService: blue-green-preview
      autoPromotionEnabled: false    # 手動承認を必須に
```

### Blue-Greenデプロイメントのメリット

| アプローチ | メリット | デメリット |
|--|--|--|
| Blue-Green | 即時ロールバック可能 | 2倍のリソース消費 |
| Canary | リソース効率がよい | デプロイに時間 |
| Recreate | シンプル | ダウンタイムあり |

### 分析テンプレート

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    count: 10
    failureLimit: 3
    provider:
      prometheus:
        address: https://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"2..",app="hello-world"}[1m])) /
          sum(rate(http_requests_total{app="hello-world"}[1m]))
    successfulRunHistoryLimit: 3
    failureHistoryLimit: 3
```

## GitOpsのセキュリティ

### GitOpsでのSecrets管理戦略

| ツール | 特徴 | 使用ケース |
|--|--|--|
| Sealed Secrets | encrypted secretsをGitにコミット | 小〜中規模 |
| External Secrets | KMS/VDIとの統合 | エンタープライズ |
| SOPS + age | 秘密鍵ベースの暗号化 | シンプルな環境 |
| HashiCorp Vault Vault | メリットベース | 高度なシークレット管理 |

### Sealed Secretsの使用例

```bash
# kubesealのインストール
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.0/controller.yaml

# Secretの暗号化
echo "SECRET=my-value" | kubeseal --format=yaml > sealed-secret.yaml
# sealed-secret.yamlを作成したものをGitにコミット

# Argo CDでSync
argo cd app create sealed-secrets --repo URL --path sealed-secret.yaml
```

## GitOpsのベストプラクティスまとめ

1. **Infrastructure as Code**: Terraformでインフラを定義
2. **GitOpsでデプロイ**: Argo CDでGitから自動Sync
3. **環境分離**: 環境ごとに別ブランチまたは別リポジトリ
4. **Secrets管理**: Sealed Secrets/External Secretsで管理
5. **モニタリング**: Argo CDのイベントをSlack/メール通知
6. **DRプラン**: etcdバックアップ＋Gitリポジトリで完全復旧