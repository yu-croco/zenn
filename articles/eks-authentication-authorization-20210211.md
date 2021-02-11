---
title: "EKSのaws-authとかIRSAとか"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes", "EKS"]
published: true
---

業務でEKSを使い始め、認証認可周りでハマった事が多かったので調べたことをまとめました。

## 前提
- 書いてること
    - EKSの認証認可（主にaws-authとIRSA）
    - そのへんを構築するための断片的なYAML/HCL
- 書いてないこと
    - 認証認可とはなんぞや
    - ゼロからk8sを構築する方法とか

## そもそもの話
EKSの認証認可についてはAWS公式が出している[クラスター認証](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/managing-auth.html)に書いてます。
> Amazon EKS は、IAM を使用して Kubernetes クラスターを認証します ( aws eks get-token コマンドを通じて、 のバージョン 1.16.156 以降、または AWS CLIKubernetes 用の AWS IAM オーセンティケーター)。
> ただし、承認には、従来の Kubernetes ロールベースアクセスコントロール (RBAC) を引き続き使用します。
> つまり、IAM は有効な IAM エンティティの認証にのみ使用されます。Amazon EKS クラスターの Kubernetes API とやり取りするためのアクセス許可はすべて、従来の Kubernetes RBAC システムを使用して管理されます。

AWS IAMで認証し、KubernetesのRBACで認可してるんですね。

