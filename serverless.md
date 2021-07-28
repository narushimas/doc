# サーバレス

参考記事
<https://news.mynavi.jp/itsearch/article/devsoft/4316>

## lambdaに載せるjarの作り方

### jarに依存jar全てを含める

    maven-shade-pluginを利用する

## handleRequestメソッド（AWS Lambdaのエントリーポイント）

## Apprication.getBeanを使う（@Injectじゃない）

## AWS lambdaで動くアプリを作る

参考<https://www.baeldung.com/java-aws-lambda>

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
    shadeによって作られるのはoriginalの方なので、lambdaにはoriginalのついた方をuploadする
