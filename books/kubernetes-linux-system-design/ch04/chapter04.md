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
