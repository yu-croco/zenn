---
title: "Argo CDにOSSコントリビュートしたときの流れ"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ArgoCD"]
published: false
---

# コレは何？
Argo CDというKubernetes関連のCDツールでOSSコントリビュートしたので、記念に流れをまとめました。
https://github.com/argoproj/argo-cd

# 経緯
Argo CDを構成する一つのargocd-repo-serverで、logFormatをJSON指定しているのに一部のログが何故かTEXT形式で出力されることを見つけました。

```
$ kubectl logs -n argocd argocd-repo-server-6b8f4449bb-cn8tb
{"level":"info","msg":"Generating self-signed gRPC TLS certificate for this session","time":"2021-05-29T00:50:34Z"}
{"level":"info","msg":"Initializing GnuPG keyring at /app/config/gpg/keys","time":"2021-05-29T00:50:34Z"}
{"dir":"","execID":"ZaGQl","level":"info","msg":"gpg --no-permission-warning --logger-fd 1 --batch --generate-key /tmp/gpg-key-recipe337068804","time":"2021-05-29T00:50:34Z"}
time="2021-05-29T00:50:34Z" level=info msg=Trace args="[gpg --no-permission-warning --logger-fd 1 --batch --generate-key /tmp/gpg-key-recipe337068804]" dir= operation_name="exec gpg" time_ms=215.4307
{"level":"info","msg":"Populating GnuPG keyring with keys from /app/config/gpg/source","time":"2021-05-29T00:50:34Z"}
...
```

Argo CDは argoproj/argo-helmを使用しており、以下のような設定で指定していました。
https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd

```yaml
# Chart.yaml
apiVersion: v2
name: sample-argocd
description: sample argocd
type: application
version: 0.1.0
dependencies:
  - name: argo-cd
    version: 3.2.4
    repository: https://argoproj.github.io/argo-helm

---
# values.yaml
argo-cd:
  controller:
    logFormat: json
  server:
    logFormat: json
  repoServer:
    logFormat: json
```

