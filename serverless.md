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
