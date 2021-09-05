# サーバレス

参考記事
<https://news.mynavi.jp/itsearch/article/devsoft/4316>

## 基礎知識

* lambdaには、jarを載せる
* jarにはmaven-shade-pluginを利用して依存jar全てをまとめる
* jar内のclassのhandleRequestメソッドが動く（AWS Lambdaのエントリーポイントになる）

## AWS lambdaで動くアプリを作る

Plain Javaで作る場合と、SpringBootを使って作る場合で異なる。
シンプルな実装ならPlain Javaの方が、楽に作れるし、起動も早い。

### Plain Javaで作る場合

参考<https://www.baeldung.com/java-aws-lambda>

`基礎知識`

* AWS lambdaのコンソールから、エントリポイントを指定する

    **クラスのフルパス::handleRequest**　とする（メソッド参照の書き方）

### SpringBootを使う場合

`基礎知識`

* @Injectではなく、X.getBean()でcomponent scanされたクラスを利用する

`手順`

1. spring initializerでプロジェクトを作る

    * java11
    * maven
    * その他は特に設定なし

2. pomにlambda用のdependencyを追加する

    ```xml
    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-lambda-java-core</artifactId>
        <version>1.1.0</version>
    </dependency>
    ```

3. pomにmaven shadeプラグインを追加する

    ```xml
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.2.4</version>
        <configuration>
            <createDependencyReducedPom>false</createDependencyReducedPom>
        </configuration>
        <executions>
            <execution>
                <phase>package</phase>
            <goals>
                    <goal>shade</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
    ```

4. handleRequestメソッドを実装しクラスを作る

    ```java
    package com.example.lambdasample;

    import com.amazonaws.services.lambda.runtime.Context;

    public class LambdaMethodHandler{
        public String handleRequest(String input, Context context){
            context.getLogger().log("Input: " + input);
            return "Hello World - " + input;
        }
    }
    ```

5. mvnでビルドする

    ```shell
    mvn clean package shade:shade
    ```

    出来上がるのは普通のjarとprefixにoriginalとついたjarがある。
    全ての依存関係を含んだのは普通のjar(originalがつかない方)

    理由
        最初に普通にjarを作った後、shadeがその名前のprefixにoriginalとつけて、
        新規に依存jar全てを含んだjarを作るため。

## 例外ハンドリング

lambdaから受け取れる情報<https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/java-exceptions.html>

Lambdaの関数内部から別サービスを呼び出すことができる？
もしできるなら例外をcatchしたらそれに対応したLambdaなどを呼べばいい

lambdaの言語別のsdkでinvokeをすればいい。
invokeするのと、stepfuncitonsを比較する？
（invokeと違ってコードから例外処理を排除できるとか？）

あとinvokeも同期処理する方法と、非同期処理する方法があって、多分Lambdaは実行時間を減らしたいから非同期がいいんだろうけど、そしたら非同期処理の更なる例外はどうするのか。（それは全部システム例外として良いか）
非同期の場合、呼び出して202が返ってくるかな？

参考

公式
<https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/API_Invoke.html>

わかりやすく整理した記事（Python）
<https://qiita.com/ume1126/items/8170a10fad6b21f0f54a#--%E5%90%8C%E6%9C%9F%E5%87%A6%E7%90%86-requestresponse>

Pythonだったらboto3を使えばいい

```py
import boto3

def lambda_handler(event, context):
    response = boto3.client('lambda').invoke(
        FunctionName='実行したいlambdaの名称',
        InvocationType='RequestResponse',
        Payload=Payload
    )

```

`FunctionName`で実行したlambdaの名称を指定する

`invocationTypeで同期と非同期を変更できる`
非同期: Event
同期: RequestRestponse

動作確認: DryRun

`Payload`で後続処理に伝えるeventに含める情報を記述できる

```py
Payload = '{\"message\":' + '業務例外が発生しました。氏名が記入されていません。' + '}'
```

## Roleの設定

Lambdaから別のLambdaをinvokeするには、`InvokeFunction`が必要

```json
{
    "Effect": "Allow",
    "Action": [
        "lambda:InvokeFunction"
    ],
    "Resource": "*"
}
```

コンソールからポリシー付与する場合
<https://took.jp/post-1037/>


## 実装サンプル

実際に呼び出したファンクションと、呼び出されたファンクション

呼び出し側

```py
import boto3

def lambda_handler(event, context):
    response = boto3.client('lambda').invoke(
        FunctionName = 'ma-narushima-invoked-function',
        InvocationType = 'RequestResponse',
        Payload = '{\"message\":' + '\"BusinessException\"' + '}'
    )
    output = response['Payload'].read().decode('utf-8')　# 呼び出し先からのresponseを受け取る
    print(output)
```

呼び出され側

```py
def lambda_handler(event, context):
    if 'message' in event:
        print(event['message']) # 受け取った情報をログに出力
        return 'message {' + event['message'] + '} was received.'
    else:
        return "message not exists."    

```

ログ

呼び出し側のログ

