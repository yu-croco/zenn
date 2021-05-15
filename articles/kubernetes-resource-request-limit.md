---
title: "Kubernetesのresources管理入門"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes"]
published: true
---

# コレは何？
Kubernetesのリソース管理を触ったので調べたことをまとめました。

# kubernetesでのリソース管理
あるDeploymentで作成されるPodはアクセス負荷が低いため低めのスペックを設定したり、あるCronJobで作成されるPodに対しては大きめのデータを処理するために高いスペックを設定したりといった具合に、個々のPodに適したリソース(CPU, memory)を指定したくなります。
そんなリソース管理のための設定として `resources` があります。

# resources
リソース指定には `limits` と `requests` があります。
`limits` はコンテナに指定できる最低限のリソースのことであり、 `requests` はコンテナに指定できる最大限のリソースのことです。

以下のように `spec.containers[].resources` に指定します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: log-aggregator
    image: images.my-company.example/log-aggregator:v6
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

# resourcesの単位
## CPU
[Kubernetesにおけるリソースの単位 - CPUの意味](https://kubernetes.io/ja/docs/concepts/configuration/manage-resources-containers/#cpu%E3%81%AE%E6%84%8F%E5%91%B3) によると、1 cpuは以下の意味を持ちます。

> Kuberenetesにおける1つのCPUは、クラウドプロバイダーの1 vCPU/コアおよびベアメタルのインテルプロセッサーの1 ハイパースレッドに相当します。

例えば2 vCPUのEC2インスタンスのスペックをフルで使いたい場合には `cpu: 2` と記載します。
1 vCPU分のリソースを割り当てたい場合には `cpu: 1` または `cpu: 1000m` と記載します(Kubernetesでは1cpu = 1000m)。


## Memory
memoryはバイト単位で割当可能である。 `1G` / `1Gi` , `512 M` / `512Mi` のように指定することもできる。

# Podのスケジュール
resourcesを用いてリソースを指定することでkube-schedulerがPodの配置先を選定する際に、nodeのスペックをより効率よく活用できるようPodが配置されます。
より具体的には、resourcesのrequestsに指定されたmemory,cpuの値を元に配置先のnodeを選定します。
requestsとlimitで大きな乖離があるPodがたくさんある場合には個々のPodが最大限利用できるリソースが小さくなり非効率なPod配分となってしまうため、注意が必要です。

# resources.limitsを超えた場合
`resources.limits.memory` の値を超えた場合には、OOM Killerにより該当のPodは強制終了されます。
`resources.limits.cpu` の値を超えた場合にはPodの強制終了は発生しません。

## CPUについて
CPUのリソース制限に関してはスロットリングというものがあるので、kubernetesのメトリクスを見つつ設定していくのが良さそうです。

詳細はこちらが大変参考になりました。

- [Understanding resource limits in kubernetes: cpu time](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)
- [Understanding CPU throttling in Kubernetes to improve application performance](https://speakerdeck.com/daikurosawa/understanding-cpu-throttling-in-kubernetes-to-improve-application-performance-number-k8sjp)
- [Kubernetes No CPU Limit: Save yourself from setting up CPU](https://amixr.io/blog/what-wed-do-to-save-from-the-well-known-k8s-incident/)
- [Resource Quality of Service in Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/resource-qos.md)


# QoS class
Kubernetesはnodeのmemory上限を超えた際にnode上のPodを強制削除します。どのPodを優先的に削除するかはQoS classという仕組みによって決まります。
QoS classには3種類あり、優先度が高いものから削除されていきます。

|  #  |  QoS class  | priority| condition|
| ---- | ---- | --- |--- |
|  1  |  Guaranteed  | 1 | Pod内のすべてのコンテナにmemory,CPUのrequestsとlimitsが設定されており、かつrequestsとlimitsで同じ値であること |
|  2  |  Burstable  | 2 | Guaranteedではなく、1つ以上のrequestsかlimitsが設定されている場合 |
|  3  |  Best-Effort  | 3 | requests, limitsともに未指定の場合 |

PodがどのQoS classであるかを確認したい場合には、 `kubectl describe pod -o yaml` した際に `status.qosClass` に表示されます。
簡単なハンズオンはこちらを参考にしてみて下さい。[PodにQuality of Serviceを設定する](https://kubernetes.io/ja/docs/tasks/configure-pod-container/quality-service-pod/)


# resourcesが未指定の場合
resourcesを指定しない場合、Podは使用できる限りのリソース（≒ nodeのリソース）を利用します。
ただし `LimitRange` を指定している場合にはそれが適用されます。


# ResourceQuota
ResourceQuotaを使うことでnamespace単位で利用可能な総リソースを設定できます（Podなどの個々のリソースではなく特定のnamespaceに対するリソースの総量を制限する）。
*詳細は[リソースクォータ](https://kubernetes.io/ja/docs/concepts/policy/resource-quotas/)を参照して下さい。

# Limit Range
namespace単位でのPod/containerのcpu/memoryやpvcのstorageのリソース制限を設定できます。
*詳細は[Limit Range](https://kubernetes.io/ja/docs/concepts/policy/limit-range/)を参照して下さい。

# 雑感
resourcesの設定はkubernetesの出すメトリクスを見つつ設定（かつ、運用を通して見直していく）していくのが良さそう。


# 参考
- [Kubernetesのリソースの基本を今度こそ理解する](https://blog.mosuke.tech/entry/2020/03/31/kubernetes-resource/)
- [コンテナのリソース管理](https://kubernetes.io/ja/docs/concepts/configuration/manage-resources-containers/)
- [コンテナおよびPodへのCPUリソースの割り当て](https://kubernetes.io/ja/docs/tasks/configure-pod-container/assign-cpu-resource/)
