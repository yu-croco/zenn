---
title: "Gatekeeper入門"
emoji: "🥅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Gatekeeper", "Conftest"]
published: true
---

# イントロ
Kubernetesを利用していると、`Deploymentのkindに対しては必ずlabelsに管理しているチームのIDを付与することを強制したい` といったリソースへの制約を課したくなることがあると思います。
そういった成約を自前で準備するのはいささか大変です。

[Gatekeeper](https://github.com/open-policy-agent/gatekeeper) は[Cloud Native Computing Foundation](https://www.cncf.io/)傘下のプロジェクトであり、CRDベースでこういった制約を定義してKubernetesに対して適応することができるツールです。

Kubernetesでは[admission controller webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)を利用することでPolicyの決定を内部のAPIサーバーから引き剥がすことができるのですが、
Gatekeeperはそのwebhooksを利用してこの機能を実現しているようです。


Gatekeeperでは設定によりKubernetesのアドミッションコントロールをカスタマイズする方法を提供しています。

ちなみに、[OPA with its sidecar kube-mgmt](https://github.com/open-policy-agent/gatekeeper)という方法でバリデーションを掛けることもできますが、Gatekeeperだと以下のメリットがあるらしいです。

> - An extensible, parameterized policy library
> - Native Kubernetes CRDs for instantiating the policy library (aka "constraints")
> - Native Kubernetes CRDs for extending the policy library (aka "constraint templates")
> - Audit functionality

# 定義方法
[How to use Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/howto)によると、まずは `ConstraintTemplate`を定義する必要があるらしいです。
`ConstraintTemplate` を定義することでCRDが生成され、その生成されたCRDを使って具体的な制約を定義する流れです。

以下の例では、`ConstraintTemplate` を定義することで `K8sRequiredLabels` というCRDが動的に生成されます。
制約の定義は[Rego](https://www.openpolicyagent.org/docs/latest/policy-language/)というクエリー言語で定義する必要があります。

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        # ここで定義した名称でCRDが生成される
        kind: K8sRequiredLabels
      validation:
        # K8sRequiredLabelsを利用する際に指定するパラメータ
        openAPIV3Schema:
          properties:
            labels:
              type: array
              items: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      # Regoという言語を使って制約を実装している
      # `.metadata.labels`に parametersで指定された値が存在することを検証している
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        }
```

生成されたCRDを使って、以下のようにどのリソースに対してどのような制約を掛けるか定義します。
この場合はすべてのNamespaceリソースに対して `.metadata.labels`に`gatekeeper`という値が付与されていることを強制しています。

```yaml
apiVersion: constrants.gatekeeper.sh/v1beta1
kind: K8srequiredlabels
metadata:
  name: ns-must-have-gk
spec:
  match:
    kinds:
      - apiGroup: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["gatekeeper"]
```

制約を掛けたくないリソースがある場合には以下のように `Config` リソースを定義することで適応から除外できるらしいです。
```yaml
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
  name: config
  namespace: "gatekeeper-system"
spec:
  match:
    - excludedNamespaces: ["kube-system", "gatekeeper-system"]
      processes: ["*"]
    - excludedNamespaces: ["audit-excluded-ns"]
-     processes: ["audit"]
```

# CIどうするの？
Gatekeeperはあくまでもapplyなどでリソースを適応する際にバリデーションするのであり、事前にその定義を満たしているかは検証できません。
そのため、個人的には[Conftest](https://github.com/open-policy-agent/conftest)を使ってKubernetesの定義に対してテストを記載するのが良いのかなと思っています。

公式([open-policy-agent/conftest](https://github.com/open-policy-agent/conftest))によると、ConftestはYAMLなどの定義に対してRegoを使って定義が意図どおりであるかを検証するツールです。Kubernetesの定義以外にもTerraformや他の定義に対しても使えるらしいです。
公式のものをそのまま拝借していますが、以下のように定義して `$ conftest test deployment.yaml` といった感じでテストを実行します。
手軽で良さそうですね。

```yaml
package main

deny[msg] {
  input.kind == "Deployment"
  not input.spec.template.spec.securityContext.runAsNonRoot

  msg := "Containers must not run as root"
}

deny[msg] {
  input.kind == "Deployment"
  not input.spec.selector.matchLabels.app

  msg := "Containers must provide app label for pod selectors"
}
```

# 参考
- [open-policy-agent/gatekeeper](https://github.com/open-policy-agent/gatekeeper)
- [GatekeeperによるKubernetesポリシの拡張](https://www.infoq.com/jp/news/2019/10/opa-gatekeeper-kubernetes/)
- [Gatekeeper Introduction](https://open-policy-agent.github.io/gatekeeper/website/docs/)
- [Conftest で CI 時に Rego で記述したテストを行う](https://amsy810.hateblo.jp/entry/2020/04/03/124913)
- [Gatekeeper/conftestのRegoをDRYに管理する](https://hack.nikkei.com/blog/advent20201224/)