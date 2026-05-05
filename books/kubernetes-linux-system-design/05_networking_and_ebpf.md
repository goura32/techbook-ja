# 第5章 ネットワークとeBPF

## eBPF（eBPF）とネットワーク

本章では、K8sのネットワークの仕組みを学びます。重点的にeBPFによるネットワーク高速化を解説します。

### K8sのネットワークモデル

```
+-----+-----+-----+-----+-----+-----+-----+
| Pod A | Pod B | Pod C | Pod D | Pod E |
| 192.168.1.1 | 192.168.1.2 | 192.168.1.3 | | 192.168.1.4 |
+-----+-----+-----+-----+-----+-----+-----+
      |       |       |       |       |
    +------+ +------+ +------+ +------+ |
    |CNI: Cilium| | CNI: Calico |    CNI: Flannel   |
    +------+ +------+ +------+ +------+ |
      |       |       |       |       |
    +------+ +------+ +------+ +------+ |
    | Pod F | Pod G | Pod H | Pod I | Pod J |
| 192.168.2.1 | 192.168.2.2 | 192.168.2.3 | | 192.168.2.4 |
+-----+-----+-----+-----+-----+-----+-----+
      |       |       |       |       |
    +------+ +------+ +------+ +------+ |
    | Network |  Gateway  | Router    |
    +------+ +------+ +------+ +------+ |
      |       |       |       |       |
    +------+ +------+ +------+ +------+ |
    | Internet  | Cloud     | On-Prem  |
  +-----+-----+-----+-----+-----+-----+
```

### CNI（Container Network Interface）の選択

| CNI | 説明 | 使用例 |
|---|-|---|
| Cilium | eBPFベース。高速。可視性 |
| Calico | BGPベース。大規模クラスターに |
| Flannel | VXLN。シンプル。小〜中規模に |
| Canal | Calico + Flannel |
| weave-net | Weave。分散型。 |

### Ciliumのアーキテクチャ

```
+-----+-----+-----+-----+
| Ciliumエージェント |
| Ciliumのコントローラ |
| Hubble（可視化 |
+-----+-----+-----+-----+
      |       |       |
    +------+ +------+ +------+
    | BPF   | eBPF | XDP  |
    |  Cgroup | eBPF |      |
    +------+ +------+ +------+
      |       |       |
    +------+ |
    +-----+-----+-----+-----+
```

## eBPFの基礎

### eBPF（拡張バーンフィールドプログラミング）

eBPFは、Linuxカーネル内で安全にカーネルコードを拡張する仕組みです。

```bash
# eBPFの活用例
kubectl get pods -n kube-system | grep cilium
kubectl -n kube-system get pods | grep cilium
helm install cilium cilium/cilium --namespace kube-system
```

## K8sのサービス

### Service Types

| タイプ | 説明 |
|---|-|
| ClusterIP | クラスタ内からのみアクセス |
| NodePort | 外部からのアクセスが可能 |
| LoadBalancer | クラウドプロバイダーのLB |
| ExternalName | DNS名へマッピング |
| ExternalIP | 外部IPへマッピング |

## CiliumとHubble

CiliumはHubble（トラフィック可視化）と組み合わせて使用します。

```yaml
# Hubbleの設定
apiVersion: cilium.io/v2alpha1
kind: CiliumNetworkPolicy
metadata:
  name: allow-all
spec:
  description: "All traffic"
  endpointSelector: {}
  ingress:
  - {}
```

## まとめ

本章で学んだこと：

- K8sのネットワークモデル
- CNIの選択（Cilium, Calico, Flannel）
- eBPFによる高速ネットワーク
- Hubbleによる可視化

次章では、K8sのアーキテクチャを学びます。
## K8sネットワークアーキテクチャ

### Network Policyの詳細設定

```yaml
# デフォルト拒否のNetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

# 特定のPodへのアクセス許可
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 80
```

### CNIプラグインの実践

| CNIプラグイン | 特徴 | 用途 |
|--|--|--|
| Cilium | eBPFベース。超高パフォーマンス | ファイアウォール、ロードバランシング |
| Calico | ネットワークポリシーが豊富 | エンタープライズ |
| Flannel | シンプル。overlayネットワーク | 小規模クラスター |
| Weave Net | メッシュ対応 | マイクロサービス |

## eBPFの仕組み

### eBPFとは

eBPF（Extended Berkeley Packet Filter）はLinuxカーネル上でプログラムを安全に実行する仕組みです。

### eBPFによるネットワーク監視

