# Event

状態の変化に関する情報をサービス館で共有するためのメカニズム

> SQS使わなくてもEventBridgeで非同期処理できる？

* eventのサンプル
<https://pages.awscloud.com/rs/112-TZM-766/images/EventBridge_overview.pdf>より

```json
{
    "version": "0",
    "id": "6a7e8feb-b491-4cf7-a9f1-bf3703467718",
    "detail-type": "EC2 Instance State-change
    Notification",
    "source": "aws.ec2",
    "account": "111122223333",
    "time": "2017-12-22T18:43:48Z",
    "region": "us-west-1",
    "resources": [
    "arn:aws:[region]:[account]:instance/i1234567890abcdef0"
    ],
    "detail": {
        "instance-id": "i-1234567890abcdef0",
        "state: "terminated"
    }
}
```

source: イベントの送信元
detail-type: イベントの型
detail: イベントの詳細情報
resources: このイベントに関連する識別子の配列

（ここからちょっと合ってるか心配）
AWS間のサービスの連携は、イベントを使って行う
（その他に伝えられるものはない。。？）

イベントのdetailに情報を追加できない場合、デフォルトで通知される内容以上を伝えることができない。。？
CloudWatchLogsなどは、detailも含めて情報を追加できないよね？
それをきっかけにあるLambdaなどを動かしても、エラーメッセージなどの情報をLambdaが取得する方法はない。

`イベントの書き換え`

イベントのdetailに情報を追加していったり、更新しながら情報を連携していく。

イベントの書き換えは、イベントメッセージトランスフォーマー

## Step FunctionsとAmazon EventBridgeのサービス統合 (2021年6月)

Step Functionsは、サーバレスオーケストレーションワークフローの構築

EventBridgeは、AWSサービスやSaaS, 独自アプリなどでイベントのルーティングができる
