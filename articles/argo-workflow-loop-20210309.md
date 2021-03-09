---
title: "Argo WorkflowsでstepのOutputsを使って並行処理をしてみる"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ArgoWorkflows"]
published: false
---

Argo Workflowで `step A` のoutputsで出力したListを後続の `step B` で並列処理したいといったケースを実現したかったので調べてみました。


# 結論
Argo Workflowsのgithubにsampleが沢山掲載されており、その中にありました。

[argo-workflows/examples/loops-param-result.yaml](https://github.com/argoproj/argo-workflows/blob/master/examples/loops-param-result.yaml)

以下はgithubのサンプルからそのまま拝借しています。 `withParam` にoutputsで出力したListを渡すことで展開してくれるようです(この場合は `{{item}}` に個々の値が入り、`seconds` というパラメータの値として渡される)。
展開された値毎にWorkflowが生成されてそれぞれが並行処理で走るため、outputsのListに100件データがあれば100個のworkflowが並行で走る感じになります。


```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: loops-param-result-
spec:
  entrypoint: loop-param-result-example
  templates:
  - name: loop-param-result-example
    steps:
    - - name: generate
        template: gen-number-list
    - - name: sleep
        template: sleep-n-sec
        arguments:
          parameters:
          - name: seconds
            value: "{{item}}"
        withParam: "{{steps.generate.outputs.result}}" # ここでoutputsのリストを展開している

  - name: gen-number-list
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import json
        import sys
        json.dump([i for i in range(20, 31)], sys.stdout)
  - name: sleep-n-sec
    inputs:
      parameters:
      - name: seconds # ここに展開された値が渡されてくる
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo sleeping for {{inputs.parameters.seconds}} seconds; sleep {{inputs.parameters.seconds}}; echo done"]
```

# 動かしてみよう
minikubeで動かしてみました。
pod操作を行うためdefaultのServiceAccountで上記のYAMLを実行すると権限不足でエラーとなるため、以下のように適当なServiceAccountを用意します。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argo-wf-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argo-wf-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin # 本来は必要最低限の権限を付与すること！
subjects:
  - kind: ServiceAccount
    name: argo-wf-sa
    namespace: default
```

上記をapplyしたら、Argo WorkflowのGUIを表示して以下を実行します。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: loops-param-result
  namespace: default
spec:
  serviceAccountName: argo-wf-sa # 作ったSAを指定
  entrypoint: loop-param-result-example
  templates:
  - name: loop-param-result-example
    steps:
    - - name: generate
        template: gen-number-list
    - - name: sleep
        template: sleep-n-sec
        arguments:
          parameters:
          - name: seconds
            value: "{{item}}"
        withParam: "{{steps.generate.outputs.result}}"

  - name: gen-number-list
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import json
        import sys
        json.dump([i for i in range(20, 31)], sys.stdout)
  - name: sleep-n-sec
    inputs:
      parameters:
      - name: seconds
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo sleeping for {{inputs.parameters.seconds}} seconds; sleep {{inputs.parameters.seconds}}; echo done"]
```

`generate` ステップが終わると、 `sleep` ステップが並行で実行されます。よさそう。
![](https://storage.googleapis.com/zenn-user-upload/2w2e4q7aurtj0rlfna9n2xct138a)

# その他
[argo-workflows/examples](https://github.com/argoproj/argo-workflows/tree/master/examples) にはその他にも沢山の便利なサンプルが載っているので、是非参考にしてみてください。

今回取り上げたloop関連では以下のようなものもあります。
- [argo-workflows/examples/loops-sequence.yaml](https://github.com/argoproj/argo-workflows/blob/master/examples/loops-sequence.yaml)
- [argo-workflows/examples/loops-param-argument.yaml](https://github.com/argoproj/argo-workflows/blob/master/examples/loops-param-argument.yaml)

