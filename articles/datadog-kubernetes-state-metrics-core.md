---
title: "DatadogのKubernetes State Metrics Coreに関する簡単なまとめ"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes","Datadog","kubeStateMetrics"]
published: true
---

# コレは何？
KubernetesのメトリクスをDatadogに流しているのだが、 一部のメトリクス（`kubernetes_state.container.cpu_requested`など）がDatadogで取得できていないため調べたら `Kubernetes State Metrics Core` にたどり着いたのでそれに関する簡単なまとめ。

# Kubernetes State Metrics Core is 何？
[Kubernetes State Metrics Core](https://docs.datadoghq.com/integrations/kubernetes_state_core/?tab=helm)によると、

> The Kubernetes State Metrics Core check leverages kube-state-metrics version 2+ and includes major performance and tagging improvements compared to the legacy kubernetes_state check.

とのこと。
要はkube-state-metricsのv2.0のことをDatadogでは `Kubernetes State Metrics Core` と呼んでいるらしい（とても紛らわしい...）。

私の検証環境ではこの `Kubernetes State Metrics Core` がenableになっていなかったため、一部のメトリクスが取得できていないようだった。

kube-state-metrics v2に関しては以下が参考になった。
- [kube-state-metrics goes v2.0](https://kubernetes.io/blog/2021/04/13/kube-state-metrics-v-2-0/)
- [kube-state-metrics v2.0.0 の変更点調査](https://qiita.com/yosshi_/items/59f8cdaff3368788529a)

# セットアップ方法
詳細は[Monitor kube-state-metrics v2.0 with Datadog](https://www.datadoghq.com/ja/blog/kube-state-metrics-v2-monitoring-datadog/)や [Setup](https://docs.datadoghq.com/integrations/kubernetes_state_core/?tab=helm#setup)に載っているが、
[DataDog/helm-charts](https://github.com/DataDog/helm-charts) 経由で入れる場合には、以下のように設定値をちょこっと変えるだけで対応できる。
基本的にはv1.0系のものと同じメトリクス名でデータが取得できるが一部は名称が変わったりしているらしい。

```yaml:values.yaml
datadog:
  # これでv2.0をenableにできる
  kubeStateMetricsCore:
    enabled: true
  # v1.0はもう不要なのでdisableして良さそう
  kubeStateMetricsEnabled: false
```

以上
