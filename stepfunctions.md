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
