# 第3章 Kubernetesの核心

## K8sの核心

本章では、Kubernetesのコアコンポーネントを学びます。

### コアコンポーネントのまとめ

| コンポーネント | 役割 |
|---|-|
| API Server | 管理用APIのエンドポイント
| etcd | K8sのブレイスストア（構成と状態
| Controller Manager | ノード・レプリケーション・サービス |
| Scheduler | ワークロードのスケジューラ |
| kubelet | ノードエージェント |
| kube-proxy | サービスとネットワーク |

## K8sのリソースオブジェクト

### Pod

最小のデプロイ単位です。

```yaml
# example-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: web
  name: web-pod
spec:
  containers:
  - image nginx:latest
  - name: nginx
    ports:
    - containerPort: 80
```

### Deployment

ReplicaSetとPodを管理します。

```yaml
# example-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort:
        resources:
        limits:
        requests:
          cpu: 100m
          memory: 128Mi
        - name: nginx
          image: nginx:latest
          ports:
          - containerPort: 80
```

### Service

Podへのアクセスを提供。

```yaml
# example-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80             # Cluster内
    targetPort: 80       # Container内
    nodePort: 30080      # 外部
  type: NodePort
```

### Ingress

HTTP/HTTPSルーティングを管理。

```yaml
# example-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Exact
        backend:
          service:
            name: web-service
            port:
        number: 80
```

### ConfigMapとSecret

設定と機密情報を分離。

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_ENV: production
  LOG_LEVEL: info

# Secret
apiVersion: v1
kind: Secret
metadata:
  name: web-secret
type: Opaque
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
```

## kubeletの仕組み

kubeletは各ノードで動作し、ポッドライフサイクルを管理します。

```
+-----+--+--+--+--+--+--+--+--+--+--+--+
| kubelet + kubelet   + kubelet  |
|  Pod 1     Pod 2    Pod 3       |
|                                    |
|                                    |
+-----+--+--+--+--+--+--+--+--+--+--+--+
```

### kubeletの動作

```bash
# kubeletの設定ファイル確認
cat /var/lib/kubelet/config.yaml

# ノードの確認
kubectl get nodes
kubectl describe node
```

## まとめ

本章で学んだこと：

- K8sのAPIオブジェクト
- Deployment / Service / Ingress
- ConfigMap / Secret
- kubeletの仕組み

次章では、ポッドとサービスのパターンを学びます。
