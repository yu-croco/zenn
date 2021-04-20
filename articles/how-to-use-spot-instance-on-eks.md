---
title: "EKS Managed Node GroupでSpot Instanceを使うために調べたことまとめ"
emoji: "🦾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["EKS", "AWS","Terraform"]
published: true
---

# これは何？
EKS Managed Node Groupに対してSpot Instanceを使いたくなって調べた際のまとめです。

# 経緯
EKS Managed Node GroupでSpot Instanceが解禁されてたので導入したい。
[Amazon EKS が、マネージド型ノードグループでの EC2 スポットインスタンスのプロビジョニングと管理をサポート](https://aws.amazon.com/jp/blogs/news/amazon-eks-now-supports-provisioning-and-managing-ec2-spot-instances-in-managed-node-groups-jp/)に記載されている通り、EKSのworker nodeでもSpot Instanceが使用できるようになり、Spot Instanceを使いたいなとなった。

# Spot Instanceの仕様概説
EKS Managed Node Groupに使用するSpot Instanceの仕様はざっくり以下の感じです。

- Instanceの割り当て戦略は `capacity-optimized` が適応されている
    - 空きプールにたくさんあるInstanceサイズのものから優先的に起動される
        - これによりInstanceが入れ替わるリスクを減らせる（らしい）
    - より余ってるInstanceタイプなので、値段も安定している（はず）
    - 参考
        - [Introducing the capacity-optimized allocation strategy for Amazon EC2 Spot Instances](https://aws.amazon.com/jp/blogs/compute/introducing-the-capacity-optimized-allocation-strategy-for-amazon-ec2-spot-instances/)
- `Capacity Rebalancing` により、Spot Instanceが中断されるリスクを早期検知してInstanceの入れ替えを行う
    - 基本的にはGraceful shutdownかつ、今動いてるinstanceがterminateされる前に新しいinstanceがreadyになる
    - ただし必ず保証されているわけではなく、場合によっては（on-demanで要求されるインスタンス数が爆上がりするなど?）EC2のterminate 2分前通知と同じタイミングで実行されることもあり、代わりのnodeがreadyになる前に既存のnodeがterminateされる可能性もある
    - 参考
        - [Managed node group capacity types](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html#managed-node-group-capacity-types)
        - [EC2 instance rebalance recommendations](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/rebalance-recommendations.html)
- Spot Instanceには `eks.amazonaws.com/capacityType=SPOT` というラベルが付与されている
    - Affinity, AntiAffinityなどに使える
- 最も遅い場合でterminate 2分前にシステムの停止作業が行われる
    - EC2にシグナルが送られ、停止へと向かう

# Spot Instanceを使う上での注意点
- いつEC2がTerminateされてもいいように、フォールトトレラントのアプリケーションに対して使用すること
    - 最悪のケースではSpot Instanceがterminateされる2分前にterminateの準備が始まる
    - DaemonSetや監視ツール用のnodeに対しては推奨されてない
- アプリケーションの可用性を高めるために、マネージド型ノードグループに複数のインスタンスタイプを含めること
    - 特定のインスタンスタイプが必ず空いているという保証はない
    - ほぼ同量の vCPU とメモリを持つ全てのインスタンスタイプを全て指定するのが推奨されている
        - 例）m5.xlarge, m4.xlarge, m5a.xlarge, m5d.xlarge, …
- 1つのNode Groupに対してはOn-Demand or Spot Instanceのどちらかのみ指定できる
    - 切り替える場合にはNode Groupが作り直される
- Spot Instanceのmax料金設定はできない
    - Managed Node Groupに対する指定は未対応の模様（Terraformでの挙動を見る限り）
        - その他、普通のlaunch templateでは指定できるものができなかったりする

# Terraformでの適応方法
基本的には `aws_eks_node_group` の `capacity_type` を `SPOT` にするだけでSpot Instanceをつかえるようになります。
[Resource: aws_eks_node_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_node_group)にも以下のように記載があります。
> capacity_type - (Optional) Type of capacity associated with the EKS Node Group. Valid values: ON_DEMAND, SPOT. Terraform will only perform drift detection if a configuration value is provided.

ただし `aws_launch_template` と `aws_eks_node_group` を併用した場合、設定値に制約があるのでapplyして確認するのが安全そう。
例えば `capacity_type=SPOT` を指定すると、 `aws_launch_template` の `instance_market_options` が使用できない。

```terraform
variable "is_prod" { type = bool }

resource "aws_launch_template" "eks_worker_template" {
  name     = "launch_template"

  # Spot Instanceを利用する場合にはaws_eks_node_group側でinstance typeを指定する
  instance_type  = var.is_prod ? "t3.medium" : ""

  # いろいろ設定が続く...
}

resource "aws_eks_node_group" "eks_worker" {
  cluster_name = "eks-cluster-name"
  node_group_name = "node-group-name"

  capacity_type = var.is_prod ? "ON_DEMAND" : "SPOT"
  instance_types = local.is_prod ? [] : ["t3.medium", "t2.medium"]

  launch_template {
    id      = aws_launch_template.eks_worker_template.id
    version = aws_launch_template.eks_worker_template.latest_version
  }

  # いろいろ設定が続く...
}
```

# 雑感
- 設定1つでSpot Instanceに切り替えられるのはとても便利
- 便利すぎて仕様を理解してないと何かあった際に障害に繋がりそう

# 参考
## 概念系
- [Amazon EKS が、マネージド型ノードグループでの EC2 スポットインスタンスのプロビジョニングと管理をサポート](https://aws.amazon.com/jp/blogs/news/amazon-eks-now-supports-provisioning-and-managing-ec2-spot-instances-in-managed-node-groups-jp/)
- [Amazon EC2 スポットインスタンスを活用したウェブアプリケーションの構築](https://aws.amazon.com/jp/blogs/news/running-web-applications-on-amazon-ec2-spot-instances/)
- [Capacity-Optimized Spot Instance Allocation in Action at Mobileye and Skyscanner](https://aws.amazon.com/blogs/aws/capacity-optimized-spot-instance-allocation-in-action-at-mobileye-and-skyscanner/#:~:text=capacity%2Doptimized%20%E2%80%93%20Allocates%20instances%20from,a%20higher%20cost%20of%20interruption.)
- [EKS Managed Node GroupでSpot Instancesを使う](https://int128.hatenablog.com/entry/2020/12/03/100853)
- [Introducing the capacity-optimized allocation strategy for Amazon EC2 Spot Instances](https://aws.amazon.com/jp/blogs/compute/introducing-the-capacity-optimized-allocation-strategy-for-amazon-ec2-spot-instances/)
- [Managed node groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html#managed-node-group-capacity-types)

## Terraforem実装系
- [cloudposse/terraform-aws-eks-node-group](https://github.com/cloudposse/terraform-aws-eks-node-group)
