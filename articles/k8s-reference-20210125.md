---
title: "Kubernetes入門の際に参考にした情報"
emoji: "⚓"
type: "idea"
topics: ["Kubernetes"]
published: true
---

## これは何？
仕事でKubernetesを触ることになったのですが、それまでに予習としてインプットに活用させて頂いた情報源をまとめました（分類分けがかなり雑ですが、優しく見てください）。

## （個人的な）結論
Kubernetesについて調べ始めると、Kubernetes自体が大きなエコシステムであることと、それを取り巻くいろんなツールが出てきて、「どこから始めればいいんだ...」となりがちです（少なくとも僕はなりました...）。
そこで個人的には、Youtubeなどの動画（Kubernetesは情報の行進が早いので、1年以内に配信されたものがオススメ。またハンズオンがあると尚良し。）を見てKubernetesの全体像を掴んだ後に、 [Kubernetes完全ガイド impress top gearシリーズ](https://www.amazon.co.jp/Kubernetes%E5%AE%8C%E5%85%A8%E3%82%AC%E3%82%A4%E3%83%89-impress-top-gear%E3%82%B7%E3%83%AA%E3%83%BC%E3%82%BA-%E9%9D%92%E5%B1%B1-ebook/dp/B07HFS7TDT/ref=sr_1_7?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=HY2PPELP1NG9&dchild=1&keywords=kubernetes&qid=1608947361&sprefix=kuberne%2Caps%2C280&sr=8-7) にざっと目を通す流れが良いと思いました（詳細を覚えるのではなく、頭にインデックスるを作る感じ）。その上で使うクラウドインフラ特有の事項を調べたり、Kubernetesのセキュリティや裏側の具体的な挙動について深ぼる感じです。

Kubernetesが「どんなことを目的に生まれたのか」、またそれを「どうやって実現しているか」の大枠をある程度理解してから詳細へと深ぼっていくことで、本質を見失わずに進んでいけるのかなと思います。


## Kubernetes
### 公式
- [Kubernetes](https://kubernetes.io/)

### 書籍
- [イラストでわかるDockerとKubernetes Software Design plus](https://www.amazon.co.jp/%E3%82%A4%E3%83%A9%E3%82%B9%E3%83%88%E3%81%A7%E3%82%8F%E3%81%8B%E3%82%8BDocker%E3%81%A8Kubernetes-Software-Design-plus-%E5%BE%B3%E6%B0%B8-ebook/dp/B08PNMRXKN/ref=sr_1_2?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=HY2PPELP1NG9&dchild=1&keywords=kubernetes&qid=1608947361&sprefix=kuberne%2Caps%2C280&sr=8-2)
    - 図解中心なので、特に最初に目を通しておくとその後のイメージの助けになる
- [Kubernetes完全ガイド impress top gearシリーズ](https://www.amazon.co.jp/Kubernetes%E5%AE%8C%E5%85%A8%E3%82%AC%E3%82%A4%E3%83%89-impress-top-gear%E3%82%B7%E3%83%AA%E3%83%BC%E3%82%BA-%E9%9D%92%E5%B1%B1-ebook/dp/B07HFS7TDT/ref=sr_1_7?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=HY2PPELP1NG9&dchild=1&keywords=kubernetes&qid=1608947361&sprefix=kuberne%2Caps%2C280&sr=8-7)
    - とにかく丁寧に情報がまとまっているので、困ったらまずはこの本から情報を探すようにしています
- [Kubernetesで実践するクラウドネイティブDevOps](https://www.amazon.co.jp/Kubernetes%E3%81%A7%E5%AE%9F%E8%B7%B5%E3%81%99%E3%82%8B%E3%82%AF%E3%83%A9%E3%82%A6%E3%83%89%E3%83%8D%E3%82%A4%E3%83%86%E3%82%A3%E3%83%96DevOps-John-Arundel/dp/4873119014/ref=sr_1_5?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=HY2PPELP1NG9&dchild=1&keywords=kubernetes&qid=1608947361&sprefix=kuberne%2Caps%2C280&sr=8-5)
    - 未読...
- [Docker/Kubernetes開発・運用のためのセキュリティ実践ガイド ](https://www.amazon.co.jp/dp/4839970505/?coliid=I1396KIJMBW9S3&colid=2ZU13XDYG1HXY&psc=1&ref_=lv_ov_lig_dp_it)
    - 未読...


### ブログ
- [攻撃しながら考えるKubernetesセキュリティ](https://speakerdeck.com/fujiihda/considering-kubernetes-security-while-attacking-2)
- [Kubernetes: Pod Security Policy によるセキュリティの設定](https://qiita.com/tkusumi/items/6692af743ae03dc0fdcc)
- [Kubernetesネットワーク 徹底解説](https://zenn.dev/taisho6339/books/fc6facfb640d242dc7ec)
- [整理しながら理解するKubernetesネットワークの仕組み](https://speakerdeck.com/hhiroshell/kubernetes-network-fundamentals-69d5c596-4b7d-43c0-aac8-8b0e5a633fc2)
- [2020年にKubernetse関連で取り組んだことまとめ](https://bells17.medium.com/2020-kubernetse-4771e660a174)
- [Kubernetesのコードリーディングをする上で知っておくと良さそうなこと](https://bells17.medium.com/things-you-should-know-about-reading-kubernetes-codes-933b0ee6181d)
- [Kubernetes道場 Advent Calendar 2018](https://qiita.com/advent-calendar/2018/k8s-dojo)
- [Run Kubernetes Production Environment on EC2 Spot Instances With Zero Downtime: A Complete Guide](https://medium.com/riskified-technology/run-kubernetes-on-aws-ec2-spot-instances-with-zero-downtime-f7327a95dea)
- [Kubernetes: 詳解 Pods の終了](https://qiita.com/superbrothers/items/3ac78daba3560ea406b2)
- [ゼロから始めるKubernetes Controller](https://speakerdeck.com/govargo/under-the-kubernetes-controller-36f9b71b-9781-4846-9625-23c31da93014)
- [Kubeletから読み解くKubernetesのコンテナ管理の裏側/Kubelet & Containers](https://speakerdeck.com/bells17/kubelet-and-containers)
- [SRE / DevOps / Kubernetes Weekly Reportまとめ#51(2021/1/17~1/22)](https://hakobiya.hatenablog.com/entry/2021/1/18/Weekly-Report-51)


### 動画
- [【EKSWorkshop】EKSやkubernetes周辺を効率よく学ぶのにオススメなチュートリアル集](https://dev.classmethod.jp/articles/eks-workshop/)


## AWS(EKS)
Kubernetesに対する基礎的な知見を備えた上で、IAMとKubernetes側のRBACとの関係や、EKS側で何をどこまで自動でやってくれるのかを把握するのが重要かなぁと思いました。

### 公式
- [Amazon EKS とは](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/what-is-eks.html)
- [Amazon EKSユーザーガイド](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/eks-ug.pdf)
- [Amazon EKS ワーカーノードの謎を解くクラスターネットワーク](https://aws.amazon.com/jp/blogs/news/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/)
    - 「EKSってノードどうなってんの？」って思ったときに参考にしました
- [weaveworks/eksctl](https://github.com/weaveworks/eksctl/tree/master/examples)
    - ハンズオンや技術書では具体的なYAMLの情報までは書いてないので、「え、それってどうやってるん？その際に〇〇とか指定せんでええの？」みたいな疑問はこちらを参考にしました

### 書籍
- [Kubernetes on AWS～アプリケーションエンジニア 本番環境へ備える](https://www.amazon.co.jp/Kubernetes-AWS-%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%8B%E3%82%A2-%E6%9C%AC%E7%95%AA%E7%92%B0%E5%A2%83%E3%81%B8%E5%82%99%E3%81%88%E3%82%8B-%E4%BC%9A%E6%BE%A4/dp/4865942351/ref=tmm_pap_swatch_0?_encoding=UTF8&qid=1611043531&sr=8-4)

### ブログ
- [Managed Node Groupsを用いてTerraformでEKSクラスタをシュッと構築する](https://tech.griphone.co.jp/2020/01/22/eks-managed-node-groups/)
    - EKSにおけるnodeの作り方って具体的にどうなんだろうという点について調べた際に参考にしました
- [TechWorld with Nana](https://www.youtube.com/channel/UCdngmbVKX1Tgre699-XLlUA)
    - 動画で図解つきの説明なので、軽く観ておくと全体像のキャッチアップに便利

## 動画
- [Amazon EKS Workshop](https://www.eksworkshop.com/)
    - AWS公式が出しているEKSのハンズオンです
    - Kubernetesの全体像はざっくり分かったけど、EKS使う際にこれってどうなんやろ？が動画で解説されています

## 所感
Kubernetesを勉強して思ったこととしては、高度に抽象化されていながらも結局ちゃんと運用するには裏側の挙動やインフラの知見は必須だなと感じました。
クラウドインフラに任せられる部分も沢山ありますし、その割合は年々増えていますが、それらを構築して運用するのは我々利用者なので、正しい理解の元に運用しないと結局はイケてるフレームワークで辛い運用をしている感じになるかなと思います。

というわけで日々精進です。
