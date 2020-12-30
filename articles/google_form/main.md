---
title: "GoogleフォームからBacklogへ課題を自動登録する"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GoogleForm", "Backlog"]
published: true
---

*Qiitaに記載しているものをこちらに移行しました。

みなさんの会社や組織は、エンジニア部門と他部門(エンジニアが開発したシステムを社内で使う人)との連携は上手くいっていますか？
私の組織はボチボチです。
今の組織に入りたての頃にカオスだった対応フローを改善し、程々波に乗ってきたので、試したことを公開します。
ご参考になれば幸いです。


## 当初の状況
私が現在所属している組織では、私の入社当初は下記のようなカンジでした。

```
1. CSチームがカスタマー対応中にシステムのバグを発見する
2. バグを見つけた人がエンジニアチームのところに直接駆け込む
3. その時その場に座っていたエンジニアがバグの対応に追われる
4. 以降、バグが枯れるまで①~③の無限ループ（私の所属するチームでは1日最大10回くらい発生していました...）
```

こんなループになると、少なくとも下記のような問題が起こりました。

```
1. バグのクリティカル度合いに関係なく対応する必要がある
  - 誰しもが自分の抱えている事象を重要だと言う（心理的には十分理解できる）
  - エンジニア側としては直接駆け込まれるとNoと言いにくい(特に相手がおエライさんの場合)
2. エンジニアの状況（もくもく開発中、休憩中）に関係なく、相手の都合に合わせて対応を迫られる
  - 残念ながら必ずしも相手のことを考慮できる人ばかりではない
3. 口頭のやりとりのため対応記録が残らない
  - 改善の余地はあるが、緊急対応では往往にしてやり取りを残している余裕はない
```

