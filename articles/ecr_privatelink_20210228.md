---
title: "同じAZ内の複数subnetからECR PrivateLinkを利用する方法"
emoji: "🔌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "ECR", "PrivateLink"]
published: true
---

# これは何？
同じVPC内で同じAZ内にある複数のsubnetからECR PrivateLinkを使う方法をまとめた記事です。
ソースコードは以下にアップしていますので、興味ある方は御覧ください。
[yu-croco/ecr_privatelink_tf_sample](https://github.com/yu-croco/ecr_privatelink_tf_sample)


# ECR PrivateLinkとは
ECSなどからECRからコンテナイメージをpullする際、リソースからECRへは基本的にインターネット経由でのアクセスとなります。ECRを利用する側のsubnet(だいたいはprivate subnet?)にNAT Gatewayなどを配置して、そこから取得するのが一般的ではないでしょうか。

この際に以下のような問題が出てくると思うのですが、これらを解決してくれるのが `PrivateLink` です。必要なVPC Endpointを配置することで、ECRからprivate通信（AWS内部の通信）でコンテナイメージをpullできます。

- セキュリティ的にちょっと不安だよね
    - できればAWS内部の通信で閉じていたいよね
- NAT Gateway高いよね..


詳細は公式の [Amazon ECR インターフェイス VPC エンドポイント (AWS PrivateLink)](https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/vpc-endpoints.html) を参照ください。
またNAT Gatewayの費用周りについてはこちらが参考になりました。 [そのトラフィック、NATゲートウェイを通す必要ありますか？適切な経路で不要なデータ処理料金は削減しましょう](https://dev.classmethod.jp/articles/reduce-unnecessary-costs-for-nat-gateway/)


# 問題点
公式ドキュメントなどを見つつ必要なリソースを建てていく際にInterfaceタイプのVPC Endpoint（いわゆるPrivateLink）の仕様上、以下の問題点に直面しました。

- 1つのAZにつき1つまでしか配備できない
    - そのため、AZ1のsubnet1とsubnet2に同時に配備することができない
- 1つのAZにつき1つまでしか配備できない
    - そのため、「AZ1のsubnet1とsubnet2のそれぞれに対してVPC Endpointを作成する」といったことはできない

私の場合、 `同じAZに複数subnetあり、それらの上に乗っているアプリケーションからECRのPrivateLinkを利用したい` といったケースだったため、上記の仕様では一見無理ゲーなのではないかとおもいました...。


# 解決策
自分で検証したりAWSサポートに質問した結果として、 `同じVPCのルーティング可能なsubnetであればどこに配置しても大丈夫` という感じでした。 そのためInterfaceタイプのVPC Endpoint用のsubnetを作り、そこにInterfaceタイプのVPC Endpointを配置するようにしました。
*ルーティングは暗黙的に配置先のVPC内に紐付いているため特に設定などはしなくて良い認識（動作確認した範囲でも大丈夫そう）。

# 構成
構成としてはざっくり以下のような感じでいけるかと思います。
![構成](https://storage.googleapis.com/zenn-user-upload/9o8bone933v537dgzibbaygktn7s)

# サンプルTerraform
VPC Endpoint周りだけピックアップしてTerraformで書くと以下のような感じになるかなと思います。
詳細は[こちら](https://github.com/yu-croco/ecr_privatelink_tf_sample)にも掲載していますので、興味有る方はどうぞ。

```terraform
resource "aws_subnet" "private_1a_vpc_endpoint" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.3.0/24"
  availability_zone = local.az_a
  tags = {
    Name = "private 1a vpc endpoint"
  }
}

resource "aws_subnet" "private_1c_vpc_endpoint" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.6.0/24"
  availability_zone = local.az_c
  tags = {
    Name = "private 1c vpc endpoint"
  }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "VPC Endpoint S3"
  }
}

// アプリケーションが乗っている各subnetと紐付ける
resource "aws_route_table_association" "private" {
  for_each = [
    aws_subnet.private_1a_app1.id,
    aws_subnet.private_1a_app2.id,
    aws_subnet.private_1c_app1.id,
    aws_subnet.private_1c_app2.id,
  ]

  subnet_id      = each.key
  route_table_id = aws_route_table.private.id
}

resource "aws_security_group" "vpc_endpoint" {
  name   = "vpc_endpoint_sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }
}

// S3はGatewayタイプなのでroute tableに紐づく
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]
}

// InterfaceタイプのVPC Endpointはそれ用のsubnetに紐付ける
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  security_group_ids  = [aws_security_group.vpc_endpoint.id]
  subnet_ids          = [aws_subnet.private_1a_vpc_endpoint.id, aws_subnet.private_1c_vpc_endpoint.id]
}

// EC2タイプの場合に必要
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  security_group_ids  = [aws_security_group.vpc_endpoint.id]
  subnet_ids = [aws_subnet.private_1a_vpc_endpoint.id, aws_subnet.private_1c_vpc_endpoint.id]
}

// Fargateの場合に必要
resource "aws_vpc_endpoint" "logs" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.logs"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  security_group_ids  = [aws_security_group.vpc_endpoint.id]
  subnet_ids = [aws_subnet.private_1a_vpc_endpoint.id, aws_subnet.private_1c_vpc_endpoint.id]
}
```

# 雑感
ネットワークナニモワカラヌ...

# 参考
- [エンドポイントを使用してプライベートサブネットでECSを使用する](https://dev.classmethod.jp/articles/privatesubnet_ecs/)
- [Amazon ECR インターフェイス VPC エンドポイント (AWS PrivateLink)](https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/vpc-endpoints.html)
