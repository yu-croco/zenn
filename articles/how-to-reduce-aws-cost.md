---
title: "AWSで検証環境のコストカットに使えそうなTIPS集"
emoji: "💳"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["AWS"]
published: false
---

# これはなに？
検証環境のAWS運用費を削減するのに使えそうなポイントをまとめています。

# 前提
この記事では主に検証環境でのコスト削減に関して記載しているため、本番環境で使用するには好ましくないもの（可用性などは考慮しなくていい前提のもの）が多数あります。

# 現状把握
推測するな計測せよ、というわけで何はともあれ現状を見るべし。
具体的には下記の順序で画面遷移をすると、AWSの各種サービス別の月額費用がわかる。
どこにどれだけかかっているかは運営するサービスの性質にによって様々なので、ひとまずこれで現状を見るのが良さげ。

- Billingの請求書
    - 個々のサービスの中で更にどのような部分で費用が掛かっているのかが分かる
- Cost Explorer
    - どのサービスに幾ら掛かってるかの全体像を理解しやすい

# 固定費削減のTIPS
## 1. Reserved Instance（RI）
王道すぎるので仕様は割愛
公式などを参照ください
- [Amazon EC2 リザーブドインスタンス](http://localhost:8000/articles/how-to-reduce-cost-of-aws)
- [Amazon RDS リザーブドインスタンス](https://aws.amazon.com/jp/rds/reserved-instances/)

## 2. Savings Plans
王道すぎるので仕様は割愛
公式などを参照ください
- [Savings Plans の料金](https://aws.amazon.com/jp/savingsplans/pricing/)
- [EC2やFargateが最大72%割引となる新しい料金モデル「Savings Plans」がリリースされました](https://dev.classmethod.jp/articles/new-savings-plans-for-compute/)

## 3. Spot Instance
王道すぎるので仕様は割愛
公式などを参照ください
最近はEC2だけでなくFargateにも対応しているので是非
- [Amazon EC2 スポットインスタンス](https://aws.amazon.com/jp/ec2/spot/?cards.sort-by=item.additionalFields.startDateTime&cards.sort-order=asc)
- [AWS Fargate Spotの発表 – Fargateとスポットインスタンスの統合](https://aws.amazon.com/jp/blogs/news/aws-fargate-spot-now-generally-available/)
- [Fargateをスポットで7割引で使うFargate Spotとは？ #reinvent](https://dev.classmethod.jp/articles/fargate-spot-detail/)

## 4. NAT Gateway
NAT Gatewayはリソース起動に対する料金も処理データに対する課金も結構高いです。 特にECSタスクなどでECRからコンテナイメージをPullする際にNAT Gatewayを経由すると非常に高く付くことがあります。

### 4-1. NAT GatewayをSingle AZで運用する
例えばAZ 1aにNAT Gatewayを配置してルートテーブルで1a,1cのAZのsubnetに紐付ける。こうすることで単純にNAT Gatewayの起動数を1/2にできる。

- 変更前のざっくりイメージ
![](https://storage.googleapis.com/zenn-user-upload/8kq95fs95r0ftnrustod3caoaqac)

- 変更後のざっくりイメージ
![](https://storage.googleapis.com/zenn-user-upload/cqeh0eohyorajh2onwzp96xpivn8)


### 4-2. PrivateLinkを使用する
ECRへの接続などをPrivateLink経由にすることで固定費を削減できる。

[Amazon VPC の料金](https://aws.amazon.com/jp/vpc/pricing/) によると、NAT GatewayとPrivateLinkでは4.4倍の費用差がある。

|  #  |  1リソースあたりの料金  |  処理データ1GBあたりの料金  |注意点|
| ---- | ---- | ---- | ---- |
|  NAT Gateway  |  $0.062/hour  |  $0.062/hour  |  -  |
|  PrivateLink  |  $0.014/hour  |  $0.01 (インターフェース型のPrivateLinkは$0.0035)  |  配備するAZ単位で課金が発生する  |

詳しい手順は公式を参照してください。[Amazon ECR インターフェイス VPC エンドポイント (AWS PrivateLink)](https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/vpc-endpoints.html)

## 5. PrivateLink
`4. NAT Gateway` で記載したとおりNAT Gatewayを使用するほうが固定費を抑えられる可能性が高いが、PrivateLinkは配備しているAZ単位で費用がかかること、また例えばECRに対してPrivateにアクセスできるようにするためにはいくつかの種類のPrivateLinkを用意する必要がある。
よってチリツモでPrivateLinkでもそこそも固定費が掛かる可能性がある。

そのため、NAT Gatewayと同様にPrivateLinkも検証環境ではSingle AZで運用しても良さそう。

# 参考
- [Amazon EC2 リザーブドインスタンス](https://aws.amazon.com/jp/ec2/pricing/reserved-instances/)
- [EC2やFargateが最大72%割引となる新しい料金モデル「Savings Plans」がリリースされました](https://dev.classmethod.jp/cloud/aws/new-savings-plans-for-compute/)
