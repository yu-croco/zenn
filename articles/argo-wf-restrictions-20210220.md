---
title: "Argo WorkflowのWorkflow Restrictionsを使ってみた"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ArgoWorkflow"]
published: false
---
Argo Workflowの `templateReferencing` を触ってまとめてみました。

## templateReferencing
### 概要
Argo Workflowで `Workflows` 経由から実行したいがWorkflowを直接実行させたくないケースがあると思います。 そんなときには`templateReferencing` を使うと簡単にWorkflowの直接実行を防げます。
[Workflow Restrictions](https://argoproj.github.io/argo-workflows/workflow-restrictions/) 曰く、templateReferencingには `Strict` と `Secure` があります。

- Strict

> Only Workflows using "workflowTemplateRef" will be processed. This allows the administrator of the controller to set a "library" of templates that may be run by its opeartor, limiting arbitrary Workflow execution.

とのことで、 `workflowTemplateRef` を使ってないWorkflowを実行することを制限しています。


- Secure

> Only Workflows using "workflowTemplateRef" will be processed and the controller will enforce that the WorkflowTemplate that is referenced hasn't changed between operations. 
> If you want to make sure the operator of the Workflow cannot run an arbitrary Workflow, use this option.

とのことで、 `Strict` に加えて、Workflow実行中にworkflowTemplateRefに変更があってもそれを受け付けないようです。

### 内部実装をちょっと追ってみる

この機能自体は以下で定義されています(ざっくり以下の感じ)。

https://github.com/argoproj/argo-workflows/blob/master/config/config.go#L363

```go
type WorkflowRestrictions struct {
	TemplateReferencing TemplateReferencing `json:"templateReferencing"`
}

type TemplateReferencing string

const (
	TemplateReferencingStrict TemplateReferencing = "Strict"
	TemplateReferencingSecure TemplateReferencing = "Secure"
)

func (req *WorkflowRestrictions) MustUseReference() bool {
	if req == nil {
		return false
	}
	return req.TemplateReferencing == TemplateReferencingStrict || req.TemplateReferencing == TemplateReferencingSecure
}

func (req *WorkflowRestrictions) MustNotChangeSpec() bool {
	if req == nil {
		return false
	}
	return req.TemplateReferencing == TemplateReferencingSecure
}
```

なるほど、 `MustUseReference` と `MustNotChangeSpec` のフラグとして使ってるんですね。

### MustUseReference
この関数は以下で呼び出されています。
https://github.com/argoproj/argo-workflows/blob/master/workflow/controller/operator.go#L3112

```go
func (woc *wfOperationCtx) setExecWorkflow() error {
	if woc.wf.Spec.WorkflowTemplateRef != nil {
		err := woc.setStoredWfSpec()
		if err != nil {
			return err
		}
		woc.execWf = &wfv1.Workflow{Spec: *woc.wf.Status.StoredWorkflowSpec.DeepCopy()}
		woc.volumes = woc.execWf.Spec.DeepCopy().Volumes
	} else if woc.controller.Config.WorkflowRestrictions.MustUseReference() { // ここで呼び出されている
		return fmt.Errorf("workflows must use workflowTemplateRef to be executed when the controller is in reference mode")
	} else {
		err := woc.controller.setWorkflowDefaults(woc.wf)
		if err != nil {
			return err
		}
		woc.volumes = woc.wf.Spec.DeepCopy().Volumes
	}
	return nil
}
```

`argo submit` コマンドやArgo WorkflowのGUIからsubmitした際に呼び出される処理の中で呼び出されています。
Workflowを実行する前に実行対象のWorkflowをセットするようですが、この関数ではセットする直前のValidationとして使っているようです。
`WorkflowTemplateRefが定義されていない` && `MustUseReferenceを定義している` 場合にはエラーとなる仕組みです。


### MustNotChangeSpec
この関数は以下で呼び出されています。
https://github.com/argoproj/argo-workflows/blob/master/workflow/controller/operator.go#L3146

```go
func (woc *wfOperationCtx) setStoredWfSpec() error {
	wfDefault := woc.controller.Config.WorkflowDefaults
	if wfDefault == nil {
		wfDefault = &wfv1.Workflow{}
	}
	if woc.wf.Status.StoredWorkflowSpec == nil {
		wftHolder, err := woc.fetchWorkflowSpec()
		if err != nil {
			return err
		}

		// Join WFT and WfDefault metadata to Workflow metadata.
		wfutil.JoinWorkflowMetaData(&woc.wf.ObjectMeta, wftHolder.GetWorkflowMetadata(), &wfDefault.ObjectMeta)

		// Join workflow, workflow template, and workflow default metadata to workflow spec.
		mergedWf, err := wfutil.JoinWorkflowSpec(&woc.wf.Spec, wftHolder.GetWorkflowSpec(), &wfDefault.Spec)
		if err != nil {
			return err
		}

		woc.wf.Status.StoredWorkflowSpec = &mergedWf.Spec
		woc.updated = true
	} else if woc.controller.Config.WorkflowRestrictions.MustNotChangeSpec() { // ここで呼び出されている
		wftHolder, err := woc.fetchWorkflowSpec()
		if err != nil {
			return err
		}
		mergedWf, err := wfutil.JoinWorkflowSpec(&woc.wf.Spec, wftHolder.GetWorkflowSpec(), &wfDefault.Spec)
		if err != nil {
			return err
		}
		if mergedWf.Spec.String() != woc.wf.Status.StoredWorkflowSpec.String() {
			return fmt.Errorf("workflowTemplateRef reference may not change during execution when the controller is in reference mode")
		}
	}
	return nil
}
```

`setExecWorkflow` から呼び出されており、Workflowを実行する前に実行対象のWorkflowをセットする処理のようです。
この処理は正確に調査できていないのでコードを読んだ上での雰囲気ですが、`mergedWf.Spec.String() != woc.wf.Status.StoredWorkflowSpec.String()` のところで、Configの設定値と現在のコンテキストで持っている設定値が違う場合にはエラーを返すようにしています。


## 導入方法
[Workflow Restrictions](https://argoproj.github.io/argo-workflows/workflow-restrictions/)曰く、以下で設定できるらしいです。

```yaml
# This file describes the config settings available in the workflow controller configmap
apiVersion: v1
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
data: |
  workflowRestrictions:
    templateReferencing: Secure
```

[Argo Helm Charts](https://github.com/argoproj/argo-helm) というHelm用のArgo Workflowを使うと以下のような感じで書けます。
超簡単なサンプルはこちらにアップしていますのでご自由にお使いください。[sample_argo_wf_helm_customization](https://github.com/yu-croco/sample_argo_wf_helm_customization)

```yaml
# Chart.yaml
apiVersion: v2
name: argo-wf
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
dependencies:
  - name: argo
    version: 0.16.2
    repository: https://argoproj.github.io/argo-helm
    alias: argo-wf

# values.yaml
argo-wf:
  controller:
    workflowRestrictions:
      templateReferencing: Secure
```

設定を反映させると ConfigMapに `workflowRestrictions` が反映されていることが分かります。

```
$ kubectl get ConfigMap
NAME                                               DATA   AGE
argo-wf-1613791232-workflow-controller-configmap   1      35s
kube-root-ca.crt                                   1      119s

$ kubectl describe ConfigMap argo-wf-1613791232-workflow-controller-configmap
Name:         argo-wf-1613791232-workflow-controller-configmap
Namespace:    default
Labels:       app.kubernetes.io/managed-by=Helm
              chart=argo-wf-0.16.2
              heritage=Helm
              release=argo-wf-1613791232
Annotations:  meta.helm.sh/release-name: argo-wf-1613791232
              meta.helm.sh/release-namespace: default

Data
====
config:
----
containerRuntimeExecutor: docker
workflowRestrictions:
  templateReferencing: Secure

Events:  <none>
```

この状態で以下のWorkflowを直接applyしてみます。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: hello-whale
spec:
  entrypoint: whalesay
  templates:
  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["hello world"]
```

apply後にworkflowの状態を見てみると、 `workflows must use workflowTemplateRef to be executed when the controller is in reference mode` と言われてエラーになっています。意図通りですね。

```
$ kube get workflow
NAME          STATUS   AGE
hello-whale   Error    27m

$ kube describe workflow  hello-whale
Name:         hello-whale
Namespace:    default
Labels:       workflows.argoproj.io/completed=true
  workflows.argoproj.io/phase=Error
Annotations:  <none>
API Version:  argoproj.io/v1alpha1
Kind:         Workflow
...
Events:
  Type     Reason          Age   From                 Message
  ----     ------          ----  ----                 -------
  Warning  WorkflowFailed  27m   workflow-controller  workflows must use workflowTemplateRef to be executed when the controller is in reference mode
```

GUIから見ても同じ感じでエラーになっています。

![](https://storage.googleapis.com/zenn-user-upload/7l7b8lt1975y66uh6y8s7jlkjkvr)

一方で以下のWorkflowTemplateを実行すると、workflowが回り始めます。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: workflow-template-submittable
spec:
  entrypoint: whalesay-template
  arguments:
    parameters:
      - name: message
        value: hello world
  templates:
    - name: whalesay-template
      inputs:
        parameters:
          - name: message
      container:
        image: docker/whalesay
        command: [cowsay]
        args: ["{{inputs.parameters.message}}"]
```

![](https://storage.googleapis.com/zenn-user-upload/kdqh20oemkcub7tq3whl8b35c405)

## まとめ
- Argo WorkflowからWorkflowを直接実行させることを禁止したい場合には `templateReferencing` を使うとできそう。
- ドキュメントだけ読んでも具体的な処理は不明確だったりするので、内部実装を追っていくと理解が捗りそう。

## 参考
- [Workflow Restrictions](https://argoproj.github.io/argo-workflows/workflow-restrictions/)
- [argoproj/argo-workflows](https://github.com/argoproj/argo-workflows)
- [Workflow Templates](https://github.com/argoproj/argo-workflows/blob/master/docs/workflow-templates.md)
