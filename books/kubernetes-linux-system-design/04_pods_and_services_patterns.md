# 第4章 ポッドとサービスのパターン

## ポッド（Pod）のパターン

本章では、ポッドとサービスの代表的なパターンを学びます。

## Replication Set

ポッドの複製を自動管理。

```yaml
# ReplicaSetの自動管理
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-nginx
  template:
    metadata:
      labels:
        app: web-nginx
    spec:
      containers:
        image: nginx:1.0
        ports:
          - containerPort: 80
        name: nginx
```

## State

ポッドの永続化。

### StatefulSet

ステートレスな状態を維持。

```yaml
## StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-statefulset
spec:
  serviceName: web-statefulset
  replicas: 3
  template:
    metadata:
      labels:
        app: web-stateful
    spec:
      containers:
        name: nginx
        image: nginx:latest
        ports:
          - containerPort: 80
        volumeMounts:
        - name: data-volume
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data-volume
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 1Gi
```

### Deployment

最も一般的なデプロイパターン。

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        |  同時
      maxUnavailable: 1  |  削除
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: my-app:latest
        resources:
          limits:
            cpu: 500m
            memory: 256Mi
          requests:
            cpu: 500m
            memory: 128Mi
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

### JobとCronJob

```yaml
# Job
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  templates:
    spec:
      containers:
      - name: backup
        image: backup-tool:latest
      restartPolicy: Never

# CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cronjob
spec:
  schedule: "0 2 */1 * *"
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool:latest
        resources:
          limits:
            cpu: 500m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 128Mi
      restartPolicy: Never
```

## ポッドとノード

ポッドはノード上で実行されます。

```
+---------------------------+
| Pod 1 (cpu: 100m, mem: 128Mi) |
+---------------------------+
| Pod 2 (cpu: 200m, mem: 256Mi) |
+---------------------------+
| Node 1                    |
| cpu: 1000m, mem: 4096Mi  |
+---------------------------+
```

## まとめ

本章で学んだこと：

- Deploymentがデフォルトに
- StatefulSetがステートを持つ
- JobとCronJobがバッチ処理
- ポッドのCPU/メモリのリソース管理

次章では、K8sのネットワークを学びます。
## Init Container

Init ContainerはメインのContainerの前に実行される特別なコンテナです。初期化タスクに使用します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-container
    image: my-app:latest
    ports:
    - containerPort: 8080
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db-service 5432; do echo waiting for db; sleep 2; done']
  - name: init-config
    image: busybox
    command: ['sh', '-c', 'cp /config-dir/*.conf /etc/app/']
    volumeMounts:
    - name: config-volume
      mountPath: /config-dir
    - name: app-config
      mountPath: /etc/app
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: app-config
    emptyDir: {}
```

### Init Containerの使用例

| 使用ケース | 説明 |
|--|--|
| データベース接続確認 | DBがReadyになるまで待機 |
| 設定ファイルの生成 | 環境変数から設定ファイルを動的生成 |
| 依存サービスの確認 | キャッシュ、APIなど外部サービスの可用性確認 |

## リソースの管理

### リクエストと制限

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

### HPA（Horizontal Pod Autoscaler）

自動スケーリングの設定です。

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### VPA（Vertical Pod Autoscaler）

 podのリソースを自動調整します。

```yaml
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    name: web-app
  updatePolicy:
    updateMode: "Auto"
```

## Podのライフサイクル

### Podのライフサイクル状態

1. Pending: APIサーバーがPodを受け付け、リソースを割り当て中
2. ContainerCreating: コンテナランタイムがイメージをダウンロード中
3. Running: コンテナが正常に起動し、実行中
4. Succeeded: コンテナが正常終了
5. Failed: コンテナが異常終了
6. Unknown: 状態が不明

### Graceful Shutdown

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-shutdown
spec:
  containers:
  - name: web-app
    image: nginx:1.25
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 10"]
```

## まとめ

本章で学んだポイント：

- PodはK8sの最小デプロイ単位
- ReplicaSetでPodの複製数を管理
- StatefulSetでステートフルなワークロードを管理
- Init Containerで初期化タスクを実行
- HPA/VPAでリソースを自動調整
- リソースリクエストと制限でQuality of Serviceを保証