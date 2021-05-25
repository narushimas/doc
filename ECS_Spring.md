# AWS ECS上に構築するSpringアプリケーション

第4回から順にやっていく (初心者の館(3))

## 第4回 AWS上のネットワーク設定

<https://news.mynavi.jp/itsearch/article/devsoft/4354>

`初めに`

AWSは（多分）慣れるまでネットワークが難しい。単語を覚えるのと、それぞれの役割を正しく理解することを意識して取り組みたい。

### 単語整理

* availability zone :
  AWSのデータセンタ。 availability zoneを越えてデータを保存することで可用性を高められる。

* VPC :
  virtual private cloud。
  VPCの中に複数のサブネットを作って構築する。
  VPCのなかに、複数のavailability zoneを使うように作れる。

* サブネット
  VPCをさらに複数のネットワークに分けた時の１つ

* ルートテーブル
  各サブネットに設定する。
  ipアドレスを見て、どこに送信するかを判定する時に使うテーブル。

  例 10.2.20.0/26というサブネットのルートテーブル例

  | 送信先 | ターゲット | 備考　|
  | --- | --- | --- |
  | 10.2.20.0/26 | LOCAL | 自身のサブネット内なら当然LOCAL |
  | 0.0.0.0/0 | Internet Gateway | 上に当てはまらなかったもの全て、インターネットに。デフォルトゲートウェイという。 |

  > ルートテーブルのターベットにInternet Gatewayがあれば、パブリックサブネットとよぶ。そうでなければプライベートサブネット。特にパブリックサブネットはデフォルトゲートウェイがInternet Gateway

* Internet Gateway

  VPC内のリソースと、インターネットをつなげるGateway
  サブネットのルートテーブルで、これを指定することでサブネットはインターネットにアクセスできる。
  パブリックサブネットなら、ルートテーブルのデフォルトゲートウェイがInternet Gatewayになっている。

* NAT Gateway

  プライベートサブネットから、インターネットにアクセスするためGateway。NAT GatewayからInternet Gatewayを通り、インターネットに接続する。
  ただし、インターネットからプライベートサブネットには入れない。
  NAT Gatewayは、publicサブネットにおかないとダメ。

> Internet Gateway, NAT Gatewayともに、デフォルトで可用性が担保されていて、必要に応じて自動で複製される。

* ENI

  elastic network interface (いわゆるNIC)

  IPアドレスをアタッチするカードであり、NATゲートウェイ（ハブ）やEC2(サーバ)に指していると思うといい？
  
### 作業

* 自分がもらったサブネット
10.2.20.0/24

> 10台は、全てプライベートIPアドレス

