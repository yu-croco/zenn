---
title: "HashiCorp Vault 入門"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["HashiCorp","Vault"]
published: true
---

# これは何？
HashiCorp Vaultに入門するためのまとめです。この資料を読むことで `HashiCorp Vault is 何？` が（なんとなく）わかるようになります。

# 背景
エンジニアが障害対応などのためにDBに対してログインする必要があるが、DBのUser管理が大変なのでなんとかしたい（共有アカウントはセキュリティ的にアレだし、個々のエンジニア用に用意するのは手間がかかり過ぎる）！
そんな折、以下の情報を見て「HashiCorp Vault使えばいけるっぽいやん！」となったのでHashiCorp Vaultについて調べました。

https://jawsdays2021.jaws-ug.jp/timetable/track-c-1040/

# HashCorp Vault is 何？
Vaultはみんな大好きHashiCorp社の提供するプロダクトのひとつであり、以下のような機密情報管理を提供しています。
https://www.vaultproject.io/

- Secretの中央管理（Centralization)
- 暗号化（Encryption）
- 認証（Authentication)
- 認可（Authorization)
- 鍵交換（Rotation）

ユースケースとしては以下のようなものが挙げられます。

- シークレット管理
    - 中央集権的な生成、保存、提供
- データプロテクション
    - データ暗号化のためのAPIと暗号化のキー管理の機能を提供し、アプリケーションのデータをセキュアに保つ
- 動的シークレット
    - すべてのクライアントで一意かつ一時的なシークレットを発行可能

また、HashiCorp Vault用のhelm chartも用意されておりkubernetesとの連携もできます。Helm Chartも用意されていますし、KubernetesをVaultの認証として利用する方法も提供されています。
https://github.com/hashicorp/vault-helm

# Vaultのアーキテクチャ
Vaultのアーキテクチャは以下のようになっています（以下の公式より拝借）。

https://www.vaultproject.io/docs/internals/architecture#high-level-overview

