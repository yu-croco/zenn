---
title: "AWS CloudWatchLogsをDatadogへお手軽転送してみる"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","CloudWatchLogs","Kinesis"]
published: false
---

ログの集約先をDatadogにしたいため、AWSから出てくるログをDatadogへ送る事になった際に調べたことをまとめました。

# 結論
Kinesis Data Firehosから特定の外部サービスにデータを転送できるようになり、この方法でDatadogへデータを転送できます。
[Analyze logs with Datadog using Amazon Kinesis Data Firehose HTTP endpoint delivery](https://aws.amazon.com/jp/blogs/big-data/analyze-logs-with-datadog-using-amazon-kinesis-data-firehose-http-endpoint-delivery/)

# 具体的な方法
基本的には[HTTP Endpoint (e.g. New Relic) Destination](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kinesis_firehose_delivery_stream#http-endpoint-eg-n[…]-relic-destination)を参考にすれば対応できます。
*Datadog側でAPI Keyを作成しておく必要があります。

```terraform
resource "aws_cloudwatch_log_group" "log_group" {
  name = "sample-app"
}

resource "aws_cloudwatch_log_subscription_filter" "test_lambdafunction_logfilter" {
  name            = "test_lambdafunction_logfilter"
  role_arn        = aws_iam_role.iam_for_lambda.arn
  log_group_name  = aws_cloudwatch_log_group.log_group.name
  filter_pattern  = "" // 全データを対象とする
  destination_arn = aws_kinesis_firehose_delivery_stream.send_to_datadog.arn
  distribution    = "Random"
}

// パラメータストアにDatadog APIを格納している前提
data "aws_kms_key" "datadog_api" {
  key_id = "alias/datadog/api_key"
}

resource "aws_kinesis_firehose_delivery_stream" "send_to_datadog" {
  name        = "terraform-kinesis-firehose-test-stream"
  destination = "http_endpoint"

  // firehoseからDatadogへ投げたデータのバックアップのために必須
  s3_configuration {
    role_arn           = aws_iam_role.firehose.arn
    bucket_arn         = aws_s3_bucket.bucket.arn
    buffer_size        = 10
    buffer_interval    = 400
    compression_format = "GZIP"
  }

  http_endpoint_configuration {
    // ref: https://docs.aws.amazon.com/firehose/latest/dev/http_troubleshooting.html
    url                = "https://aws-kinesis-http-intake.logs.datadoghq.com/v1/input"
    name               = "Datadog Log"
    access_key         = data.aws_kms_key.datadog_api.value
    buffering_size     = 15
    buffering_interval = 600
    role_arn           = aws_iam_role.firehose.arn
    s3_backup_mode     = "FailedDataOnly"

    request_configuration {
      content_encoding = "GZIP"

      // Datadogでのタグとして設定できる
      common_attributes {
        name  = "env"
        value = "test"
      }
      common_attributes {
        name  = "app-name"
        value = "my-app"
      }
    }
  }
}
```

# 余談
Fargateタイプのリソースを扱っている場合には、side carでdd-agentを動かすのが一番楽ではないかと思います。
*秘密情報は[Systems Manager パラメータストアを使用した機密データの指定](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/specifying-sensitive-data-parameters.html)を参考に、パラメータストア経由でデータを取得するようにするのが良いと思います。
