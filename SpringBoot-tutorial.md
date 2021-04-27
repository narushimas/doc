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
    * dozermapperが使えない(mavenの不具合？セントラルにはあるっぽい)

    ```xml
        <dependency>
            <groupId>com.github.dozermapper</groupId>
            <artifactId>dozer-spring-boot-starter</artifactId>
        </dependency>
    ```

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

    * (誤植) DATE_TIME_FORMATTじゃなくて、DATE_TIME_FORMATTER

    * AssertJはメソッドチェーンで書ける

        ```java
            assertThat(検証対象)
                .isEqualTo("hello");
        ```

    * DTOのテストには、extracting()が便利

        extractingで、要素を抽出して試験できる

        ```java
            List<DTO> ls = findAll(); //DBから全行取得するメソッドのテスト
            assertThat(ls)
                .extracting(DTO::id, DTO::name, DTO::age) //idと、nameと、ageだけ試験する
                .contains(
                    tuple(1, "TARO", 56),
                    tuple(2, "HANAKO", 47),
                    tuple(3, "TOSHIO", 68)
                )
            
        ```
