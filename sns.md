# SNS

FIFOとStandardがある

FIFOだとProtocolにSQSしか選べなかったが、StandardならEmailを初め色々選べた

## lambdaからsnsを動かす例

<https://qiita.com/tsumita7/items/bbafba094db5794d0374>

## snsとses

snsはシステム管理者など、事前にアドレスを登録しておける場合に利用する。
ユーザへの通知などアドレスを実行時に判断したい場合は、sesを使う。

https://cloud5.jp/sns-ses/
