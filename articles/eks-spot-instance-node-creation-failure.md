---
title: "EKS Managed Node GroupでSpot Instanceを使用した際にNodeCreationFailureとなるケースがある"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["EKS","SpotInstance"]
published: false
---

# これは何？
EKS Managed Node GroupでSpot Instanceを利用する際に、時々NodeCreationFailureとなるケースがあるので調査した際のメモです。

# 前提
- On-DemandのEC2 Instanceでは発生しない
- Spot Instanceで使用したいInstance Familyを1つだけ指定すると発生しない
    - Spot Instanceの空きがない場合のエラーを除く

# 事象
Terraformで `aws_eks_node_group` をSpot Instanceとして起動しようとした際に、以下のエラーが出ることがある。

```
Error: error waiting for EKS Node Group (<your-cluster>:<your-node-group>) creation: NodeCreationFailure: Instances failed to join the kubernetes cluster. Resource IDs: [i-xxxxxxxx i-xxxxxxxx i-xxxxxxxx ]
```

- 同時に3つのManaged Node Groupを起動しようとしている
- Spot Instanceで指定しているInstance Familyは5個くらい指定している
    - AWS公式でも可用性を考慮するとできる限り広く指定しておくのが良いというため

# エラー内容の直接原因
公式（ [Amazon EKS troubleshooting](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting.html) ）には以下のように書いてある。

> NodeCreationFailure: Your launched instances are unable to register with your Amazon EKS cluster. Common causes of this failure are insufficient node IAM role permissions or lack of outbound internet access for the nodes. Your nodes must be able to access the internet using a public IP address to function properly. For more information, see VPC IP addressing. Your nodes must also have ports open to the internet. For more information, see Amazon EKS security group considerations.

前提でも記載したとおり、On-DemandのEC2 Instanceでは発生していないので、設定値（IAM Role権限もIP設定周りも）自体が問題である可能性は非常に低そう。

Spot Instance特有のなにかによるものだが、コレばっかりは内部実装のため謎が多い。
