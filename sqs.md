# SQS

## プロデューサーとコンシューマー

## キューの種類

標準キュー: 入れた順番と出る順番に規則性はない。また1つ入れたものが2回以上出てくることもある。アプリケーション側で冪等性を担保する必要がある。

FIFOキュー: 順番と出る回数が担保されている。

#### キューの使い分け

バッチ処理の順番を確実に指定したい場合、FIFO
バッチ処理の順番を

## ロングポーリングとショートポーリングの違い

<https://docs.aws.amazon.com/ja_jp/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html>

## 可視性タイムアウト

    あるコンシューマがメッセージをポーリングして処理している時に、他のコンシューマが触れない状態。いわばロック状態。
    処理が成功したらコンシューマによってメッセージが削除され、処理が失敗したら再度触れる状態に戻る。

## デッドレターキュー

公式の説明
<https://docs.aws.amazon.com/ja_jp/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html>

デッドレターキューとは

    正常に処理できないメッセージが送られるキュー
        例えばコンシューマーが5回以上メッセージを受信してもメッセージが削除されない場合に、メッセージが送られてくる

使い方

    SQSを2つ作って、片方のSQSをキュー、もう片方をデッドレターキューとして使う
    キューにてデッドレターキューを指定する。またデッドレターキューに渡すまでのポーリング回数(最大受信回数)を指定する。
    （ポーリングされた後うまく処理されたら削除されるはずなのに、それが何回もされないということは問題のあるメッセージということ）

制約

    * FIFOキューのデッドレターキューはFIFOである必要があり、標準キューのデッドレターキューは標準キューである必要がある
    * キューとデッドレターキューは同じリージョンに存在する必要がある
    * デッドレターキューには保存期間があり、３日などで削除する。

嬉しいこと

    正しく処理できなかったメッセージだけが格納されるので、失敗処理を分析しやすい

    * デッドレターキューに配信されたメッセージに関するアラームを設定する。
    * ログを調べて、メッセージがデッドレターキューに配信される原因となった可能性のある例外を特定する。
    * デッドレターキューに配信されたメッセージの内容を分析し、ソフトウェアに関する問題や、プロデューサーまたはコンシューマーのハードウェアに関する問題を診断する。
    * コンシューマーがメッセージを処理するために十分な時間が設定されているかどうかを調べる。

考えたこと（以下を整理すればいいのでは）

    デッドレターキューに入っただけではどんな例外かわからないので、クライアントにどんな通知をするべきかわからない。
    例外がシステム例外か業務例外か調べて、適切なメッセージを通知する必要がある。

    * デッドレターキューにどんなメッセージが格納されるのか
    * 格納されたメッセージをどうやって処理してクライアントに返せば良いのか

### 今となってはデッドレターキューを使う必要ない？

    lambdaの宛先指定機能を使えばもうデッドレターキューはいらない？
    lambdaには処理成功時と、処理失敗時の連携先を指定できる。さらに含められる情報がデッドレターキューよりも多い。
    <https://dev.classmethod.jp/articles/lambda-dlq-vs-destinations/>

    DLQとDestinationsのメリデメ
    
| | DLQ | Destinations | 
| --- | --- | --- |
| 情報量 | △ | ○ |
| リトライのしやすさ | ○ | × | 
| 連携対象の自由度 | ○ | ○ | 
| 実装量 | ○ | ○ |

Destinationsは、SQS, SNS, Lambda, EventBridgeに連携可能

**TODO**

1. Destinationsの場合のリトライ方法の整理が必要

2. Destinationsを使う時、lambdaの成功、失敗時のsqsからデータが消えるかいなか。失敗時、destinationsに連携しつつsqsにメッセージが残ってしまうのでは。などを調べてまとめる。

3. Destinationsを使うとき、lambda以外の選択肢を利用する方法

以下、連携処理機能の組み合わせ
（連携先はひとまず全部lambdaにした）

1. ma-narushima-lambda-destination
2. ma-narushima-destination-onfailure-receiver
3. ma-narushima-destination-onsuccess-receiver

参考
<https://qiita.com/kojiisd/items/efcb2ac3d5cc176534ba>

lambda-destinationを成功にする非同期リクエスト

```
aws lambda invoke --cli-binary-format raw-in-base64-out  --function-name ma-narushima-lambda-destination --invocation-type Event --payload '{ "Success": true }' response.json
```

lambda-desitinationを失敗にする非同期リクエスト

```
aws lambda invoke --cli-binary-format raw-in-base64-out  --function-name ma-narushima-lambda-destination --invocation-type Event --payload '{ "Success": true }' response.json
```

## コンシューマーの種類

    lambda
    fargate
    EC2

    など自由に選べる
    それぞれの実装方法の違い

## 参考

クラソル
<https://business.ntt-east.co.jp/content/cloudsolution/column-135.html>

## 単語

スロットリング
    一定時間内に受信可能なリクエスト数を制限し、制限を上回るリクエストがなされた際には受信を拒否しエラーコードを返却すること。

## どうすべきか迷うこと

dlqに格納されたメッセージ、なぜsqsにある時に処理できなかったのか理由がわからない。
sqsに最初に格納されたbodyしかないものになっていて、エラー情報が含まれていない。

CloudWatch Logsを使用して失敗原因を追跡できる？
<https://docs.aws.amazon.com/ja_jp/sns/latest/dg/sns-dead-letter-queues.html>

## SQSとLambdaの連携

Lambdaからポーリングするのと、LambdaのトリガにSQSを指定しておく手がある。

LambdaのトリガにSQSを指定するのは実はロングポーリング
https://bluepixel.hatenablog.com/entry/2020/05/06/070732

lambdaとsqsのトリガ
https://d1.awsstatic.com/serverless-jp/20181022-LambdaxSQS.pdf