```bash
# bpftraceによるネットワークパケットの監視
bpftrace -e '
    kprobe:tcp_conn_request {
        printf("Connection request: %p %u:%u -> %u:%u\n",
               args->sk,
               load ((unsigned int *)args->sk + 24),
               load ((unsigned int *)args->sk + 28),
               load ((unsigned int *)args->sk + 4),
               load ((unsigned int *)args->sk + 8));
    }
'
```

### CiliumのK8s統合

```bash
# Ciliumのインストール
cilium install

# Ciliumステータス確認
cilium status --wait

# NetworkPolicyの適用
cilium endpoint policy -i <ep-id> -f policy.json
```

### eBPFによるパフォーマンス最適化

| 最適化 | 説明 | 効果 |
|--|--|--|
| KubeProxy代替 | CiliumによるService LB | 90%%のレイテンシ削減 |
| ポリシー評価 | カーネル空間でのファイアウォール | 95%%のパフォーマンス向上 |
| DNS最適化 | カーネル空間でのDNS解決 | 50%%のDNS改善 |
| 可視化 | プログラムレベルの監視 | 詳細なトレーシング |
## eBPFの詳細技術

### eBPFのセキュリティ機能

```bash
# eBPFによる不正アクティビティの検出
sudo bpftrace -e '
    tracepoint:raw_syscalls/sys_enter {
        @syscalls[comm] = count();
    }
    timer:s:5 {
        clear(@syscalls);
    }
'

# Network Policy enforcement logs
sudo bpftrace -e '
    kprobe:cilium_call_hook {
        printf("CILIUM POLICY: verdict=%d source_ip=%d port=%d\n",
               args->verdict, args->source_ip, args->port);
    }
'
```

### eBPFによるサービスメッシュの実現

Cilium Service Meshは従来のSidecarパターンに依存せずにネットワーク機能をカーネル空間で実行します。

| 機能 | Sidecar方式 | eBPF方式 |
|--|--|--|
| レイテンシ | Sidecar追加のオーバーヘッド | カーネル空間で直接処理 |
| リソース消費 | 追加のコンテナCPU/Memory | カーネル内の最小オーバーヘッド |
| 可視性 | Sidecar経由のメトリクス | カーネルレベルの完全な可視性 |
| プラグイン | 多くのサイドカーが必要 | プラグイン方式で柔軟 |

### eBPFの主要ユースケース

1. **ネットワーク**: CNIプラグイン、Service Mesh、Load Balancing
2. **セキュリティ**: NetworkPolicy、ファイアウォール、IDS/IPS
3. **モニタリング**: カーネルレベルのトレース、メトリクス収集
4. **デバッグ**: アプリケーションからカーネルへのトレーシング

## ネットワーク診断ツール

### K8sのネットワークトラブルシューティング

```bash
# Pod間接続テスト
kubectl exec -it pod-a -- wget -qO- http://service-b:8080/health
kubectl exec -it pod-b -- wget -qO- http://service-c:8080/health

# DNS解決テスト
kubectl debug -it --image=radial/busyboxplus:curl -n default debug-dns --  nslookup service-a.default.svc.cluster.local
kubectl debug -it --image=radial/busyboxplus:curl -n default debug-dns --  nslookup service-a.default.svc.cluster.local svc.cluster.local

# NetworkPolicyのデバッグ
kubectl get networkpolicy -A
kubectl describe networkpolicy -n <NAMESPACE>

# Ciliumのデバッグ
cilium endpoint list
cilium status --verbose
cilium monitor
```

### ネットワークパフォーマンスベンチマーク

```bash
iperf3クライアント
iperf3 -c <サーバーIP> -t 30 -p 5201

iperf3サーバー
iperf3 -s -p 5201

# Kubernetesクラスター内のiperf3テスト（Pod経由）
kubectl run iperf3-server -it --image=dgihm/iperf3 --restart=Never -- -s
kubectl run iperf3-client -it --image=dgihm/iperf3 --restart=Never -- -c iperf3-server.default.svc.cluster.local
```

## ネットワーク設計パターン

### Namespaced Network Architecture

```
+-------------------+  Internet   +-------------------+
|  Ingress (Cilium)| <---------> |  External Clients |
+--------+----------+             +-------------------+
         |
+--------v----------+
|   Cilium Load    |
|   Balancer       |
+--------+----------+
         |
+--------v----------+
|  Namespace A     |
|  (Production)    |
+--------+----------+
         |
+--------v----------+
|  Namespace B     |
|  (Staging)       |
+--------+----------+
```

### Cross-Cluster Networking

```yaml
apiVersion: cni.cilium.io/v1alpha1
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: cross-cluster-allow
spec:
  endpointSelector:
    matchLabels:
      app: api-server
  ingress:
  - fromEntities:
    - remote-cluster
    - remote-namespace
  toEntities:
  - all
```