*画像は[クラスター認証](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/managing-auth.html)から拝借
![](https://storage.googleapis.com/zenn-user-upload/ko4fo6wvazlyb4o8bwecpy75rfhl)

認証では特に色んな方法があると思いますが、この記事ではIAM Roleを用いてすることを前提に話を進めていきます。

ケースとしてはざっくり以下の2つについてまとめていきます。

1. 外部からEKSに対してアクセスする場合
    - kubectlなどを使う場合など
2. EKSからAWSリソースへとアクセスする場合
    - Pod上で動くアプリケーションからS3へアクセスする場合など

## 1. 外部からEKSに対してアクセスする場合
検証環境などではエンジニアがEKSに対し検証などのためにアクセスしたくなるケースがあると思います。例えば「Podを起動したがエラーになってしまうのでログが見たい」など。
そんなときには`aws-auth` を使うことでIAM RoleとEKSのRBACを紐付けます。

今回はチームA(teama)に対して参照を許可する権限を付与する場合を想定して進めます。

### aws-auth is何？
[クラスターのユーザーまたは IAM ロールの管理](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/add-user-role.html)によると、
> Amazon EKS クラスターを作成すると、このクラスターを作成した IAM エンティティユーザーまたはロール (例: フェデレーティッドユーザー) は、コントロールプレーンのクラスターの RBAC 設定で system:masters のアクセス許可が自動的に付与されます。この IAM エンティティは、ConfigMap やその他の表示可能な設定には表示されません。そのため、もともとクラスターを作成した IAM エンティティを必ず追跡してください。クラスターを操作する権限を他の AWS ユーザーやロールに付与するには、Kubernetes 内の aws-auth ConfigMap を編集する必要があります。

EKSクラスターを作成する際に使用したIAM以外のIAMリソースでEKSクラスターにアクセスしたい場合には、`aws-auth`に登録する必要があるようです。
aws-authはEKS上でConfigMapとして登録されています。

### 登録手順
#### 1. aws-authを登録する
EKSにアクセスしたいIAM Roleやをaws-authに追加します。
Terraformだと以下の感じです。この値がEKS側のConfigMapで`teama:view` というグループに紐づくような感じです。

```terraform
resource "kubernetes_config_map" "sample_aws_auth" {
  metadata {
    name = "aws-auth"
    namespace = "kube-system"
  }

  data = {
    mapRoles = <<YAML
    - rolearn: <紐付けたいIAM RoleのARN> 
    username: teama_developer
    groups:
    - teama:view
YAML
  }
}
```

aws-authはConfigMapとしてEKSに登録されるため、 YAMLで書く場合にはこんな感じになります。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: IAM_ROLE_ARN
      username: teama_developer
      groups:
        - teama:view
```

#### 2. EKSにaws-authをBindする
EKS側のRBACとaws-authを紐付けます。今回はCluster単位で `view` を付与したいのでClusterRoleBindingを使っています。

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developer-view-binding
subjects:
  - kind: Group
    name: teama:view # aws-authに登録したグループ
roleRef:
  kind: ClusterRole
  name: view # Kubernetesがデフォルトで持っているClusterRole
  apiGroup: rbac.authorization.k8s.io
```

作成したCLusterRoleBindingを覗いてみると、先ほど作成したものが表示されると思います。

```
$ kube get ClusterRoleBinding
NAME                                                   ROLE                                                                               AGE
...
developer-view-binding                                 ClusterRole/view                                                                   19m
...


$ kube describe ClusterRoleBinding developer-view-binding
Name:         developer-view-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  view
Subjects:
  Kind   Name            Namespace
  ----   ----            ---------
  Group  developer:view
```

これで指定したIAM RoleをもつエンジニアがEKS上でviewのRoleを利用できるようになります。


## 2. EKSからAWSリソースへとアクセスする場合
EKSから他のAWSリソースへアクセスしたい場合には以下の対応が必要です。

- EKS
    - ServiceAccountを作成する
    - Podに付与する
- AWS IAM
    - OIDCプロバイダーを作成を作成する
    - EKS ServiceAccountに対応するIAM Roleを作り、その際OIDCプロバイダーの情報を紐付ける


`OIDC` というのは、[サービスアカウントの IAM ロール - 技術概要](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html)曰く、
>2014 年、AWS Identity and Access Management に OpenID Connect (OIDC) を使用したフェデレーティッドアイデンティティのサポートが追加されました。この機能により、サポートされている ID プロバイダーで AWS API コールを認証し、有効な OIDC JSON ウェブトークン (JWT) を受け取ることができます。このトークンを AWS STS AssumeRoleWithWebIdentity API オペレーションに渡し、 IAM 一時的なロールの認証情報を受け取ることができます。これらの認証情報を使用して、Amazon S3 や DynamoDB などの任意の AWS サービスとやり取りできます。
>
>Kubernetes は、独自の内部 ID システムとして長い間、サービスアカウントを使用してきました。ポッドは、Kubernetes API サーバーのみが検証できる自動マウントトークン（OIDC JWT ではない）を使用して Kubernetes API サーバーで認証できます。これらのレガシーサービスアカウントトークンは期限切れにならず、署名キーの更新は難しいプロセスです。Kubernetes バージョン 1.12 では、サービスアカウント ID を含む OIDC JSON ウェブトークンである新しい ProjectedServiceAccountToken 機能のサポートが追加され、設定可能な対象者がサポートされます。
>
>Amazon EKS は、ProjectedServiceAccountToken JSON ウェブトークンの署名キーを含むクラスターごとにパブリック OIDC 検出エンドポイントをホストするようになり、IAM などの外部システムが Kubernetes によって発行された OIDC トークンを検証して受け入れることができるようになりました。

とのことで、OIDC経由でIAM Roleを使用できるらしいです（よく分かってない）。

また、ServiceAccountをPodに付与することで、Pod単位で必要な権限を絞ることができるのでセキュリティの面で推奨されています。
そのためNodeに紐づくIAM Roleに対してではなく、Podに対応するServiceAccountを作成します。
この考え方をIRSA(IAM Role for ServiceAccount)というらしいです。IRSAに関する詳細は[EKSを理解する（第2回）IRSAを用いたPod単位のIAMロール割り当て](https://www.bigtreetc.com/core-technology/eks-irsa/)が参考になると思います。

### 登録手順
手順としてはざっくり3つです。
1. EKSクラスターのOIDCプロバイダーを作成
2. IAM Roleを作成してPolicyにOIDCプロバイダーの情報を付与する
3. ServiceAccountを作成する
4. PodにServiceAccountを付与する

手動で準備する場合にはAWS公式が出している[サービスアカウントの IAM ロール](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/iam-roles-for-service-accounts.html)を辿っていくと出来ると思います。

#### 1. EKSクラスターのOIDCプロバイダーを作成
OIDCプロバイダーを作成します。

```terraform
resource "aws_iam_openid_connect_provider" "oidc_provider" {
  url = "${eks clusterのcluster_oidc_issuer_url}"
  client_id_list = ["sts.amazonaws.com"]
  thumbprint_list = ["9e99a48a9960b14926bb7f3b02e22da2b0ab7280"] # CA thumbprint
}
```

#### 2. IAM Roleを作成してPolicyにOIDCプロバイダーの情報を付与する
それをServiceAccount用のIAM Roleに関連付けます
今回はServiceAccountに対してS3への参照権限を付与します。

```yaml
resource "aws_iam_role" "s3_service_account" {
  name = "s3_sa"
  assume_role_policy = data.aws_iam_policy_document.s3_service_account.json
}

locals {
  eks_namespace = "dev"
}

data "aws_iam_policy_document" "s3_service_account" {
  statement {
    sid = "1"

    actions = [
      "s3:List*",
      "s3:Get*",
    ]

    resources = [
      "arn:aws:s3:::*",
    ]
  }

  // この辺が参考になると思います
  // https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html
  statement {
    effect = "Allow"
    actions = [
      "sts:AssumeRoleWithWebIdentity"
    ]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.oidc_provider.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "${replace(aws_iam_openid_connect_provider.oidc_provider.url, "https://", "")}:sub"
      values = [
        "system:serviceaccount:${local.eks_namespace}:${aws_iam_role.s3_service_account.name}"
      ]
    }
  }
}
```

#### 3. ServiceAccountを作成する
EKS側でServiceAccountも作ります。
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: <Terraformで作成したaws_iam_role.s3_service_accountのARN>
  name: s3-sa
```

#### 4. PodにServiceAccountを付与する

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: s3-sa # 作成したServiceAccountを指定
      ...
```

これでPodからS3へのデータ取得が可能となります。

## 雑感
- AWSとKubernetesの責任範囲が分かると何となく流れが掴めそう
- 不具合出た際にはこの辺を一つづつ辿るしか無いので、やっぱり裏側の仕組みはある程度知っておくのが良さそう

## 参考
### aws-auth
- [クラスターのユーザーまたは IAM ロールの管理](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/add-user-role.html)
- [EKSでの認証認可 〜aws-iam-authenticatorとIRSAのしくみ〜](https://katainaka0503.hatenablog.com/entry/2019/12/07/091737)
- [クラスター認証](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/managing-auth.html)
- [EKS Kubernetes プロバイダと下準備](http://blog.father.gedow.net/2019/10/04/eks-kubernetes-provider-config/)
- [Terraform kubernetes_config_map](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/config_map)

### IRSA
- [Resource: aws_iam_openid_connect_provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_openid_connect_provider)
- [Kubernetes サービスアカウントに対するきめ細やかな IAM ロール割り当ての紹介](https://aws.amazon.com/jp/blogs/news/introducing-fine-grained-iam-roles-service-accounts/)
- [サービスアカウントの IAM ロール](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [IAM Roles for Service Accounts をTerraformで手軽に体験してみる](https://onsd.hatenablog.com/entry/2019/09/21/015522)
