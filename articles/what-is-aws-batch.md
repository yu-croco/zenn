---
title: "AWS Batchに入門してみた"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","AWSBatch"]
published: false
---
# これはなに？
AWS Batchに入門したので、その際に調べたことをまとめました。

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
AWS Batchに投げるJobの単位であり、shell scriptやDocker container imageなどがそれに当たる。
JobはEC2またはFargate上にコンテナリソースとして起動する。
Jobは名前やIDを指定することで他のJobを参照することができ、あるJobの成功により別のJobを実行するといったことができる。

## Job Definitions
Jobがどのように起動するかを設定するものであり、Jobのブループリントのようなものである。
メモリ、CPU、環境変数などの指定ができる。
IAM Roleを利用してAWSリソースへのアクセスができる。

## Job Queues
AWS BatchへJobを実行した際、JobはQueueにためられ、設定した優先度に応じて処理される。

## Compute Environment
EC2/Fargateなどのインスタンスタイプ（c5.2xlargeなど）やvCPUなどのリソースの下限/上限などを設定する。

# Terraformで構成してみる
Terraform公式のサンプルを拝借しつつ構成を見てみよう。

- [Resource: aws_batch_compute_environment](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/batch_compute_environment)
- [Resource: aws_batch_job_definition](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/batch_job_definition)
- [Resource: aws_batch_job_queue](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/batch_job_queue)

```terraform
resource "aws_iam_role" "ecs_instance_role" {
  name = "ecs_instance_role"

  assume_role_policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
    {
        "Action": "sts:AssumeRole",
        "Effect": "Allow",
        "Principal": {
        "Service": "ec2.amazonaws.com"
        }
    }
    ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "ecs_instance_role" {
  role       = aws_iam_role.ecs_instance_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
}

resource "aws_iam_instance_profile" "ecs_instance_role" {
  name = "ecs_instance_role"
  role = aws_iam_role.ecs_instance_role.name
}

resource "aws_iam_role" "aws_batch_service_role" {
  name = "aws_batch_service_role"

  assume_role_policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
    {
        "Action": "sts:AssumeRole",
        "Effect": "Allow",
        "Principal": {
        "Service": "batch.amazonaws.com"
        }
    }
    ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "aws_batch_service_role" {
  role       = aws_iam_role.aws_batch_service_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBatchServiceRole"
}

resource "aws_security_group" "sample" {
  name = "aws_batch_compute_environment_security_group"

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_vpc" "sample" {
  cidr_block = "10.1.0.0/16"
}

resource "aws_subnet" "sample" {
  vpc_id     = aws_vpc.sample.id
  cidr_block = "10.1.1.0/24"
}

// Compute Environmentの定義
resource "aws_batch_compute_environment" "sample" {
  compute_environment_name = "sample"

  compute_resources {
    instance_role = aws_iam_instance_profile.ecs_instance_role.arn

    instance_type = [
      "c4.large",
    ]

    max_vcpus = 16
    min_vcpus = 0

    security_group_ids = [
      aws_security_group.sample.id,
    ]

    subnets = [
      aws_subnet.sample.id,
    ]

    type = "EC2"
  }

  service_role = aws_iam_role.aws_batch_service_role.arn
  type         = "MANAGED"
  depends_on   = [aws_iam_role_policy_attachment.aws_batch_service_role]
}

// Job Definitionの定義
resource "aws_batch_job_definition" "sample" {
  name = "tf_test_batch_job_definition"
  type = "container"
  // batchで動かすコンテナのimageなどを指定
  container_properties = <<CONTAINER_PROPERTIES
{
    "command": ["ls", "-la"],
    "image": "busybox",
    "memory": 1024,
    "vcpus": 1,
    "volumes": [
      {
        "host": {
          "sourcePath": "/tmp"
        },
        "name": "tmp"
      }
    ],
    "environment": [
        {"name": "VARNAME", "value": "VARVAL"}
    ],
    "mountPoints": [
        {
          "sourceVolume": "tmp",
          "containerPath": "/tmp",
          "readOnly": false
        }
    ],
    "ulimits": [
      {
        "hardLimit": 1024,
        "name": "nofile",
        "softLimit": 1024
      }
    ]
}
CONTAINER_PROPERTIES
}

// Job Queueの定義
resource "aws_batch_job_queue" "test_queue" {
  name     = "tf-test-batch-job-queue"
  state    = "ENABLED"
  priority = 1 // 実行優先度
  compute_environments = [aws_batch_compute_environment.sample.arn]
}
```

# TIPS
- JobがRUNNABLEのまま進まない問題は以下を参考にすると良さそう
    - [AWS Batch ジョブが RUNNABLE ステータスで止まっているのはなぜですか?](https://aws.amazon.com/jp/premiumsupport/knowledge-center/batch-job-stuck-runnable-status/)
    - [Troubleshooting AWS Batch](https://docs.aws.amazon.com/batch/latest/userguide/troubleshooting.html#job_stuck_in_runnable)
- コンピューティング環境のインスタンスタイプはあとから変えられない
    - [[AWS Batch]Terraformでコンピューティング環境のインスタンスタイプを後から変更する苦肉の策](https://n-s.tokyo/2019/03/awsbatch-change-instance-type/)

# 参考
- [AWS再入門ブログリレー AWS Batch編](https://dev.classmethod.jp/articles/relay_looking_back_aws-batch/)
- [AWS Batch とは何か](https://qiita.com/pottava/items/d9886b2e8835c5c0d30f)