1. VPC作成

  ここでは、VPCを作成し、サブネットを４つに分割する。

  Publicサブネット1, Privateサブネット1
  Publicサブネット2, Privateサブネット2

  IPアドレスは、VPCのCIDRが`10.2.20.0/24`なので、
  これを4等分する。
  
  第４オクテットしか使えないので、クラスフルアドレス（オクテット単位(8, 16, 24)でネットワーク部を指定することはできない。）

  しかしクラスレスアドレスで、このように分けられる。(第４オクテットの上位2bitで、４ネットワークを分割する)

  | 名前 | CIDRブロック |
  | --- | --- |
  | Publicサブネット1 | `10.2.20.0/26` |
  | Privateサブネット1 | `10.2.20.64/26` |
  | Publicサブネット2 | `10.2.20.128/26` |
  | Privateサブネット2 | `10.2.20.192/26`|

  1. amazonコンソールに、narushimasユーザでログイン

  2. 上の検索窓から、`VPC`で検索

  3. まずElastic IPアドレスを取得する

      | 項目 | 設定値 |
      | --- | --- |
      | tag | Name: `ma-narushima-elasticip` |

      特に指定しなかったけど、以下になった。
      `35.74.10.18`

  4. 以下の情報を入力して、作成。

      | 項目 | 設定値 | 備考 |
      | --- | --- | --- |
      | IPv4 CIDR block:* | `10.2.20.0/24` | - |
      | Name | `ma-narushima-vpc` | - |
      | Public subnet's IPv4 CIDR:* | 10.2.20.0/26 | こうすることで、1 ~ 63がPublicサブネットで利用できるIPになる |
      | Private subnet's IPv4 CIDR:* | 10.2.20.64/26| 64~127がPrivateサブネットで利用できるIPになる |
      | Elastic IP Allocation ID:* | 上で取得したElastic IP | - |

      サブネットのtag名は、作成時に指定できない？っぽいので、VPC作成後にサブネットから検索して、`ma-narushima-Public-subnet1`,  `ma-narushima-Private-subnet1`とした。

      一度、サブネットのIPアドレスの分割を間違えた.
      アベイラビリティゾーン用のIPアドレスが残っていないので、VPCを作り直そうと思った。しかしVPCは消せなかった。VPC内のNATゲートウェイを削除したら、消せるようになった。

  5. 追加でサブネットを作成

      VPCと同時に作ったサブネットのavailablity zone指定し忘れたが、Public, Privateともに`ap-northeast-1a`になっていたので、これとして進める。つまり、新しく作るサブネットはこことは別のavailability zoneにつくる。`ap-northeast-1c`にした。`ma-narushima-Public-subnet2`, `ma-narushima-Private-subnet2`

  6. ルートテーブルのタグ変更
  
      よくわからないけど、ルートテーブルが2つありタグがついていないので、`ma-narushima-routetable1`, `ma-narushima-routetable2`としておいた。これもよくわからないけど、Mainという項目がYesのものと、NoのものがあったのでYesとなっているものを1にしておいた。
      -> Yesがメインルートテーブル、Noがカスタムルートテーブルというらしい。

  7. 追加で作ったパブリックサブネットを、VPCのルートテーブルに登録した。

      元々、最初に作ったパブリックサブネットだけ、カスタムルートテーブルに登録されていて、残り3つはメインルートテーブルに登録されていた。メインルートテーブルのパブリックサブネットをカスタムルートテーブルに登録した。

## 第5回 ロードバランサの設定

<https://news.mynavi.jp/itsearch/article/devsoft/4359>

　ロードバランサはALB(Application Load Barancer)。ALBはパブリックサブネット用と、プライベートサブネット用に2つ用意する。
（疑問）初歩的だけど、この絵について、プライベートにロードバランサ用意しているということは、パブリックはアベイラビリティゾーンA, プライベートはアベイラビリティゾーンBというような使い方をしたいため？

  Elastic Load Barancerは、Application, Network, Gatewayに分かれる。

  ターゲットグループ

    トラフィックを流す対象のこと。EC2, Lambda, プライベートIPアドレスなど。

  設定については、以下だけ注意した。

  ALB: ma-narushima-public-alb, ma-narushima-private-alb

  SG: ma-narushima-public-alb-sg, ma-narushima-private-alb-sg

  TG: ma-narushima-public-tg, ma-narushima-private-tg

  プライベートサブネットのロードバランサのセキュリティグループのソース
  `10.2.20.0/24` (VPCのIPアドレス)

  よくわからなかったけど、ヘルスチェック用のhtmlの名前は、パブリック、プライベート両方とも`/backend-for-frontend/index.html`とした。アプリケーションコンテキストパスの下にこれをつけて、ヘルスチェックするらしい。

## 第6回 SpringBootアプリケーション作成

<https://news.mynavi.jp/itsearch/article/devsoft/4363>

プロジェクト作成部分に、なかなか説明が少ない。

以下プロジェクト名にしてみた

* Backendアプリケーション: backend
* BFFアプリケーション: bff

SpringInitializerを使った。mavenプロジェクト。

ただし、データベースアクセスとか、Modelクラスとかないので、SpringBootチュートリアルでしたような、Mybatis Frameworkや、lombok, spring webなどのチェックは付与しなかった。
`間違い` Modelクラスあるし、Lombok使う前提で描かれてるので、lombokだけはチェックしておけばよかった。

### GitHubへの登録

IntelliJの上部メニューのVCSから、「Share Project on Github」を押して、
認証したら、サクッとリポジトリ登録できた。楽で驚いた。

backend

<https://github.com/narushimas/backend>

bff

<https://github.com/narushimas/bff>

### pom.xmlへの追記

2つのプロジェクト両方に、記事にある記述を貼り付けていく。
dependencyだけ追加すれば大丈夫だった。pluginはmavenのものだったが、IntelliJでmavenプロジェクトで作っていたからか、元々設定できていた。

IntelliJからGit操作すると、addなしでcommitできる？
diffもみやすい。