![](https://storage.googleapis.com/zenn-user-upload/a11fe7dccaece67f94a23a03.png)

## Secrets Engine
シークレットエンジンはデータの作成/保存/暗号化を提供しており、Vaultの中心機能とも言える部分です。シークレットエンジンを利用することで、認証されたクライアントがシークレット（何かにアクセスするために必要な「何か」）を使用する際の管理ワークフローが実装できます。例えば特定のRDSに対して一時的にread可能なUserをVaultが自動で作成するといったことができます。

Secrets Engineは各種クラウドインフラ/DBなどに対応しており、例えば一時的に利用可能なIAM UserやMySQL Userを生成できます。

## Policy
Vaultではクライアントに対するアクセスコントロールのためにPolicyを提供しています。Policyは以下のようにPathベースのリソース（以下の場合では`database/*`）に対して実行可能な権限を `capabilities` として定義することが基本です。記法はTerraformと同じ感じです。

```terraform
path "database/*" {
  capabilities = ["read", "list"]
}
```

※この記事のハンズオンではRoot Tokenを利用して操作していますが、商用環境などでは必要最低限の権限を持つユーザーを作成したり、認証プロバイダー経由での認証などを行うことをおすすめします。

## Auth Method
Vaultへの認証とPolicyの紐付けを管理しています。認証には各種クラウドインフラ/AzureAD/Kubernetesなどと連携できます。

## Audit Devices
Vaultにおけるすべてのrequest/responseのログを管理している。殆どの操作に伴うrequest/responseに含まれる文字列はHMAC-SHA256によりhash化された状態で出力されます。出力先はFile/Syslog/Socketが提供されています。


# 軽いハンズオン
Vaultを使って一時的に利用可能なMySQL Userを自動生成してみます。ざっくりした流れは以下の感じです。
![](https://storage.googleapis.com/zenn-user-upload/12b3855e84f998fa4f238d3b.png)

※このハンズオンは以下に沿って自分で動かしたときの記録なので、気になる方は以下の記事も見て下さい。
https://github.com/hashicorp-japan/vault-workshop-jp/blob/master/contents/db.md


## Vaultをインストールする
手元の環境にVaultをインストールします。

```
$ brew tap hashicorp/tap
$ brew install hashicorp/tap/vault
$ vault -version // バージョンが表示されればok
Vault v1.1.1+ent ('7a8b0b75453b40e25efdaf67871464d2dcf17a46')

# コマンド補完をインストール
$ vault -autocomplete-install
$ exec $SHELL
```

## Vaultサーバーを起動する
Vaultのサーバーを起動します（便宜上developモードで起動しています）。起動した際に色々と必要な情報を表示するので手元に控えておきます。

```
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault server -dev
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Go Version: go1.15.13
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: false, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.7.3
             Version Sha: 5d517c864c8f10385bf65627891bc7ef55f5e827

==> Vault server started! Log data will stream in below:

....

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: cp1sStszUVUtZX+oE31CRxei3l4HQlV5iaip2PJ0TaU=
Root Token: s.1tIM6CCjPle33LQL4EBl6pC4
```

## Vaultにloginする
Vaultサーバー起動時に返された `Root Token` を使ってVaultにloginします。

```
$ export VAULT_ADDR='http://127.0.0.1:8200'
$ vault login
Token (will be hidden):  // ここにRoot Tokenを入れる
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.1tIM6CCjPle33LQL4EBl6pC4
token_accessor       fbWMZnjT81aGg148aroeTIhr
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

Secretの一覧を見てみましょう。

```yaml
$ vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_c12b3b70    per-token private secret storage
identity/     identity     identity_ceb10b15     identity store
secret/       kv           kv_bb6fd737           key/value secret storage
sys/          system       system_d3f8f769       system endpoints used for control, policy and debugging
```

## Database Secretを利用可能にする
defaultではDatabaseのsecret engineを利用できないため、明示的にONにする必要があります。

```
$ vault secrets enable -path=database database
Success! Enabled the database secrets engine at: database/
# secretsに `database/`が追加されている
$ vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_c12b3b70    per-token private secret storage
database/     database     database_8dec999e     n/a
identity/     identity     identity_ceb10b15     identity store
secret/       kv           kv_bb6fd737           key/value secret storage
sys/          system       system_d3f8f769       system endpoints used for control, policy and debugging
```

## アクセス対象のDBを用意する
以下のようにMySQLを用意します。

```yaml
$ docker run --name mysql -e MYSQL_ROOT_PASSWORD=rooooot -p 3306:3306 -d mysql:5.7.22
$ docker exec -it mysql bin/bash
$ mysql -u root -prooooot -h127.0.0.1
mysql> create database sample_vault;
Query OK, 1 row affected (0.01 sec)
mysql> use sample_vault;
mysql> create table products (id int, name varchar(50), price varchar(50));
Query OK, 0 rows affected (0.01 sec)
mysql> insert into products (id, name, price) values (1, "Terraform", "1580");
Query OK, 1 row affected (0.00 sec)
```

## VaultからMySQLに接続するための設定を行う
VaultからMySQLを操作するためにMySQLの設定情報を登録します。

※ DB関連の設定の詳細はこちらが参考になりそうです。
https://www.vaultproject.io/docs/secrets/databases/mysql-maria
https://www.vaultproject.io/api-docs/secret/databases/mysql-maria

```
$ vault write database/config/vault-mysql-db \
  plugin_name=mysql-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
  allowed_roles="role-vault-sample" \
  username="root" \
  password="rooooot"
```

## MySQLに対する操作をVaultにを設定する
VaultからMySQLに対してUserを生成するために、VaultとSQLの紐付けを `database/roles/` 配下に設定します（pluginの仕様）。
今回は `sample_vault.products` テーブルに対するSELECT権限のみ付与しています。またこの際に`default_ttl`（生成時のTTL）や再接続できる期限（`max_ttl`）などを設定することができます。

```
$ vault write database/roles/role-vault-sample \
  db_name=vault-mysql-db \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON sample_vault.products TO '{{name}}'@'%';" \
  default_ttl="120s" \
  max_ttl="180s"

Success! Data written to: database/roles/role-vault-sample

$ vault list database/roles
Keys
----
role-vault-sample
```

## VaultからMySQL Userを作成する
VaultからMySQLのUserを作成してみます。`database/creds/<先程作成したRole名>`に対してreadすることで新しいクレデンシャルが発行されます。
なおこのクレデンシャルはすべてのクライアントで一意のため、実行するたびに新しいクレデンシャルが発行されます。

```
$ vault read database/creds/role-vault-sample
Key                Value
---                -----
lease_id           database/creds/role-vault-sample/Ka3aJbZOs8oa3bjC8mc9YdZV
lease_duration     2m
lease_renewable    true
password           064mL4-cCTTfDDUnyd6z
username           v-root-role-vault-lxUKDka3w8xixU
```

## MySQLにログインしてみる
上記のUser情報でMySQLにログインしてみましょう。

```
mysql -u v-root-role-vault-lxUKDka3w8xixU -p064mL4-cCTTfDDUnyd6z -h127.0.0.1
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 19
Server version: 5.7.22 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| sample_vault       |
+--------------------+
2 rows in set (0.00 sec)

mysql> use sample_vault;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from products;
+------+-----------+-------+
| id   | name      | price |
+------+-----------+-------+
|    1 | Terraform | 1580  |
+------+-----------+-------+
1 row in set (0.00 sec)
```

このユーザーにはproductsテーブルに対する参照権限のみ与えられているため、productsテーブルに対する書き込みや他テーブルの追加はできません。

```
mysql> insert into products (id, name, price) values (2,"Vault", "2000");
ERROR 1142 (42000): INSERT command denied to user 'v-root-role-vault-lxUKDka3w8xixU'@'127.0.0.1' for table 'products'
mysql> create table users;
ERROR 1142 (42000): CREATE command denied to user 'v-root-role-vault-lxUKDka3w8xixU'@'127.0.0.1' for table 'users'
```

また、`max_ttl` を迎えた場合にはUserを利用できなくなります。

```
# 180秒後には再ログインできなくなっている
mysql -u v-root-role-vault-lxUKDka3w8xixU -p064mL4-cCTTfDDUnyd6z -h127.0.0.1
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1045 (28000): Access denied for user 'v-root-role-vault-lxUKDka3w8xixU'@'127.0.0.1' (using password: YES)
```

というわけで、いい感じにVaultから一時的なMySQL Userを作成することができました！

※業務で利用するには色々と変更/工夫する必要がありそうですが、それらの設定も公式Docsを辿るとだいたい記載されていそうなので、気になる方はのぞいてみてください。
https://www.vaultproject.io/docs

# 雑感
- Vaultすごい！（ｺﾅﾐｶﾝ）
- 商用環境で使うには色々調査と工夫が必要そう..
  - VaultからアクセスするDBのuser/passwordの管理
    - Vaultに任せてもいいけど、既存のリソースへの影響ががが
  - Vaultへの認証
    - 認証プロバイダー経由での認証がしたい（対応している）
  - 適切なPolicy管理
  - Vault自体の管理
    - 落ちると...
  - etc.
  
# 参考
Vaultについていい感じにまとまっています。
https://jawsdays2021.jaws-ug.jp/timetable/track-c-1040/

HashiCorpの中の人がまとめた資料
https://docs.google.com/presentation/d/14YmrOLYirdWbDg5AwhuIEqJSrYoroQUQ8ETd6qwxe6M/edit#slide=id.g55c21e2b16_2_1600

HashiCorpの中の人が書いたValueハンズオン（日本語）
メジャー機能を一通り触ってみたい人におすすめです。
https://github.com/hashicorp-japan/vault-workshop-jp

公式のDocs
実装で困った時にまずは見てみるのが良さそうです。
https://www.vaultproject.io/docs
