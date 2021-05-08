---
title: "Argo WorkflowsでSSO + RBACを使ってみる"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ArgoWorkflows", "AzureAD"]
published: true
---

# コレは何？
Argo Workflowsに対してAzure AD経由でSSOするために調べたことをまとめました。


## 1. Azure AD Applicationを登録
[Azure AD App Registration Auth using OIDC](https://argoproj.github.io/argo-cd/operator-manual/user-management/microsoft/#azure-ad-app-registration-auth-using-oidc)の `1. Register a new Azure AD Application` と `2. Setup permissions for Azure AD Application` に沿ってAzure ADに登録する。
*ArgoCD向けの説明だがだいたいそのまま使える

### Register a new Azure AD Application
[Quickstart: Register an application](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)に沿ってAzure ADのApplicationを登録する。
登録した際に `Redirect URI` , `Application (client) ID` , `Directory (tenant) ID` , `Secret` の値を控えておく。

`Redirect URI` はSSOでログイン後にリダイレクトさせたいURIを指定する。
形式としては `https://<ARGO_WF_DOMAIN>/oauth2/callback` という感じのを登録する（例: `https://localhost:2746/oauth2/callback` )。

### Setup permissions for Azure AD Application
`Token Configuration` で `groups` に加えて `email` も加えておく。

## 2. Azure AD Enterprise applicationsの設定
`1.` で作成したAzure Applicationに対して `Enterprise applications` というものができているので、 そこの `Users and groups` でargo-workflowsで操作を許可したいユーザー/グループを追加する必要がある。
これを設定しておかないと、Argo Workflowsへのログインは可能であるが権限が何もないため `Failed to load Error: Unsuccessful HTTP response` というエラーが出る。

## 3. Argo Workflows側の設定
### SSO化
[Argo Server SSO](https://argoproj.github.io/argo-workflows/argo-server-sso/)によると、SSO化自体はArgo Server起動時に `-auth-mode sso` を与えることで対応できる。 
その上でclientSecretなどの情報を用意する必要がある。

[argoproj / argo-helm](https://github.com/argoproj/argo-helm)を利用している場合には以下のように設定するとSSOと諸々の準備ができる。
対応箇所は[こちら](https://github.com/argoproj/argo-helm/blob/master/charts/argo/values.yaml#L257-L285)。

```yaml
argo:
  server:
    sso:
     ## The root URL of the OIDC identity provider.
     issuer: https://sts.windows.net/<AZURE_AD_DIRECTORY_ID>/
     ## Name of a secret and a key in it to retrieve the app OIDC client ID from.
     clientId:
       name: argo-server-sso
       key: client-id
     # Name of a secret and a key in it to retrieve the app OIDC client secret from.
     clientSecret:
       name: argo-server-sso
       key: client-secret
     ## The OIDC redirect URL. Should be in the form <argo-root-url>/oauth2/callback.
     redirectUrl: https://<REDIRECT_URI>/oauth2/callback
     rbac:
       enabled: true
```

### Client IDとClient SecretをArgo Workflowsへ渡す
Azure AD側で登録した `Application (client) ID` , `Secret` をargo-workflowsから読み込めるようにする。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argo-server-sso
type: Opaque
data:
  client-id: <Application (client) ID>
  client-secret: <Secret>
```

### RBAC設定
[Argo Server SSO - SSO RBAC](https://argoproj.github.io/argo-workflows/argo-server-sso/#sso-rbac)を参考にしてAzure ADの特定のgroupIdに対してRBACしてみる。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argo-wf-readonly-role
rules:
  # pod get/watch is used to identify the container IDs of the current pod
  # pod patch is used to annotate the step's outputs back to controller (e.g. artifact location)
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
      - watch
      - patch
  # logs get/watch are used to get the pods logs for script outputs, and for log archival
  - apiGroups:
      - ""
    resources:
      - pods/log
    verbs:
      - get
      - watch
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argo-wf-readonly
  annotations:
    # The rule is an expression used to determine if this service account 
    # should be used. 
    # * `groups` - an array of the OIDC groups
    # * `iss` - the issuer ("argo-server")
    # * `sub` - the subject (typically the username)
    # Must evaluate to a boolean. 
    # If you want an account to be the default to use, this rule can be "true".
    # Details of the expression language are available in
    # https://github.com/antonmedv/expr/blob/master/docs/Language-Definition.md.
    workflows.argoproj.io/rbac-rule: "<groupId>"
    # The precedence is used to determine which service account to use whe
    # Precedence is an integer. It may be negative. If omitted, it defaults to "0".
    # Numerically higher values have higher precedence (not lower, which maybe 
    # counter-intuitive to you).
    # If two rules match and have the same precedence, then which one used will 
    # be arbitrary.
    workflows.argoproj.io/rbac-rule-precedence: "1"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argo-wf-readonly
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argo-wf-readonly
subjects:
  - kind: ServiceAccount
    name: argo-wf-readonly-role
    namespace: argo-wf
```

# 参考
- [Argo CD と Argo Workflows で SSO + RBAC](https://blog.hatappi.me/entry/2021/01/03/190416)
- [argo-workflowsのSSO+RBACをazure連携させる](https://krrrr.hatenablog.com/entry/2021/01/20/183851)
- [多分わかりやすいOpenID Connect](https://tech-lab.sios.jp/archives/8651)
- [Azure AD に登録できる 「アプリ」と「リソース」、「API 権限」を理解する](https://jpazureid.github.io/blog/azure-active-directory/oauth2-application-resource-and-api-permissions/)
