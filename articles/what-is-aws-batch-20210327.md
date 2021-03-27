---
title: "AWS Batchに入門してみた"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","AWSBatch"]
published: true
---
# これはなに？
大きめのバッチ処理を動かすためにAWS Batchに入門したので、その際に調べたことをまとめました。主に環境構築周りについてまとめています。
解説の中で出てくるソースコードの全体はこちらにまとめていますので、興味ある方はどうぞ。
[yu-croco/sample_aws_batch_terraform](https://github.com/yu-croco/sample_aws_batch_terraform)

# AWS Batch is なに？
AWS公式の[AWS Batch](https://aws.amazon.com/jp/batch/)によると、

> AWS Batch を使用することにより、開発者、科学者、エンジニアは、数十万件のバッチコンピューティングジョブを AWS で簡単かつ効率的に実行できます。AWS Batch では、コンピューティングリソース (CPU やメモリ最適化インスタンスなど) の最適な数量とタイプを、送信されたバッチジョブの量と具体的なリソース要件に基づいて動的にプロビジョニングします。

というわけで、バッチ用の処理を扱うリソースを提供してくれるようです。
また、

> AWS Batch では、AWS Fargate や Amazon EC2、またスポットインスタンスなどの幅広い AWS コンピューティングサービスおよび機能において、バッチコンピューティングワークロードが計画やスケジュールされ、実行されます。

とのことで、EC2やFargateなどいろいろ選べるようです。

# コンセプト
[What Is AWS Batch?](https://docs.aws.amazon.com/batch/latest/userguide/what-is-batch.html)によると、以下の概念があるようです。

## Jobs
AWS Batchに投げるJobの単位です。shell scriptやDocker container imageなどがそれに当たります。
JobはEC2またはFargate上にコンテナリソースとして起動します。またJobは特定の値を指定することで他のJobを参照することができ、あるJobの成功により別のJobを実行するといったことができます。

## Job Definitions
Jobがどのように起動するかを設定するものです（内部ではECS関連のAPIを動かしているっぽい）。
実行するコンテナに必要なメモリ、CPU、環境変数などの指定ができ、IAM Roleを利用してAWSリソースへのアクセスなどもできます。AWS BatchでECRからDocker pullする場合などにはここで権限を付与する形となります。

## Job Queues
AWS BatchへJobを実行した際、JobはQueueに貯められた上、設定した優先度に応じて処理されます。

## Compute Environment
Job Definitionを実行するための仮想環境を定義します。 EC2/Fargateなどのインスタンスタイプ（c5.2xlargeなど）やvCPUなどのリソースの下限/上限などを設定します。


# Terraformで構成してみる
Terraform公式のサンプルを拝借しつつ構成を見てみます。
*subnetなどのネットワーク周りのことは丸っと省略しているので、気になる方は[yu-croco/sample_aws_batch_terraform](https://github.com/yu-croco/sample_aws_batch_terraform)を覗いてみてください。個人的にはAWS Batchそのもののリソース定義よりも、ネットワーク周りの準備でハマりました..

## job definition周り

```terraform
locals {
  job_definition = {
    command = ["echo", "hello world"],
    image = "alpine:latest",
    memory = 1024,
    vcpus = 1,
    "jobRoleArn": aws_iam_role.job_definition.arn,
  }
}

resource "aws_iam_role" "job_definition" {
  name = "aws-batch-job-role-${var.batch_name}"
  assume_role_policy = data.aws_iam_policy_document.assume_to_aws_batch.json
}

data "aws_iam_policy_document" "assume_to_aws_batch" {
  statement {
    effect = "Allow"
    actions = [
      "sts:AssumeRole",
    ]
    principals {
      type = "Service"
      identifiers = [
        "ecs-tasks.amazonaws.com",
      ]
    }
  }
}

// ECRからPullするための権限もろもろ
resource "aws_iam_role_policy_attachment" "ecs_task_AmazonEC2ContainerRegistryReadOnly" {
  role       = aws_iam_role.job_definition.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

// terraform applyの度に差分が出てversion更新されるので、運用のベスプラはちょっとわからない..
resource "aws_batch_job_definition" "job_definition" {
  name = "aws-batch-job-definition"
  type = "container"
  container_properties = data.template_file.job-definition.rendered
}

data "template_file" "job-definition" {
  template = jsonencode(local.job_definition)
}
```

## compute environment周り

```terraform
/*
 note: 仕様上、一度作成するとAMIが固定される。最新のAMIを適応するためにはコンピューティング環境を作り直す必要がある。
 https://docs.aws.amazon.com/ja_jp/batch/latest/userguide/compute_environments.html#managed_compute_environments
*/
resource "aws_batch_compute_environment" "aws-batch-computing-environment" {
  compute_environment_name = "aws-batch-compute-env"
  service_role = aws_iam_role.aws-batch-service-role.arn
  type = "MANAGED"
  state = "ENABLED"

  // 仕様上cpuとサービスロール以外はupdateができない
  // 指定値を変えるとaws_batch_compute_environmentがreplaceされる
  compute_resources {
    // 他にも色々ある https://docs.aws.amazon.com/ja_jp/batch/latest/userguide/allocation-strategies.html
    allocation_strategy = "BEST_FIT"
    instance_role = aws_iam_instance_profile.aws-batch-instance-role.arn
    instance_type = ["c4.large"]
    max_vcpus = 256
    min_vcpus = 0
    security_group_ids = [aws_security_group.aws_batch.id]
    subnets = var.subnets
    type = "EC2"
  }

  // 削除時に先にroleが削除され、コンピューティング環境の削除でスタックする。根本的な対処はちょっと謎..
  // https://github.com/hashicorp/terraform-provider-aws/issues/8549
  depends_on = [aws_iam_role.aws-batch-service-role]

  // ジョブキューはコンピューティング環境を最低限一つは指定する必要があるため、
  // replaceになる場合は `create_before_destroy` である必要がある
  lifecycle {
    create_before_destroy = true
  }
}

// AWS Batchのインスタンスが使用する
resource "aws_iam_role" "aws-batch-instance-role" {
  name = "${var.name}-aws-batch-instance-role"
  force_detach_policies = true
  assume_role_policy = data.aws_iam_policy_document.instance-assume-role.json
}

resource "aws_iam_role_policy_attachment" "aws-batch-instance-role" {
  role = aws_iam_role.aws-batch-instance-role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
}

resource "aws_iam_instance_profile" "aws-batch-instance-role" {
  name = "${var.name}-aws-batch-instance-profile"
  role = aws_iam_role.aws-batch-instance-role.name
}

data "aws_iam_policy_document" "instance-assume-role" {
  statement {
    actions = [
      "sts:AssumeRole"
    ]

    effect = "Allow"

    principals {
      type = "Service"
      identifiers = [
        "ec2.amazonaws.com",
      ]
    }
  }
}

// AWS Batchそのものが使用する
resource "aws_iam_role" "aws-batch-service-role" {
  name = "${var.name}-aws-batch-service-role"
  // ロールを破棄する前にロールが持つポリシーを強制に切り離す
  force_detach_policies = true
  assume_role_policy = data.aws_iam_policy_document.service-assume-role.json
}

resource "aws_iam_role_policy_attachment" "aws-batch-service-role" {
  role = aws_iam_role.aws-batch-service-role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBatchServiceRole"
}

data "aws_iam_policy_document" "service-assume-role" {
  statement {
    actions = [
      "sts:AssumeRole",
    ]

    effect = "Allow"

    principals {
      type = "Service"
      identifiers = [
        "batch.amazonaws.com",
      ]
    }
  }
}
```

## job queue周り

```terraform
resource "aws_batch_job_queue" "this" {
  name                 = var.name
  state                = "ENABLED"
  // 同じコンピューティング環境に複数job queueがある場合には、priorityの値が高いほうがより優先的に処理される
  priority             = 1
  compute_environments = var.compute_environment_arns
}
```

# ハマリポイント
## jobが `RUNNABLE` でスタックする
調べるとaws batchの中で最も多いっぽいハマりポイントのようです。
原因となる事項は[Jobs Stuck in RUNNABLE Status](https://docs.aws.amazon.com/batch/latest/userguide/troubleshooting.html#job_stuck_in_runnable)にまとまっていますが、複数の可能性があるようでデバッグが結構メンドイそう..。
今回は自分がハマった点を以下にまとめておきます。

*また以下も参考になると思います。
- [AWS Batch ジョブが RUNNABLE ステータスで止まっているのはなぜですか?](https://aws.amazon.com/jp/premiumsupport/knowledge-center/batch-job-stuck-runnable-status/)
- [Troubleshooting AWS Batch](https://docs.aws.amazon.com/batch/latest/userguide/troubleshooting.html#job_stuck_in_runnable)

### Compute環境からECS service endpointへアクセスできてない
以下の感じでコンピューティング環境からECS service endpointへアクセスが必要になるらしい。

> **No internet access for compute resources**
>  Compute resources need access to communicate with the Amazon ECS service endpoint. This can be through an interface VPC endpoint or through your compute resources having public IP addresses.
>  For more information about interface VPC endpoints, see Amazon ECS Interface VPC Endpoints (AWS PrivateLink) in the Amazon Elastic Container Service Developer Guide.
>  If you do not have an interface VPC endpoint configured and your compute resources do not have public IP addresses, then they must use network address translation (NAT) to provide this access. For more information, see NAT Gateways in the Amazon VPC User Guide. For more information, see Tutorial: Creating a VPC with Public and Private Subnets for Your Compute Environments.

対処法としては以下の3種類。
個人的には、`2. ECS Interface VPC Endpoints (AWS PrivateLink)を使用する` が良い気がしています。ECRなども使う場合にはそちらもPrivateLink用の設定をするのが良いと思うので、こちらと合わせてやってしまうのが楽かなと思います。

#### 1. コンピューティング環境が配置されているsubnetでパブリック IPv4 アドレスの自動割り当てを許可する
[AWS Batch ジョブが RUNNABLE ステータスで止まっているのはなぜですか?](https://aws.amazon.com/jp/premiumsupport/knowledge-center/batch-job-stuck-runnable-status/) にある以下の説明箇所がそれに該当するかと思います。

> 6. コンピューティング環境のサブネットごとに、[説明] を選択し、[パブリック IPv4 アドレスの自動割り当て] プロパティの値を確認します。

#### 2. ECS Interface VPC Endpoints (AWS PrivateLink)を使用する
[Amazon ECS interface VPC endpoints (AWS PrivateLink)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/vpc-endpoints.html)を参考に、以下3つのVPC Endpoint(PrivateLink)を建てると、Internetを経由せずにAWS内部の通信でECS service endpointへアクセスが可能となります。

- com.amazonaws.region.ecs-agent
- com.amazonaws.region.ecs-telemetry
- com.amazonaws.region.ecs

#### 3. Nat Gatewayを使用する
[Tutorial: Creating a VPC with Public and Private Subnets for Your Compute Environments](https://docs.aws.amazon.com/batch/latest/userguide/create-public-private-vpc.html)を参考にNAT Gateway経由でECS service endpointへアクセスする感じだそうです。
結局はNAT Gatewayを置くだけではなく、public subnetで `auto-assigned public IPv4 addresses` をenableにしないといけない感じっぽい（ちゃんと調べてない）？？

# その他TIPS
- コンピューティング環境のインスタンスタイプはあとから変えられない
  - 変更する際にはreplaceで新しいものが立ち上がる。job queueはcompute_environmentsに依存するため、compute_environments側で `create_before_destroy = true` のlifecycleを設定するのが無難そう
  - [[AWS Batch]Terraformでコンピューティング環境のインスタンスタイプを後から変更する苦肉の策](https://n-s.tokyo/2019/03/awsbatch-change-instance-type/)

# 雑感
- AWS Batchの機能そのものはすごいけど、リソース準備が大変..
  - 内部で他のサービスを使っていることもあり、一通り動くようにするために準備するリソースが多い
  - 小さなデバッグが難しい（全部用意してから始めて問題が見つかる..）
- public subnet前提の記事ばかりで、`RUNNABLE`でのハマり解消に苦労した..
- 細かいところで癖がある

# 参考
- [Resource: aws_batch_compute_environment](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/batch_compute_environment)
- [Resource: aws_batch_job_definition](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/batch_job_definition)
- [Resource: aws_batch_job_queue](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/batch_job_queue)
- [AWS再入門ブログリレー AWS Batch編](https://dev.classmethod.jp/articles/relay_looking_back_aws-batch/)
- [AWS Batch とは何か](https://qiita.com/pottava/items/d9886b2e8835c5c0d30f)
- [Terraform, AWS Batch, and AWS EFS](https://medium.com/swlh/terraform-aws-batch-and-aws-efs-8682c112d742)
- [Jobs Stuck in RUNNABLE Status](https://docs.aws.amazon.com/batch/latest/userguide/troubleshooting.html#job_stuck_in_runnable)