## 解決策
そもそもバグを大量に作るエンジニアが悪い、という点については今回は目を瞑って頂きまして、、 
このループをなんとかできないかなーと思って調べていたところ、Backlogでメールよる課題登録ができることを知りました。
[Backlog メールによる課題登録](https://backlog.com/ja/help/adminsguide/add-issues-via-email/userguide2306/)

文面ベースなら割と客観的にバグのクリティカル度が分かるし、自分一人で抱え込まずにチーム内で対応方針を決めやすくもなるので、これを使ってバグ報告フローを下記のように整備しました。

![GFからGAS経由でBacklogへ課題登録](https://storage.googleapis.com/zenn-user-upload/jfbspij8h193of8uoro66fr9ykov)

## 具体的な手順
### Backlogでメールによる課題登録を設定する
- [Backlog 課題登録用メールアドレスの追加](https://backlog.com/ja/help/adminsguide/add-issues-via-email/userguide2312/)を参考に、- `課題登録用メールアドレスの追加` から、送信対象のメールアドレスを作成します。
- 課題登録時の通知は、ひとまず運用者全員にすると見逃しが無くなると思います。

### Googleフォームを作る
- 何はともあれ、[Google フォーム](https://www.google.com/intl/ja_jp/forms/about/)から報告してもらいたい内容を設定します。

#### 気をつけるべきポイント
- 基本的に記入項目を必須入力欄にする
  - 少なくとも白紙で報告されることはなくなる
- 各質問の補足欄に回答例や特に記載してほしい内容を記載する
  - 不具合報告に慣れていない方でも、サンプルを見れば最低限の水準の報告書が出来上がる

### GoogleフォームのGoogle Apps Scriptを設定する
- 作成したGoogleフォームの右上のボタンから`スクリプトエディタ`を選択すると、Google Apps Scriptのエディタが開きますので、エディタに下記のスニペットのような感じで記載します
  - 送信された回答を上から順に一つずつ取得しているだけなので、イケてるやり方とは到底言えませんが...

```javascript
function sendTroubleEmail(e) {

  // 送信された各回答と対応する質問名を格納する
  var itemResponses = e.response.getItemResponses(); 

  /*
  * メール本文
  * Googleフォームに設定した質問項目と対応させてください
  */
  // タイトル
  const contentOfTitle = itemResponses[0].getResponse();
  // 報告元の部署名/氏名
  const contentOfRequester = itemResponses[1].getResponse();
  // 発生日
  const contentOfOccuredDate = itemResponses[2].getResponse();
  // 対応緊急度 
  const contentOfPriority = itemResponses[3].getResponse();
  // 優先度の理由
  const contentOfReasonOfPriority = itemResponses[4].getResponse();
  // 対象システム
  const contentOfTargetSystem = itemResponses[5].getResponse();
  // 現在の状態
  const contentOfCurrantSituation = itemResponses[6].getResponse();
  // 再現率
  const contentOfTroubleReproductionRate = itemResponses[7].getResponse();
  // 発生した画面のURL
  const contentOfTargetURL = itemResponses[8].getResponse();
  // 不具合に巻き込まれた会員情報
  const contentOfTargetMemberInformation = itemResponses[9].getResponse();
  // 不具合説明
  const contentOfTroubleDetail = itemResponses[10].getResponse();
  // 期待される結果
  const contentOfIdealResult = itemResponses[11].getResponse();
  // 不具合再現手順
  const contentOfTroubleReproductionProcedure = itemResponses[12].getResponse();
  // 自由記述欄
  const contentOfFreeDescription = itemResponses[13].getResponse();
    
  /*
  * 質問項目
  */
  const TITLE = "不具合報告";
  const REQUESTER = "報告元の部署名/氏名";
  const OCCURED_DATE = "発生日";
  const PRIORITY = "対応緊急度";
  const REASON_OF_PRIORITY = "優先度の理由";
  const TARGET_SYSTEM = "対象システム";
  const CURRENT_SITUATION = "現在の状態";
  const REPRODUCTION_RATE = "再現率";
  const TARGET_URL = "発生した画面のURL";  
  const MEMBER_INFORMATION = "不具合に巻き込まれた会員情報";
  const TROUBLE_DETAIL = "不具合説明";
  const IDEAL_RESULT = "期待される結果";
  const REPRODUCTION_PROCEDURES = "不具合再現手順";
  const FREE_DESCRIPTION = "自由記述欄";

  /* 
  * メール送信先アドレス
  * Backlogの課題登録用メールアドレスを使用します。
  */ 
  const TO = "hogehogefugapiyo@i4.backlog.jp";
  
  // メール件名
  const SUBJECT = "【" + TITLE + "】"+ contentOfTitle;
  
  //メールの内容(backlogの課題詳細に表示される)
  var body = "【" + REQUESTER + "】" + "\n" + contentOfRequester + "\n\n" +
    "【" + OCCURED_DATE + "】" + "\n"  + contentOfOccuredDate + "\n\n" +
    "【" + PRIORITY + "】" + "\n"  + contentOfPriority + "\n\n" +
    "【" + REASON_OF_PRIORITY + "】" + "\n"  + contentOfReasonOfPriority + "\n\n" +
    "【" + TARGET_SYSTEM + "】" + "\n"  + contentOfTargetSystem + "\n\n" +
    "【" + CURRENT_SITUATION + "】" + "\n"  + contentOfCurrantSituation + "\n\n" +
    "【" + REPRODUCTION_RATE + "】" + "\n"  + contentOfTroubleReproductionRate + "\n\n" +
    "【" + TARGET_URL + "】" + "\n"  + contentOfTargetURL + "\n\n" +
    "【" + MEMBER_INFORMATION + "】" + "\n"  + contentOfTargetMemberInformation + "\n\n" +
    "【" + TROUBLE_DETAIL + "】" + "\n"  + contentOfTroubleDetail + "\n\n" +
    "【" + IDEAL_RESULT + "】" + "\n"  + contentOfIdealResult + "\n\n" +
    "【" + REPRODUCTION_PROCEDURES + "】" + "\n"  + contentOfTroubleReproductionProcedure + "\n\n" +
    "【" + FREE_DESCRIPTION + "】" + "\n"  + contentOfFreeDescription + "\n\n";

  /*
  * メール送信実行
  */
  MailApp.sendEmail(
    TO,
    SUBJECT,
    body
  );
}
```

- 作成した関数をいつどの様に呼び出すかを設定する必要があるので、`設定`→`現在のプロジェクトのトリガー`を開き、作成した関数を設定し、`フォームからフォーム送信時に通知`と設定します。
  - これを設定しないと、Googleフォームから送信ボタンを押しても関数が実行されず、メールが飛びません...

![トリガー](https://storage.googleapis.com/zenn-user-upload/kermatdxne0j34ijbnl7yksef3a5)

設定はこれにて完了！

Googleフォームから課題を記載し、Backlogに課題登録されることを確かめてみましょう！
下図のように、Googleフォームに記載した内容がBacklogの課題本文に記載されていたら動作確認完了です。

![結果](https://storage.googleapis.com/zenn-user-upload/49y1oyukgsl6ka1xhh7in7sm4fkf)

## その他
- 既存のワークフローを変更する場合には、プログラムを書く前に関係各所に周知しましょう。
- また、その際にはフローを変えることで皆んなにどんなイイコトがあるかを明示しましょう
  - 「またエンジニアが自分の都合の良い風に言い始めたよー」とかならないように
  - 感情の対立が一番厄介です...
