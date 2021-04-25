---
title: "[🍊]EKS MNGでSpot Instanceを使用してNodeCreationFailureとなった際の調査メモ"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["EKS","SpotInstance"]
published: true
---

# これは何？
TerraformでEKS Managed Node GroupでSpot Instanceを利用する際に、時々 `NodeCreationFailure` となるケースがあるので調査した際のメモです。

# 結論
今のところ、Spot Instanceの内部挙動と関係しているようで根本原因は分かっていません(そのためタイトルに🍊を添えてます..)。。
詳しい方いらっしゃいましたら、ご指摘お願いします...。

# 前提
- On-DemandのEC2 Instanceでは発生しない
- 1つのsubnetに複数のMNGが配置されている
- subnetのCIDRブロックは `/24` なので251ipある

# 事象
Terraformで `aws_eks_node_group` をSpot Instanceとして起動しようとした際に、以下のエラーが出ることがある。

```
Error: error waiting for EKS Node Group (<your-cluster>:<your-node-group>) creation: NodeCreationFailure: Instances failed to join the kubernetes cluster. Resource IDs: [i-xxxxxxxx i-xxxxxxxx i-xxxxxxxx ]
```

もう少し補足すると、以下のような感じである。
- `for_each` を使って同時に複数のManaged Node Groupを起動しようとしている
- `instance_types` には10個くらい指定している
    - AWS公式でも可用性を考慮するとできる限り広く指定しておくのが良いというためひとまず突っ込んでおいた

# エラー内容の直接原因
公式（ [Amazon EKS troubleshooting](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting.html) ）には以下のように書いてある。

> NodeCreationFailure: Your launched instances are unable to register with your Amazon EKS cluster. Common causes of this failure are insufficient node IAM role permissions or lack of outbound internet access for the nodes. Your nodes must be able to access the internet using a public IP address to function properly. For more information, see VPC IP addressing. Your nodes must also have ports open to the internet. For more information, see Amazon EKS security group considerations.

前提でも記載したとおり、On-DemandのEC2 Instanceで起動した際には発生していないので、設定値（IAM Role権限もIP設定周りも）自体が問題である可能性は非常に低そう。

Spot Instance特有の何かが原因のように思われるが、コレばっかりはSpot Instanceの内部の挙動を知る必要があるため無理そう。

# 試したこと
## 1. Spot Instanceで使用したい `instance_types` を減らす
特に根拠はないが...、「候補が多すぎるため、一度に複数のインスタンスが起動して選出されようとしていてネットワーク周りの何かを逼迫させているのかな？」と思い、 おもむろに `instance_types` に1つだけインスタンスタイプを指定してみた。
すると、普通にapplyが通った（どうして...）。。。

何回か繰り返し動作確認したところ、基本的には正常に動作する。

が、複数追加するとまたしても同じエラーが発生する。
やはりSpot Instance起動時になにかあるようだ。。

# 雑感

AWSナニモワカラン...
