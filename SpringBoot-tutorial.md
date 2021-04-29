# チュートリアルでハマった点などのメモ

## 3. 環境構築

lombocやmavenの事前インストールなどは不要
プロジェクトを作成し、必要なチェックをすれば、InteliJがやってくれる

1. プロジェクトの作成
    * SpringInitializerを利用する
      * Type: maven
      * Packaging; jar (webアプリと思って、warにする必要はない)

    * dependencies

    以下にチェックをつける
      * Spring Web
      * Lombok
      * H2 Database
      * MyBatis Framework

## 4. REST APIの作成

1. pomの設定

1. TodoResourceクラスの作成
    * javax.validationは、pomにて追加が必要
    (最近spring-boot-starter-webからvalidationが外されたため)

    ```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
    ```

    * @JsonFormatが uuuuになってるけど、yyyyの誤植かな？ -> yyyyで実施した

1. Repositoryの作成
    * @Selectなどを使うことで、Repository.xmlを書かない
    * @Optionって何
    @Insertと一緒に使って、主キーを動的に生成する等

    ```java
        @Options(useGeneratedKeys = true, keyProperty = "id")
        @Insert("INSERT INTO users (name) VALUES(#{name})")
        boolean insert(User user);
    ```

1. Repositoryの単体テスト
    * @MyBatisTestは、pomに追加が必要
    version指定が必要な時と、不要な時の違いって何。。？

    ```xml
        <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter-test</artifactId>
        <version>2.1.3</version>
        <scope>test</scope>
        </dependency>
    ```

    * AssertJはメソッドチェーンで書ける

        ```java
            assertThat(検証対象)
                .isEqualTo("hello");
        ```

    * DTOのテストには、extracting(Function<T、R>)が便利

        extractingで、要素を抽出して試験できる

        ```java
            List<DTO> ls = findAll(); //DBから全行取得するメソッドのテスト
            assertThat(ls)
                .extracting(DTO::getId, DTO::getName, DTO::getAge) //idと、nameと、ageだけ試験する
                .contains(
                    tuple(1, "TARO", 56),
                    tuple(2, "HANAKO", 47),
                    tuple(3, "TOSHIO", 68)
                )  
        ```

        > このメソッド参照難しい。ラムダ式で書くなら、i -> i.getId();。
        ectractingの内部で、DTOインスタンスを渡して実行するようにして使われている。apply(DTO dto);

    * List<DTO>の試験なら、tupleを使うけど、DTO単体の時はtupleを使わない
        tuple使うと、()がつく。単体の結果は()つかないので、containsで失敗になる。

1. Serviceの単体テスト
    RepositoryはMock化する。Mockitoより、BDDMockitoを使う。

### mockito, BDDMockitoのメソッド

| method | 役割 |
| --- | --- |
| times(int n) | n : 呼ばれる期待回数 |

* setup mock
    BDDMockitoでは、@MockBeanでモックを作る

    動作の定義
  * 戻り値がある時
        given(todoRepository).findById(1L).willReturn(Optional.of(expectTodo));
  * 戻り値がない時
        willDoNothing().given(todoRepository).create(expectTodo);
  * 例外をthrowする時
        willThrow(new SQLException("test")).given(this.resultSet).close();

* check
    アサーション
    should。mockitoでいう、verify。
    呼び出された回数を確認するなどに使う。

    mockito

    ```java
    verify(service, times(1)).getContentById(id);
    ```

    BDDMockito

    ```java
    then(service).should(times(1)).getContentById(id);
    ```

    比較検証
    assertThat(xxx, yyy);

    > [memo]
    > Mockオブジェクトは、全メソッドがロジックなしのメソッド（スタブメソッド）になる。
    > スタブメソッドは、defaultでNullを返すことになっている。
    > @SpyでMockオブジェクトを生成すると、メソッドがロジックをもったままになる。特定のメソッドだけthenなどで上書きすれば、狙ったメソッドだけ部分的にスタブ化できる。

1. TodoControllerクラスの作成
    業務で書いてきたものとの違いは、RestControllerであること。

    * RequestMappingをコントローラーに付与して、メソッドにはGetMappingと、PostMapping, PutMapping, DeleteMappingを使ってリクエストの種類に応じて対応する。

    * Formを作って、JSPを返すのではなく、Formと同じような入れ物であるResourceを返す。

1. Controllerの単体テスト
    ResponseEntityと、getForEntityを理解する必要がある。

## アプリケーションの実行

1. 実行可能jarファイルの作成
    mavenタブから、mvn package

1. 実行可能jarファイルの実行
    jarを右クリックして実行

1. REST API Clientによる動作確認
    REST APIの実行は、Chrome拡張の、Talend API Tester(旧名 DHC)
