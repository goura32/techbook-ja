# 第9章 オブザーバビリティ

## K8sのオブザーバビリティ

本章では、K8sのオブザーバビリティについて学びます。

### オブザーバビリティとは

```
+-----+-----+-----+-----+-----+-----+-----+-----+
| Observability = | Logs(ログ) + Metrics(メトリクス)|
| Tracing(トレース)| +------+ +------+ +------+ +------+ |
+-----+-----+-----+-----+-----+-----+-----+
```

| 指標 | 説明 |
|---|-|
| ログ | 記録されたイベント |
| メトリクス | 計測された数値 |
| トレース | リクエストの追跡 |

### Prometheus

K8sの標準メトリクス。

```yaml
# Metrics（メトリクス）設定
apiVersion: v1
kind: Pod
metadata:
  name: prometheus
  namespace: prometheus
spec:
  containers:
    containers:
    - name: prometheus
      image: prom/prometheus:latest
      ports:
        - containerPort:
          containerPort: 9090
      volumeMounts:
        volumes:
        - name: prometheus-config
          volumeConfigMap:
            name: prometheus-config
            - name: prometheus-config
              volumeConfigMap:
                name: prometheus-config
```

### Prometheus + Grafana

Grafanaで可視化。

```bash
# Grafanaのインストール
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana --namespace monitoring

# GrafanaにPrometheusをダッシュボードを追加
```

### OpenTelemetry（OTel）のトレース

OpenTelemetryは標準的なトレース。

```yaml
# OpenTelemetryの設定
apiVersion: v1
kind: Pod
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  containers:
  - name: otel-collector
    image: otel/opentelemetry-collector:latest
    volumeMounts:
    - name: otel-config
      mountPath: /conf
  volumes:
  - name: otel-config
    configMap:
      name: otel-config
```

## まとめ

本章で学んだこと：

- オブザーバビリティとはLogs + Metrics + Traces
- MetricsがK8sに標準
- Grafana + OpenTelemetryが分散トレース

次章では、K8sのプラットフォーム設計を学びます。