[githubのコード](https://github.com/argoproj/argo-helm/blob/master/charts/argo-cd/values.yaml#L805)を見ても、 「`logFormat: json` で指定すればJSON出力されるよ」としか書いてないので不具合だなーと思いました。


# 流れ
不具合っぽいなーと思ってからの流れとしては以下の感じです。

## 1. ISSUEが出てないか確認
他の誰かも同じような不具合を踏んでいるケースが多いので、ひとまずISSUEに挙がっていないか確認しました。
確認したところ、closedですがそれらしいISSUEが見つかりました。 中身を見ると確かに同様の症状ですが、現在のバージョンでも発生するのでまだ改修され切ってないまま完了扱いとなっているようです。

https://github.com/argoproj/argo-cd/issues/5641

## 2. ISSUEを出す
現状では同様の症状でISSUEが出ていないようでしたのでISSUEを出しました。
パッと調査しても全然原因箇所が分からなかった（該当のログだけで実装元に遡るのが難しすぎた...）ので、ひとまずArgo Projectの関係者に届けばいいなというノリで出しました。
https://github.com/argoproj/argo-cd/issues/6234

ISSUEを書く際には、Contribution guideやISSUEのテンプレートに色々とルールが書いてあるので、それをチェックしながら書きました。
また、過去に出されているISSUEなども参考にしました。

https://argo-cd.readthedocs.io/en/latest/developer-guide/contributing/

## 3. 原因を教えてもらう
Argo Projectの関係者から、原因箇所と発生の理由を教えてもらいました（感謝感激！）。
外部のライブラリを使用する際にlogger設定を引数として渡す必要があるのですが、その際にユーザーが指定した設定値を使用せず、単にloggerをinitして渡していたようです。

![](https://storage.googleapis.com/zenn-user-upload/b98b66ed4a0191622e0072b9.png)

## 4. 対処方法を考える
原因はわかったので対処方法を考えました。

まずは、原因となってるloggerに現在の設定値を取得するメソッドがあるかを確認しました。logger側でメソッドが生えていればそれを使うだけで解決できそうなので簡単だなと思いました。
loggerではLogrusというものが使われていたので、ソースコードをforkして内部をウロウロしました。
https://github.com/sirupsen/logrus

logLevelに関しては[GetLevel](https://github.com/sirupsen/logrus/blob/master/logger.go#L363)というメソッドがあったので簡単に取得できそうでしたが、logFormatに関してはメソッドが生えていなさそうでした...。
そのため、logrusに対して現在設定しているlogFormatを取得できるメソッドを作成してPRを出そうとしました。

が、Logrusは既にメンテナンスモードに入っており、新機能などは受け付けていませんでした。。

> Logrus is in maintenance-mode. We will not be introducing new features. It's simply too hard to do in a way that won't break many people's projects, which is the last thing you want from your Logging library (again...).

Argo Projectの関係者からは、「知る限りではLogrusはメンテナンスモードに入ってるから機能拡張のPRは受け付けてもらえなさそうだね。でも、Argo CDの内部で自前でloggerの設定情報を取得する機構を用意することはできそうだね。」というアドバイスを頂きました（感謝感激v2！）。

loggerの設定はライブラリのメソッドからは取得できないため、以下のどちらかの選択肢が取れると考えました。

- 最初にユーザーから渡されてくる設定値を引数に混ぜて、該当の処理までひたすら引数で渡していく
- 何らかの方法でグローバルに値を取得する

> 最初にユーザーから渡されてくる設定値を引数に混ぜて、該当の処理までひたすら引数で渡していく

既存のコードを見た限りでは、既存の引数として使われている値に相乗りすることは難しそうだったので、純粋に引数を増やすことになりそうでした。
不具合の該当箇所はArgo CD内の複数箇所に散らばっており、単純に引数を増やすと修正箇所が非常に多くなってしまいます。修正したい内容に対して影響範囲があまりにも大きく、良い改修とは言えないなと思い採用しませんでした。


> 何らかの方法でグローバルに値を取得する

ArgoCDを起動する際にユーザーからloggerの設定情報が渡されてくるため、メインとして使用されているlogger初期化処理のところで現在の設定値をグローバルな値として設定すれば、必要な箇所から自由に取得できると考えました。
いわゆるグローバル変数みたいに用意するのはトリッキーですし、Golang特有の機能も見つからなかったため環境変数として用意することにしました。

## 5. 修正コードを書く
ArgoCDの起動時にユーザーから渡されてくるloggerの設定情報を環境変数にセットするように修正しました。
既存のコードには `SetLogFormat` と `SetLogLevel` があり、ここでアプリケーションのベースとして使われているloggerの設定がされています。
これらのメソッドはサーバー起動の際にのみ使用されているため基本的に安全に値をセットできると判断し、ここで環境変数にもlogger設定をセットする処理を追加しました。

```go
func SetLogFormat(logFormat string) {
	switch strings.ToLower(logFormat) {
	case utillog.JsonFormat:
		os.Setenv(common.EnvLogFormat, utillog.JsonFormat)
	case utillog.TextFormat:
		os.Setenv(common.EnvLogFormat, utillog.TextFormat)
	default:
		log.Fatalf("Unknown log format '%s'", logFormat)
	}

	log.SetFormatter(utillog.CreateFormatter(logFormat))
}

func SetLogLevel(logLevel string) {
	level, err := log.ParseLevel(logLevel)
	errors.CheckError(err)
	os.Setenv(common.EnvLogLevel, level.String())
}
```

セットしたlogger設定を参照する側のコードは以下のように実装しました。
環境変数からlogger設定を取得して新しいloggerを作成します。これを呼び出すことでユーザーが最初に指定した設定値を引き継ぐことができます。

```go
const (
    JsonFormat = "json"
    TextFormat = "text"
)

func NewWithCurrentConfig() *logrus.Logger {
	l := logrus.New()
	l.SetFormatter(CreateFormatter(os.Getenv(common.EnvLogFormat)))
	l.SetLevel(createLogLevel())
	return l
}

func CreateFormatter(logFormat string) logrus.Formatter {
	var formatType logrus.Formatter
	switch strings.ToLower(logFormat) {
	case JsonFormat:
		formatType = &logrus.JSONFormatter{}
	case TextFormat:
		if os.Getenv("FORCE_LOG_COLORS") == "1" {
			formatType = &logrus.TextFormatter{ForceColors: true}
		} else {
			formatType = &logrus.TextFormatter{}
		}
	default:
		formatType = &logrus.TextFormatter{}
	}

	return formatType
}

func createLogLevel() logrus.Level {
	level, err := logrus.ParseLevel(os.Getenv(common.EnvLogLevel))
	if err != nil {
		level = logrus.InfoLevel
	}
	return level
}
```

## 6. PRを出す
コードを修正したので[Contribution guide](https://argo-cd.readthedocs.io/en/latest/developer-guide/contributing/)や過去にマージされているPRを参考にしてPRを出しました。
既存のコンテキストに対する理解が浅いため自分の見解を記載するようにしました。

https://github.com/argoproj/argo-cd/pull/6301#event-4815213597

## 7. review＆approveをもらう
reviewして頂き、無事にapproveとなりました！
![](https://storage.googleapis.com/zenn-user-upload/e7f15f79fc4cf78eeff7f126.png)

# 雑感
OSSの不具合を踏んで、ISSUEを出して、修正PRを出して、approveもらうまでの一通りを経験できる貴重な体験だった。
これを期にOSSコントリビュート活動を続けていきたい！
