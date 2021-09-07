# Step Funcitons

全体をステートマシンとよぶ

ASL(Amazon States Language)で書く

Lambda/DynamoDB/SNS/SQSなどを連携できる
（Activityという機能を用いて、サーバやコンテナ上に配置したアプリケーションをStepFuncitonsから呼び出すこともできる）

## ステートマシンを実行する方法

* API呼び出し
    StartExecution

* マネージメントコンソールから手動で実行

* CloudWatch Eventsを使用する
    (EventBridgeを利用する)

* SDKを使用する

* タスク内でネストして実行する

##　情報をサーバレスに伝える

dataを持ちまわることができる。

```txt
入力データ
{
    "name" : "narushima",
    "num" : "3"
}

後続タスクに渡す方法
{
    "Type": "Task",
    "InputPath": "$num", # 入力データのnumを後続タスクに渡す
    "Resource" ": "arn:aws:lambda~~~~" # lambdaを指定する
}
```

step functionsにおけるlabmda error
<https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/bp-lambda-serviceexception.html>

resultPathについて
https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/input-output-resultpath.html

Choiceを利用することでJSONのあるパラメータの値に応じて分岐先を決定できる。
これを利用して業務例外のハンドリングができるのでは。
<https://dev.classmethod.jp/articles/aws-step-functions-states-choice/>

stepfuncitonsのパラメタについて分かりやすい記事
<https://dev.classmethod.jp/articles/step-functions-parameters/>

要は...
stepfuncionsはJSON情報を持ちまわって処理をこなすもの。


stepfunctionsは、長時間実行されるワークフローなどに利用するのが良い。
一方でSQSは、短期的に大量に実行されるようなシステムに強い。
