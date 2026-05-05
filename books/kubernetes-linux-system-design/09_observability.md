# 第9章 オブザーバビリティ

## はじめに

本章では、Kubernetesクラスター内のアプリケーションの観測可能性（オブザーバビリティ）を確保する方法を学びます。監視、ログ収集、分散トレーシングが組み合わさって初めて、複雑な分散システムの正常性を把握できます。

本章で学的内容：
- オブザーバビリティの3本柱（_metrics、Logs、Traces）
- PrometheusとGrafanaの統合
- Fluentdによるログ収集
- OpenTelemetryによる分散トレーシング
- Kubernetesの組み込みメトリクス

## オブザーバビリティとは

オブザーバビリティ（観測可能性）とは、外部からの観測だけで内部状態を把握できるシステムの特性です。SREでは以下の3つを「3本柱」として捉えます。

| 指標名 | 説明 | 主要ツール |
|--------|------|-----------|
| メトリクス | 数値で計測された時系列データ | Prometheus, Grafana, Datadog |
| ログ | イベントの記録 | Fluentd, Loki, ELK Stack |
| トレース | リクエストのトレース | Jaeger, Zipkin, OpenTelemetry |

これらのデータを統合して監視ダッシュボードを作ることで、障害の早期発見と原因特定が可能になります。

### Prometheusのデプロイ

PrometheusはKubernetesの標準メトリクス収集システムです。

```yaml
# prometheus-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:latest
          ports:
            - containerPort: 9090
              name: http
          volumeMounts:
            - name: config
              mountPath: /etc/prometheus
            - name: data
              mountPath: /prometheus
      volumes:
        - name: config
          configMap:
            name: prometheus-config
        - name: data
          emptyDir: {}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
          - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
          - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
```

### Grafanaダッシュボードのインストール

```bash
# grafanaデプロイ
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest
          ports:
            - containerPort: 3000
          env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: "admin"
          volumeMounts:
            - name: data
              mountPath: /var/lib/grafana
      volumes:
        - name: data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: LoadBalancer
  ports:
    - port: 3000
      targetPort: 3000
  selector:
    app: grafana
EOF

# ポートフォワーディングでアクセス
kubectl port-forward -n monitoring svc/grafana 3000:3000
```

ブラウザで `http://localhost:3000` にアクセス（ユーザー/パスワード: admin/admin）。

Prometheusのデータソースとして追加：

```bash
# curlでデータソースを登録
curl -X POST http://localhost:3000/api/datasources \
  -H "Content-Type: application/json" \
  -d '{
    "name": "prometheus",
    "type": "prometheus",
    "url": "http://prometheus.monitoring:9090",
    "access": "proxy"
  }'
```

### Fluentdによるログコレクション

Fluentdはログの収集・処理・ルーティングを行うログ集約ツールです。

```yaml
# fluentd-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.16
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch.logging.svc.cluster.local"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
          resources:
            limits:
              memory: 512Mi
              cpu: 500m
            requests:
              memory: 256Mi
              cpu: 100m
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

### OpenTelemetry（分散トレース）

OpenTelemetryはクラウドネイティブな分散トレーシングの標準です。

```yaml
# otel-collector-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector:latest
          ports:
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
          volumeMounts:
            - name: config
              mountPath: /etc/otel-collector
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: "0.0.0.0:4317"
          http:
            endpoint: "0.0.0.0:4318"
    processors:
      batch:
        send_batch_max_size: 1000
        timeout: 5s
    exporters:
      jaeger:
        endpoint: "jaeger-collector.default:14250"
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [jaeger]
```

### Kubernetesの組み込みメトリクス

Kubernetesクラスターはnode-exporter、kube-state-metricsなどの組み込みメトリクスを提供します。

```bash
# ノードのCPU・メモリ使用量を確認
kubectl top nodes

# PodのCPU・メモリ使用量を確認
kubectl top pods -n default

# Kubernetesのメトリクスを取得
curl -s http://prometheus.monitoring:9090/api/v1/query \
  -d 'query=kube_pod_container_resource_usage'
```

## まとめ

本章で学んだこと：

- オブザーバビリティとはLogs + Metrics + Tracesの3本柱
- PrometheusはKubernetesの標準メトリクス収集システム
- Grafanaでメトリクスを可視化
- Fluentdでログを集約
- OpenTelemetryで分散トレーシング
- kubectl topでリアルタイムのCPU／メトリクス確認

次章では、K8sのプラットフォーム設計を学びます。