Userクラスの記述がない？ようだけど、実装方法がわからない。
Controllerでの使い方を見るに、Lombok BuilderというようなModelの実装方法のようだ。

backendプロジェクトには、あとからlombok追加

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

GitHubに川端さんのサンプルがあったので、これと同じようにUserを作った。

<https://github.com/debugroom/mynavi-sample-aws-ecs/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/ecs/backend/app/model/User.java>

* ComponentScan対象ディレクトリの設定

今回は、app.web下のContorollerをscanしたいので、これで十分。

```java
@Configuration
@ComponentScan("app.web")
public class MvcConfig implements WebMvcConfigurer {
}
```

* コンテキストパスの設定

resourcesの下にapplication.propertiesがあったが、これをymlに変えて、記事の通り実装した。

* 他のRestApiの呼び出し

RestOperationsを使う

```java
@RequestMapping(method = RequestMethod.GET, value = "users" )
public String getUsers(Model model){
    String service = "/backend/api/v1/users";
    model.addAttribute("users", restOperations.getForObject(service, User[].class));
    return "users";
}
```

* DNSサーバの設定

このDNSは、フロントからバックエンドをURLベースでたどり着くための設定で、フロント側のAPに設定する
DNSのマッピング情報は、バックエンド用のALBに設定しておく。

bffのapplication.ymlに以下を記述する。

ALBのDNSドメインをプロパティファイルから取得して設定することで、Javaソースコード上でAWS環境に依存しないアプリケーション実装にすることができる。
（ServicePropertiesで、下の値をもったString dnsを定義できる）

具体的なurlは、AWSコンソールのALBのprivateからとってきた。

```yml
service:
  dns: http://internal-ma-narushima-private-alb-411929720.ap-northeast-1.elb.amazonaws.com
```

将来的に、クラウドフォーメーションのスタックからDNSサーバのurlなどを取得できる。

CloudFormationのスタックには、作ってみるまで分からないサービスの色々な値が入る。
DNSの論理名もALB作ってみるまで分からないけど、その情報のキーは定義しておけるから、実際に作ってみて設定された値を、キーを元に取りだす。


記事にはないけど、ServicePropetiesクラスは、@Dataつけてgetterを自動生成しておかないとダメ。WebMvcConfigurerからgetDns()を呼び出すので。

## 第7回 Dockerイメージ作成

* ECSとは

Elastic Container Service

* Docker化するために

プロジェクトの中に、Dockerfileを作成する

backendプロジェクト、bffプロジェクトの両方に追加する

Dockerfileの内容は、

1. JDKやmavenのインストール
1. gitからプロジェクトのclone
1. mavenビルド
1. jar実行

という流れ

dockerイメージ作るためにLinux使う。
LinuxはEC2で用意する。

楽に作業できるように、vscodeのRemote developmentを使う

~/.ssh/　を作り、configファイルとpemファイル(EC2作成時に手に入れる)を置く。
pemの権限は400にしておく必要があるかも。

Amazon-linuxで作ったので、初期ユーザはec2-userだった。

dockerコマンドをsudoなしで使えるようにするには、実行ユーザをdockerグループに追加する。
-> グループに追加してもsudoしないと実行できないが。。？

dockerビルドで作るイメージ名はgitのリポジトリ名と揃えてみた

```shell
docker build backend/ -t narushimas/backend:latest
```

-t でイメージ名とタグ名を指定できる

dockerユーザは元々、narushimasで作ってあったのでそれを使う。

* Dockerビルドの失敗

docker buildは、内部のmvn installで失敗してしまった。

2つ問題があった

1. `invalid target release: 11`

    というエラーで、dockerがjava1.8でmvnを動かしているのに、pomのjava-versionが11だったのが問題だったらしい。1.8にしたら直った。
    ただし、一度でもpomのjava-versionが11で実行されていると、11で作られてしまっているものが残っている？のか1.8にしても同じエラーが出続けた。docker imagesを見ると、imageがいくつかできていたのでdocker rmiで全部消して再度docker buildしたら成功した。
    Docker buildは途中で失敗しても情報を保持しているらしい？

2. `There are test failures.`

    デフォルトで生成された中身のないテストクラスにメソッドが1つあって、それがfail扱いされていた。削除したら直った。

* docker push

docker pushに成功してできた

