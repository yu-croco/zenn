---
title: "Terraformのremote_stateについておさらい"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Terraform"]
published: false
---

Terraformのterraform_remote_stateを使ったので頭の整理を兼ねてまとめました。
*ソースコードはこちらにアップしていますので、ご自由にお使いください。 [terraform_remote_state_sample](https://github.com/yu-croco/terraform_remote_state_sample)

## terraform_remote_state is 何？
Terraformの公式ドキュメント（[The terraform_remote_state Data Source](https://www.terraform.io/docs/language/state/remote-state-data.html) ） 曰く 、

> The terraform_remote_state data source retrieves the root module output values from some other Terraform configuration, using the latest state snapshot from the remote backend.

とのことで、あるリソースのoutputを別のリソースから参照できる（しかもリモートに格納されてる最新のやつを）らしい。
TerraformのリソースはAとBとして別々に管理されているけれど、何らかの理由で「BからAのxxを参照したい」といったケースに役立ちそう。

流れとしてはざっくり以下の感じ。
1. S3にステートを管理するためのバケットを用意する
2. TerraformとS3バケットを連携する
3. outputリソースを定義する
4. `terraform_remote_state` を使用してoutputした値を参照する


それではみていこう。

## 流れ
独立して実行管理されているIAM Roleを管理するディレクトリ（`iam` と `iam2`があり、 `iam` 実行後に `iam2` が実行される）があり、 `iam2` 側で `iam` の実行結果に依存する情報を使用したいケースを想定してみていくことにする。

### 1. S3にステートを管理するためのバケットを用意する
以下のリソースをapplyしてS3バケットを作る

```terraform
resource "aws_s3_bucket" "sample_tf" {
  bucket = "sample-tf"
  versioning {
    enabled = true
  }
}
```

### 2. TerraformとS3バケットを連携する

`iam` 側で以下のような感じの定義を `backend.tf` として作り `terraform init` すると、1.で作成したS3バケットと連携が取れるようになる。
S3バケットは予め用意していないといけないので注意が必要。

```terraform
terraform {
  backend "s3" {
    bucket = "sample-tf"
    key    = "terraform.tfstate1"
    region = "ap-northeast-1"
  }
}
```

### 3. outputリソースを定義する
`iam` 側で以下のように定義する。

```terraform
// main.tf
data "aws_iam_policy_document" "sample_tf" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "sample_tf" {
  name                  = "sample_tf"
  assume_role_policy    = data.aws_iam_policy_document.sample_tf.json
  description           = "for terraform example"
  force_detach_policies = false
  tags                  = {}
}

output "iam_role_name" {
  value = aws_iam_role.sample_tf.name
}
```

この状態で `terraform plan` すると以下のような感じで`output` が作成される予定であることが分かる

```
Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + iam_role_arn = (known after apply)
```

applyするとbackendで指定したリソースがS3バケットに作成される。



### 4. `terraform_remote_state` を使用してoutputした値を参照する
`iam2` 側で以下のように定義する。 
terraform_remote_stateを使って `aws_iam_role.sample_tf2` のname定義で `iam` 側で定義したnameを参照するようにしている。

```terraform
data "aws_iam_policy_document" "sample_tf2" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "sample_tf2" {
  // data.terraform_remote_state.${terraform_remote_stateで定義したリソース名}.outputs.${main/iam側のoutputで指定したリソース名}
  name                  = "${data.terraform_remote_state.sample_tf.outputs.iam_role_name}-2"
  assume_role_policy    = data.aws_iam_policy_document.sample_tf2.json
  description           = "for terraform example"
  force_detach_policies = false
  tags                  = {}
}
```

これをapplyするとちゃんと `sample_tf-2` というIAM Roleが作成される。



## 参考
- [The terraform_remote_state Data Source](https://www.terraform.io/docs/language/state/remote-state-data.html)
- [【Terraform】terraform_remote_stateデータソースの使い方](https://dev.classmethod.jp/articles/how-to-use-terraform-remote-state/)