```txt
Test Event Name
test

Response
null

Function Logs
START RequestId: a0fa349f-5e4b-4c0d-9687-a28fe9091f9a Version: $LATEST
"message {BusinessException} was received."
END RequestId: a0fa349f-5e4b-4c0d-9687-a28fe9091f9a
REPORT RequestId: a0fa349f-5e4b-4c0d-9687-a28fe9091f9a	Duration: 390.24 ms	Billed Duration: 391 ms	Memory Size: 128 MB	Max Memory Used: 78 MB

Request ID
a0fa349f-5e4b-4c0d-9687-a28fe9091f9a
```

呼び出され側のログ

```txt
START RequestId: b7e37b1f-8b43-4e27-8efd-84bf35466a45 Version: $LATEST
BusinessException
END RequestId: b7e37b1f-8b43-4e27-8efd-84bf35466a45
REPORT RequestId: b7e37b1f-8b43-4e27-8efd-84bf35466a45	Duration: 1.05 ms	Billed Duration: 2 ms	Memory Size: 128 MB	Max Memory Used: 51 MB	Init Duration: 127.19 ms	
```

## さらにちゃんと例外ハンドリングする実装例

呼び出し側の実装

業務例外が発生したら、クライアントがすべき正しい操作をmessageとして例外に埋め込む。
それをクライアント通知処理をするための後続処理に連携する（ここではlambdaを噛ませているが、直接snsを使ってもいい）

TODO: クライアントの情報をどうやって得る？
SQSにはユーザIDを入れ、それをキーにデータベースからメールアドレスを取り出す？

```py
import json
import boto3

# 例外ハンドラーを呼びだす関数
def call_exception_handler(e):
    d = {'error_name':e.__class__.__name__, 'error_message':e.args}
    return boto3.client('lambda').invoke(
        FunctionName = 'ma-narushima-invoked-function',
        InvocationType = 'RequestResponse',
        Payload = json.dumps(d)
    )

# ZeroDivisionErrorを起こす関数
def raiseZeroDivisionError():
    1 / 0
    
# 自作例外クラス 
class MyError(Exception):
    pass

# 自作例外を発生させる関数
def raiseMyError(error_message):
    raise MyError(error_message)

def lambda_handler(event, context):
    response = ""
    try:
        # raiseZeroDivisionError()
        raiseMyError('User input was insufficient') # 例外発生時にユーザに通知したいメッセージを格納する
    except ZeroDivisionError as e: 
        response = call_exception_handler(e)
    except MyError as e:
        response = call_exception_handler(e)
        
    output = response['Payload'].read().decode('utf-8')
    print(output)

```

# 呼び出され側の実装

```py
def lambda_handler(event, context):
    print(event) # エラー情報を含むeventをログに出力
    error_name = event['error_name']
    error_message = event['error_message']
    return f'Handled by {error_name} and send message {error_message} to client.'
```

呼び出し側のログ

```txt
Test Event Name
test

Response
null

Function Logs
START RequestId: 135cd4d3-0951-4845-8b89-33a1185dcead Version: $LATEST
"Handled by MyError and send message ['User input was insufficient'] to client."
END RequestId: 135cd4d3-0951-4845-8b89-33a1185dcead
REPORT RequestId: 135cd4d3-0951-4845-8b89-33a1185dcead	Duration: 1019.76 ms	Billed Duration: 1020 ms	Memory Size: 128 MB	Max Memory Used: 77 MB	Init Duration: 230.97 ms

Request ID
135cd4d3-0951-4845-8b89-33a1185dcead
```

呼び出され側のログ

自作エラー発生時に呼び出された場合
```txt
2021-09-06T02:22:57.100+09:00	START RequestId: 9bb3166c-c436-4548-a881-92a86382ffce Version: $LATEST

2021-09-06T02:22:57.104+09:00	{'error_name': 'MyError', 'error_message': ['User input was insufficient']}

2021-09-06T02:22:57.104+09:00	END RequestId: 9bb3166c-c436-4548-a881-92a86382ffce

2021-09-06T02:22:57.104+09:00	REPORT RequestId: 9bb3166c-c436-4548-a881-92a86382ffce Duration: 0.89 ms Billed Duration: 1 ms Memory Size: 128 MB Max Memory Used: 52 MB
```

しっかり埋め込んだBusinessExceptionが取得できている

Lambdaの共通のコードを管理する方法としてLambdaレイヤーというのがあるらしい。これ上手く使える？
<https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/configuration-layers.html>

### 再実行

タイムアウトの場合、3回実行された。それが全部失敗したら、dlqに入った。

システム例外の場合、3回実行された。それが全部失敗したら、dlqに入った。

lambdaの自動リトライも2回（実行回数は合計3回）で、sqsのMaximum receivesも3回なので確かに3回取得・実行されて、ダメならdlqとわかる。

そこで実験的にMaximum receivesを500にしたら再実行3回ののち、dlqに格納されなくなった。
しかし、sqsにも残っていない。なぜ？
sqsに残っていたらしい。可視性タイムアウトが終わったタイミングで再度lambdaが動いているようだ。
トリガーがSQSに格納されたタイミングのはずなのだが、可視性タイムアウトが終了したタイミングでも再度トリガーをかけてくれる。