<https://hub.docker.com/repository/docker/narushimas/backend>
アカウント narushimas

## 第8回 ECSクラスタの作成

ECS(Elastic Container Service)クラスタは、dockerが動くEC2だと思おう。それに限らないらしいが、しばらくそれで頭に入れておいてよさそう。

ECSクラスタは、public, privateそれぞれについて、アベイラビリティゾーンの数だけ必要なので、4つ作るのだけど、時間がかかるので、今回はひとまずpublic, private1つずつで対応する。

クラスターテンプレートは、「EC2 Linux + ネットワーキング」で作成する。（多分裏でできるのは、EC2で、その上にdockerのインストールなどがされてる感じ？）

クラスター名

ma-narushima-cluster-private
ma-narushima-cluster-public

VPCは、自分が以前作ったものを選択

サブネットは、以前作ったものが4つ（private2つ、public2つ）あるが、今回はprivateとpublicを1つずつ使う。

private -> ma-narushima-Private-subnet1
public -> ma-narushima-Public-subnet1

* キーペアは、ma-narushimaとかいうやつ。サンプル系はこれで固定しよう。

* セキュリティグループの設定

ソースをどう設定すればいいかわからない。
privateの方は多分下でいい

* private

追加するのはSSHと、カスタムTCPルール

SSHのソースは、publicのEC2からのアクセスしか許さないから、publicのipアドレス域を指定すればいいかな？と思ったけど、記事ではVPCのipアドレス域を指定してそう？

結局VPCのCIDRで、`10.2.20.0/24`にしておいた。

アベイラビリティゾーンが2つある時、片方のprivateに、別のアベイラビリティゾーンのpublicからsshできていいのか？

カスタムTCPルールには、ポートを32768-61000で指定して、ソースはipアドレスではなく、セキュリティグループで、ALB（多分private）で指定しているセキュリティグループを指定する。`sg-0ef8eeb7dfc2056cb`

javaアプリが使うポートは32768-61000の中のどれかにはなるってことだろう。publicのbffから、privateのALBを介してprivateのECSに到達したいので、このALBはprivateで良いはずだ。そしてECSにアプリからアクセスするのは、privateのALBを介する時だけということだろう。

* public

privateとほとんど同じ。ただし、カスタムTCPルールのソースは、publicのALBにした。

## 第9回 ECSタスクの定義

ECSタスクとは

Dockerコンテナの設定（マッピングするポート番号や、割り当てるCPUやメモリ）を指定すること。docker-composeファイルに近いな？

* ECSコンテナに割り当てるタスク実行用のIAMロール作成
  
IAMメニューから、ロールの作成
（ユースケースって何？）

ロール名: ma-narushima-ecs-task-role

ロールを作って、そこに付与したいポリシー（例えばS3アクセスポリシー）をつけていく。

今回はとりあえず、AmazonECSTaskExecutionRolePolicyでOK。

* ECSタスク定義

タスク名:

ma-narushima-backend-task

ma-narushima-bff-task

タスクロール:None(タスク実行ロールとややこしいが、こちらはタスクとして動くコンテナユーザの権限かな？)

タスク実行ロール：上で作った、ma-narushima-ecs-task-role

タスクメモリ: springアプリなら1024MB以上ということなので、画像とも合わせて1024MBにした

タスクCPU: よくわからないけど、画像見て1vcpuにした

コンテナ名:

ma-narushima-ecs-backend

ma-narushima-ecs-bff

メモリ制限: 1024にした

ポートマッピングは動的にするために、0:8080とした。
0とすると、うまくやってくれるらしい。

## 第10回 ECSサービスの実行

* AmazonECSServiceRolePolicyをアタッチしたIAMロールの作成

第9回の記事でも似たことやってので、それを参考に実施する。

(前回作ったTaskExecutionと、今回のServiceRoleのポリシーってどんな違いがある？ServiceRoleの方が権限が強そうな感じもするけど)

ロール名は、
ma-narushima-ecs-service-roleという名前にした。

しかし、ロール作成時に、
`Cannot attach a Service Role Policy to a Customer Role.`
というエラーが発生した。

Customer Roleって何？

TODO 公式ページを読む

<https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_roles_terms-and-concepts.html>

Service Roleというのは、サービスに割り当てるRoleらしい。
Customer Roleは、ユーザに割り当てるRole。
