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
