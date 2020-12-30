---
title: "Service Meshとは何であるか"
emoji: "😼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes", "ServiceMesh"]
published: true
---

Kubernetes関連のエコシステムに入門して学んだことをまとめている。今回はService Meshについて。

## 背景
物事には基本的にそれに至るまでの背景が存在する。Service Meshにもまた、それが生まれてきた背景がある。まずはその流れをざっと追っていくことにする。


### モノリシック時代のアプリケーション開発
みんな大好きモノリシックなアプリケーションだ。1つのディレクトリー内にフロントエンド/バックエンドなどのリソースが同居しており、その先にはDBなどの永続化機構と繋がっているだろう。エンジニアは同じプロジェクト内で開発を行う。

アプリケーションは1つのプロセスとして動いており、デプロイする際にも稼働する際にもどんなときでもみんな1つのまとまりとして動いている。

### マイクロサービス時代のアプリケーション開発
モノリシックなアプリケーションが大きくなるにつれて、色々なツラミが出てくるだろう。詳しい話は割愛するが、ざっくり以下のようなことが起こるのではないだろうか。

- 依存関係/影響範囲の肥大化
  - 機能Aで問題が発生すると、関係ない機能に影響が出たりサービス全体に影響が出るかもしれない
- 冗長化やパフォーマンス改善をピンポイントで行えない
  - 機能Aに対するアクセスに対してだけ冗長化したいが、そのためにはインスタンスをまるごと増やさないといけない

そんな流れに伴ってマイクロサービスなる考え方が出てきた（実際はもっと歴史があるだろうから、そこは調べて欲しい）。従来の大きな1つのアプリケーションをドメイン単位にサービスとして分割し、それぞれが独立したものとして機能するような構成へと変わっていった。

アプリケーションはコンテナ上で管理され、kubernetesのようなオーケストレーションツールを用いて運用されている、いわゆるクライドネイティブといわれるものが主流になってきた。この辺りの話は、クラウドネイティブソフトウェアのための持続可能なエコシステムの構築を目的とした[Cloud Native Computing Foundation (CNCF)](https://www.cncf.io/)という組織があるので、そちらを覗いて頂くのが良いだろう。クラウドネイティブの定義は[そちら](https://github.com/cncf/toc/blob/master/DEFINITION.md)が参考になるだろう。

これによって各開発者は自分たちの関係ある機能に対してより集中でき、サービス単位でのスケールアウトなどもより簡単に実施できるようになってきた。

すべてが万々歳のように見えるようだが、一方で問題も出てきた。マイクロサービス化に伴って今までは同じプロセス上に存在していたリソースへのアクセスが面倒になった。例えば、サービスAがサービスBと連携するにはサービスAがサービスBのエンドポイントにアクセスしなくてはならないのだ。以前は対象のファイルをimportなり何なりすれば簡単にアクセスできたのだが、今では外部アクセスが必須である。サービスAとサービスBは全く別のサービスとして独立しており、必要なデータを取得するにはAPIを介するよりほかない。

ざっくりのイメージとしては以下の感じだろうか（ツッコミどころはたくさんあるだろうが、あくまでも雰囲気を掴むためのものだ）。
![](https://storage.googleapis.com/zenn-user-upload/7nq1wyompsnm0tw3uq22cuh98jse)

外部アクセスということは、常に失敗と隣り合わせな処理であることを意味する。サービスAからサービスBにアクセスする時、サービスBはもしかしたらトラブルでレスポンス速度が非常に遅くなっているかもしれないし、場合によっては応答しないかもしれない。そのため各サービスはリトライやタイムアウトなどの機構を備えておかなければならない。サービスの数が数個であればなんとかなりそうだが、数十、数百になるともう管理できないだろう。

また、不具合があった場合のトレースも複雑化していくだろう。サービス間の連携が多くなるほど、何処のサービスの何が原因で不具合が発生しているのかを調査するのが難しくなる。ログ機構を各サービス毎に揃えようにも、既述のように扱うサービスが多くなってくるごとに管理が難しくなることは目に見えている。

このようにモノリシックからマイクロサービスへの変遷に伴ってサービス間通信に対する考慮事項がバク増した。

Service Meshはこういったサービス間通信に関連する問題を解決するために生まれたのである。


## Service Mesh
マイクロサービス（や、分散システム）中心のシステム設計におけるサービス間通信の管理をしてくれる。

### 特徴
- アプリケーション層（L7）でのトラフィックルーティング
- ロードバランサー
- サービスリリース管理
  - カナリアリリースなど
- 動的サービスディスカバリー
- ヘルスチェックタイムアウト/リトライなどの基礎
- サービスのモニタリング
- サービス間のエンドポイントやネットワークのploicy管理
- ロギングやコンフィグの提供

### コンセプト
Service Meshは `data plane` と `control plane` の2つの概念から構成されている。

#### 1. Data Plane
サービス間通信を直接捌く（ロードバランシング、サービスディスカバリ、ヘルスチェックなどなど）、いわゆる `実際に仕事する奴` である。
data planeは各マイクロサービスのサイドカーとして配置され、サービス間の通信制御を行う。

Data Planeで有名なサービスの一つとしては [Envoy](https://www.envoyproxy.io/) がある。

#### 2. Control Plane
control planeは `data planeに命令したり仕事ぶりを管理する奴` である。control panelはサービス間の通信に対して直接関わることはないが、data planeに対するpolicyの適応などを管理者（我々エンジニア）に提供する。

control planeで有名なサービスの一つとしては[Istio](https://istio.io/latest/docs/concepts/what-is-istio/)がある。

[Istio](https://istio.io/latest/docs/concepts/what-is-istio/)公式には以下のような図解があり、data planeとcontrol planeの役割のイメージがつく感じになっている。

![Istio](https://storage.googleapis.com/zenn-user-upload/h8uox6r4aqt5chsy00gzcvfuxjr8)

## ユースケース
Service Meshのユースケースにはざっくり以下のようなものがある（他にもあるだろう）。

### 1. 動的サービスディスカバリ、ルーティング
クラウドネイティブな設計では、サービスのIPなどは常に変わりゆく前提（新しくデプロイされたり、作り直されるとサービスのIPなどは変動する）である。
また、カナリアリリースをしたい場合には、新しくリリースしたリソースに対して、アクセス全体のN%のトラフィックを流して動作保証を得たくなるだろう。

### 2. サービス間のアクセス信頼性の向上
サービス間通信でのリトライやタイムアウトなどを提供しているため、不具合時のリカバリに役立つだろう。

### 3. マイクロサービスの監視
サービス間通信のレイテンシーやエラーを監視出来るため、サービス障害の予防や原因究明などに役立つだろう。


## 参考
- [Service Mesh Ultimate Guide: Managing Service-to-Service Communications in the Era of Microservices](https://www.infoq.com/articles/service-mesh-ultimate-guide/)
- [Service mesh data plane vs. control plane](https://blog.envoyproxy.io/service-mesh-data-plane-vs-control-plane-2774e720f7fc)
- [AWS App Mesh aws/aws-app-mesh-examples](https://github.com/aws/aws-app-mesh-examples#why-use--app-mesh)
- [Envoy (Envoy proxy)、Istio とは？](https://qiita.com/seikoudoku2000/items/9d54f910d6f05cbd556d)
- [wikipedia Cloud Native Computing Foundation](https://ja.wikipedia.org/wiki/Cloud_Native_Computing_Foundation)
- [Service Meshes](https://softwareengineeringdaily.com/2020/01/07/service-meshes/)
- [Istioに入門する](https://techstep.hatenablog.com/entry/2020/12/26/112229)
