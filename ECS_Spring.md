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
  送信先のipアドレスをみて、どこに送信するか決めるもの。
  事前に作成しておき、サブネットに紐づける。
  今回もそうするが、複数のサブネットに1つのルートテーブルを紐づけることもできる。

  例 10.2.20.0/26というサブネットのルートテーブル例

  | 送信先 | ターゲット | 備考　|
  | --- | --- | --- |
  | 10.2.20.0/26 | LOCAL | 自身のサブネット内なら当然LOCAL |
  | 0.0.0.0/0 | Internet Gateway | 上に当てはまらなかったもの全て、インターネットに。デフォルトゲートウェイという。 |

  > ルートテーブルのターゲットにInternet Gatewayがあれば、パブリックサブネットになる。そうでなければプライベートサブネット。

* Internet Gateway

  VPC内のリソースと、インターネットをつなげるGateway
  サブネットのルートテーブルで、これを指定することでサブネットはインターネットにアクセスできる。
  パブリックサブネットなら、ルートテーブルのデフォルトゲートウェイがInternet Gatewayになっている。

* NAT Gateway

  プライベートサブネットから、インターネットにアクセスするためGateway。NAT GatewayからInternet Gatewayを通り、インターネットに接続する。
  ただし、インターネットからプライベートサブネットには入れない。
  NAT Gatewayは、publicサブネットにおかないとダメ。
  NAT Gatewayには、Elastic IPをアタッチする。これはプライベートサブネットから外部にアクセスする際のIPを固定するため。IPによるアクセス制限があるサービスなどを使うために必要になったりする。

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

      一度、サブネットのIP分割方法を間違えたので、VPCを作り直そうと思ったが、VPCは消せなかった。VPC内のNATゲートウェイを削除したら、消せるようになった。

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
（疑問）初歩的だけど、この絵について、プライベートにロードバランサ用意しているということは、パブリックはアベイラビリティゾーンA, プライベートはアベイラビリティゾーンBというような使い方もできるようにしたいため？

  Elastic Load Barancerは、Application, Network, Gatewayに分かれる。

  ターゲットグループ

    トラフィックを流す対象のこと。EC2, Lambda, プライベートIPアドレスなど。

  設定については、以下だけ注意した。

  ALB: ma-narushima-public-alb, ma-narushima-private-alb

  SG: ma-narushima-public-alb-sg, ma-narushima-private-alb-sg

  TG: ma-narushima-public-tg, ma-narushima-private-tg

  ロードバランサにセキュリティグループを設定する。ロードバランサの送信先をまとめて、ターゲットグループという。

  プライベートサブネットのロードバランサのセキュリティグループのソース
  `10.2.20.0/24` (VPCのIPアドレス)

  よくわからなかったけど、ヘルスチェック用のhtmlの名前は、パブリック、プライベート両方とも`/backend-for-frontend/index.html`とした。アプリケーションコンテキストパスの下にこれをつけて、ヘルスチェックするらしい。（あとから振り返ると、BFFアプリケーションと同じ名前にすべきだったか？）

  ALBのDNS名は、コンソールから、ALB、DNS Nameとして、調べられる
  `ma-narushima-public-alb-1213230744.ap-northeast-1.elb.amazonaws.com`

## 第6回 SpringBootアプリケーション作成

<https://news.mynavi.jp/itsearch/article/devsoft/4363>

プロジェクト作成部分に、なかなか説明が少ない。

以下プロジェクト名にしてみた

* Backendアプリケーション: backend
* BFFアプリケーション: bff

SpringInitializerを使った。mavenプロジェクト。

ただし、データベースアクセスとか、Modelクラスとかないので、SpringBootチュートリアルでしたような、Mybatis Frameworkや、lombok, spring webなどのチェックは付与しなかった。
`間違い` Modelクラスあるし、Lombok使う前提で描かれてるので、lombokだけはチェック付けておけばよかった。

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

Userクラスの記述がないため、実装方法がわからない。
Controllerでの使い方を見るに、Lombok BuilderというようなModelの実装方法らしい。

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

この記事でも将来的には、クラウドフォーメーションのスタックからDNSサーバのurlなどを取得する。

CloudFormationのスタックには、作ってみるまで分からないサービスの色々な値が入る。
DNSの論理名もALB作ってみるまで分からないけど、その情報のキーは定義しておけるから、実際に作ってみて設定された値を、キーを元に取りだす。

記事にはないけど、ServicePropertiesクラスは、@Dataつけてgetterを自動生成しておかないとダメ。WebMvcConfigurerからgetDns()を呼び出すので。

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

ECS(Elastic Container Service)クラスタは、dockerが動くEC2だと思う。それに限らないらしいが、しばらくそれで頭に入れておいてよさそう。

ECSクラスタは、public, privateそれぞれについて、アベイラビリティゾーンの数だけ必要なので、本記事の構成なら4つ作るのだけど、時間がかかるので、今回はひとまずpublic, private1つずつで対応する。

クラスターテンプレートは、「EC2 Linux + ネットワーキング」で作成する。（多分裏でできるのは、EC2で、その上にdockerのインストールなどがされてる感じ？）

クラスター名

* `ma-narushima-cluster-private`
* `ma-narushima-cluster-public`

VPCは、自分が以前作ったものを選択

サブネットは、以前作ったものが4つ（private2つ、public2つ）あるが、今回はprivateとpublicを1つずつ使う。

private -> ma-narushima-Private-subnet1
public -> ma-narushima-Public-subnet1

* キーペアは、ma-narushimaとかいうやつ。サンプル系はこれで固定しよう。

ECSも、実態はEC2なので、キーペアを発行していて、IPアドレスと合わせて、ログインできる。ログを確認するときなどにログインする。

* セキュリティグループの設定

ソースをどう設定すればいいかわからない。
privateの方は多分下でいい

* private

追加するのはSSHと、カスタムTCPルール

SSHのソースは、publicのEC2からのアクセスしか許さないから、publicのipアドレス域を指定すればいいかな？と思ったけど、記事ではVPCのipアドレス域を指定してそう？

結局VPCのCIDRで、`10.2.20.0/24`にしておいた。

アベイラビリティゾーンが2つある時、片方のprivateに、別のアベイラビリティゾーンのpublicからsshできていいのか？

カスタムTCPルールには、ポートを32768-61000で指定して、ソースはipアドレスではなく、セキュリティグループで、ALB（多分private）で指定しているセキュリティグループを指定する。`sg-0ef8eeb7dfc2056cb`

javaアプリが使うポートは32768-61000の中のどれかにはなるってことだろう。publicのbffから、privateのALBを介してprivateのECSに到達したいので、このALBはprivateで良いはず。ECSにアプリからアクセスするのは、privateのALBを介する時だけということだろう。

* public

privateとほとんど同じ。ただし、カスタムTCPルールのソースは、publicのALBにした。

## 第9回 ECSタスクの定義

ECSタスクとは

Dockerコンテナの設定（マッピングするポート番号や、割り当てるCPUやメモリ）を指定すること。docker-composeファイルに近い？

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

AmazonECSServiceRolePolicyってのは、ECSがよく使うサービスに対するポリシーが含まれている？という感じ？

ma-narushima-ecs-service-roleは、AmazonECSServiceRolePolicyがattachされていない状態で作成されていた。

これにAmazonECSServiceRolePolicyを付与しようとしたが、選択肢に出てこない。

記事には、ECSを初めて起動する場合、ロールは自動で作成されるとあるが、ECS起動したことがないと、Policy付与できないの？？

とりあえずRoleつくらず次に進んでみる

ECSを開き、以前作成したクラスタを選択する。

ma-narushima-cluster-private
ma-narushima-cluster-public

クラスター起動タイプに、EC2を選ぶ。

裏で以下のように、先頭にEC2ContainerServiceがついたEC2が作成される。
`EC2ContainerService-ma-narushima-cluster-public`

クラスターを作成し終えたら、パブリックサブネットのALBのDNSに以下のBFFアプリケーションのパスを加えるというのだが、これはどうやる？
ややこしい。ALBにDNS用の名前を与えるだけ。

パブリックのALBには、最初から以下の名前が指定されているが。。？
ma-narushima-public-alb-1213230744.ap-northeast-1.elb.amazonaws.com

これでアプリにアクセスできない

<http://ma-narushima-public-alb-1213230744.ap-northeast-1.elb.amazonaws.com/bff/index.html>

-> `503 Service Temporarily Unavailable`

bffのDockerFileに誤りがあったので、bffプロジェクトのDockerFileを修正、githubに反映。その後、dockerビルド用のEC2にログインして、git pull --rebase, docker build -t narushimas/bff:latest, docker login, docker push narushimas/bff:latestでDockerHubを最新化した。

さらに、AWSコンソールのECSから、publicのClusterを選択し、updateで最新化した。

これでも直らないので、CloudWatchのメトリクスをみて、問題を探してみる。

更新したbffのクラスターで、サービスがrunnigになっていなかった。更新していないbackendの方はrunnig。これが原因か?

ma-narushima-bff-serviceのEventsタグに、以下の記述があった。

EventId: 5b556272-9cf2-497e-9b66-cc5313cc85a6

Date:
2021-06-02 14:29:43 +0900

Massage: service ma-narushima-bff-service was unable to place a task because no container instance met all of its requirements. Reason: No Container Instances were found in your cluster. For more information, see the Troubleshooting section.

これとは関係ないけど、ECSログのパス
<https://docs.aws.amazon.com/AmazonECS/latest/developerguide/logs.html>

ECSにsshすると見えるらしいけど、ECSのipアドレスはどこから確認できる？

と思って探したら、問題のpublicの方は、EC2が起動してなさそう。。(privateの方は起動している)

VPCに設定するファイアフォールである、Network ACLに問題がある？

よくわからないけど、InBoundルールに適当に追加してみた。もともとフルオープンで、意味なさそうだけど。
<https://ap-northeast-1.console.aws.amazon.com/vpc/home?region=ap-northeast-1#NetworkAclDetails:networkAclId=acl-064ad8c1918f78e1c>

変わらない。

ECSのpublicクラスター用のEC2(public ipなし)はできてたから、適当にpublicサブネットにpublicIP付きのEC2たてて、そこをアクセスしてみることにする。

問題のあるpublic側のECSのEC2にログインしてlogを確認した

```shell
cat /var/log/ecs/ecs-agent.log
```

```log
level=error time=2021-06-08T07:01:26Z msg="Unable to register as a container instance with ECS: RequestError: send request failed\ncaused by: Post \"https://ecs.ap-northeast-1.amazonaws.com/\": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)" module=client.go
level=error time=2021-06-08T07:01:26Z msg="Error registering: RequestError: send request failed\ncaused by: Post \"https://ecs.ap-northeast-1.amazonaws.com/\": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)" module=agent.go
```

同じエラーが出た人が解析していた

<https://qiita.com/aikan-umetsubo/items/e4cf1c6a092d82503c63>

自分もこの人と同じように、ping google.comしてみたけど、EC2から外にネットワークが繋がってないらしい。

privateからはインターネットにアクセスできる。繋がってる。やっぱりインターネットにつながっていないことが理由っぽい。
どうしたら、publicの方をインターネットにつながるようにできるか。

インターネットに繋がらないとき、ファイアウォールは、セキュリティグループ（EC２単位）、ネットワークACL（サブネット単位）でそれぞれ考える。

ルートテーブルでは、問題のEC2があるpublicサブネット1がIGWに紐付けできていた。さらにいうと、privateサブネット1は、このpublicサブネット1のNAT GWを利用してインターネットアクセスしているので、サブネットはインターネットアクセスできる状態である。となると、EC2のセキュリティグループの問題？

iptablesで調べてみた

問題のEC2(インターネットアクセスできない)

Chain INPUTをみると、tcp全てDROPしているような？

```shell
[ec2-user@ip-10-2-20-9 ~]$ sudo iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
DROP       tcp  --  anywhere             anywhere             tcp dpt:51678
DROP       all  -- !ip-127-0-0-0.ap-northeast-1.compute.internal/8  ip-127-0-0-0.ap-northeast-1.compute.internal/8  ! ctstate RELATED,ESTABLISHED,DNAT

Chain FORWARD (policy DROP)
target     prot opt source               destination         
DOCKER-USER  all  --  anywhere             anywhere            
DOCKER-ISOLATION-STAGE-1  all  --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
DOCKER     all  --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere            

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

Chain DOCKER (1 references)
target     prot opt source               destination         

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt source               destination         
DOCKER-ISOLATION-STAGE-2  all  --  anywhere             anywhere            
RETURN     all  --  anywhere             anywhere            

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
target     prot opt source               destination         
DROP       all  --  anywhere             anywhere            
RETURN     all  --  anywhere             anywhere            

Chain DOCKER-USER (1 references)
target     prot opt source               destination         
RETURN     all  --  anywhere             anywhere   
```

普通のEC2(インターネットアクセス可能)

```shell
[ec2-user@ip-10-2-20-44 ~]$ sudo iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination       
```

iptablesのファイアフォールって、sgと一緒か？

<https://ap-northeast-1.console.aws.amazon.com/ec2/v2/home?region=ap-northeast-1#SecurityGroup:group-id=sg-0c77b78354a8b35a1>

インスタンスにパブリックIPアドレスがないとインターネットアクセスできないらしい。それが原因？プライベートサブネットのやつは、NATゲートウェイがパブリックIP持ってるからそれでできたのか？

<https://aws.amazon.com/jp/premiumsupport/knowledge-center/ec2-connect-internet-gateway/>

これをベースに探して、原因っぽいものを見つけた。

publicサブネットの設定

`Auto-assign public IPv4 address`
`No`

サブネット内で起動したインスタンスに自動でIP4アドレスを割り当てるもの。これをYesにしないとダメか。

有効化して、EC2を再起動したけどIP割り当てられない。
EC2を作り直してみる。

割り当てられた。

しかしindex.htmlにアクセスしても、404。
springbootの返す404っぽいので、APは起動していそう。

ここで、index.html作ってないことと、context-pathの設定をしていないことを思い出し、川畑さんのリポジトリ見て、それぞれ追加した。

context-pathは以下のように指定

application.yml

```yml
service:
  dns: http://internal-ma-narushima-private-alb-411929720.ap-northeast-1.elb.amazonaws.com

server:
  servlet:
    context-path: /bff
```

spring-bootのsrc直したら、docker image作り直すが、
その際に前回のcacheを利用されてしまい、更新がされなかった。

`--no-cache`オプションをつけて回避した。

```shell
docker build --no-cache bff/
```

しかし、遅い。なんか差分だけビルドしてくれるようなのないか。

そしてdocker-hubのコンテナイメージを更新した場合、ECSの実行するコンテナイメージを最新化するには何をすればいい

とりあえずタスクを消したら(stopしたら)、なんか自動でタスクが再作成された。
その後、コンテナイメージも最新化されてたらしく、アクセス成功した。

clusterのupdateは違う？

<http://ma-narushima-public-alb-1213230744.ap-northeast-1.elb.amazonaws.com/bff/index.html>

Call backend serviceボタンを押しても、次の画面にならないから、
川畑さんのリポジトリ参照してるけど、どうやって記事のような表が帰ってくる？

user.htmlがいた

<https://github.com/debugroom/mynavi-sample-aws-ecs/blob/master/backend-for-frontend/src/main/resources/templates/users.html>

bffのindex.html -> bffのController -> RestAPIでuser情報取得 -> 情報をusers.htmlに埋め込んでhtmlを返す流れっぽい

Controllerの実装部分

```java
@Controller
public class BackendForFrontendController {

    @Autowired
    RestOperations restOperations;

    @RequestMapping(method = RequestMethod.GET, value = "users")
    public String getUsers(Model model){
        String service = "/backend/api/v1/users";
        model.addAttribute("users",
                restOperations.getForObject(service, User[].class));
        return "users";
    }

}
```

restOperations.getForObjectで、REST APIのレスポンスを、Userクラスにバインドして、それをmodelに追加してるぽい。

戻り値の"users"は、jspじゃなくて、users.htmlなんだけど、それにもmodelから自動バインドできるということか

```html

<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org" lang="ja">
<head>
    <title>Hello! Thymeleaf</title>
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="stylesheet" href="static/css/flex.css" media="(min-width: 1280px)">
    <link rel="stylesheet" href="static/css/flex_mobile.css" media="(min-width: 320px) and (max-width: 767px)">
    <link rel="stylesheet" href="static/css/flex_tablet.css" media="(min-width: 768px) and (max-width: 1279px)">
    <link rel="stylesheet" href="static/css/users.css" media="(min-width: 1280px)">
    <link rel="stylesheet" href="static/css/users_mobile.css" media="(min-width: 320px) and (max-width: 767px)">
    <link rel="stylesheet" href="static/css/users_tablet.css" media="(min-width: 768px) and (max-width: 1279px)">
    <script type="text/javascript" src="webjars/jquery/jquery.js"></script>
</head>
<body>
<h1>Hello! AWS ECS sample!</h1>
<table>
    <thead>
    <tr>
        <th>No</th>
        <th>User ID</th>
        <th>User Name</th>
    </tr>
    </thead>
    <tbody>
    <tr th:each="user, status : ${users}">
        <td th:text="${status.count}"></td>
        <td th:text="${user.userId}"></td>
        <td th:text="${user.userName}"></td>
    </tr>
    </tbody>
</table>
</body>
</html>
```

jspではなく、Thymeleafとかいうのを使ってるらしい

`tr th:each="user, status : ${users}"`
のところ、
${users}で取得したクラスのリスト（配列？）のuserクラスと、ループのステータスを使ってループする。スタータスは要素番号など色々とれる

Thymeleaf th:eachについて

<https://www.ne.jp/asahi/hishidama/home/tech/java/spring/boot/thymeleaf/th_each.html>

あと、川畑さんのbackencのapplication.ymlにdnsプロパティあったけど、必要？
いつつかう。

* privateサブネットのbackendAPの挙動を確かめるために、Talend API TesterでALBのDNS名にアクセスしたけど、Abortedだった。どうやらALBに割り当てられるDNS名は、名前解決するとプライベートipアドレスになるらしい。
<https://dev.classmethod.jp/articles/internal-alb-https/>

privateサブネットにアクセスできるpublicサブネットから、nslookupでALBのDNS名を解決すると

```shell
[ec2-user@ip-10-2-20-44 ~]$ nslookup internal-ma-narushima-private-alb-411929720.ap-northeast-1.elb.amazonaws.com
Server:         10.2.20.2
Address:        10.2.20.2#53

Non-authoritative answer:
Name:   internal-ma-narushima-private-alb-411929720.ap-northeast-1.elb.amazonaws.com
Address: 10.2.20.120
Name:   internal-ma-narushima-private-alb-411929720.ap-northeast-1.elb.amazonaws.com
Address: 10.2.20.221
```

同じように、publicサブネットのEC2から、curlしたら正しく返ってきたので、backendAPは正しく動いている

ALBのDNS名（名前解決されてprivateIPアドレスになる） + backendAPで用意しているURL

```shell
curl internal-ma-narushima-private-alb-411929720.ap-northeast-1.elb.amazonaws.com/backend/api/v1/users
```

```shell
[{"userId":"1","userName":"Taro"},{"userId":"2","userName":"Jiro"}] 
```

javaプロジェクトのソースを変更して、再度docker buildしようとしたら

```shell
[ec2-user@ip-172-31-37-43 ~]$ docker build --no-cache bff/ -t narushimas/bff:latest
Sending build context to Docker daemon  326.1kB
Error response from daemon: Error processing tar file(exit status 1): write /src/main/java/config/WebApp.java: no space left on device
```

というエラーがでた。ホストマシンのディスク容量不足らしい。
EC2のディスク容量を増やしてみることにした。

AWSのコンソールから見ると、8GiBになっていた。

```
Device name : /dev/xvda
Volume size(GiB) : 8
```

EC2のインスタンスのStrageタグから、
ActionsからModify Volumeを選び、16GiBに変更してみた。

100％使用されていた/dev/xvda1の容量が16GiBになって空いた。

df  （該当箇所抜粋）

``shell
/dev/xvda1      16764908 8386856   8378052  51% /
```

docker image rmで出来上がったimage削除してもそれなりに空いた。そっちでもよかったかも


call backend しても404なので、users.htmlがないからかと思ったが、追加しても404。
MvcConfigに以下設定(addResourceHandlersの追加)が必要?


```java
@ComponentScan("app.web")
public class MvcConfig implements WebMvcConfigurer {
    @Autowired
    ServiceProperties properties;

    @Bean
    public RestOperations restOperations(RestTemplateBuilder restTemplateBuilder){
        return restTemplateBuilder.rootUri(properties.getDns()).build();
    }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/");
    }
}
```

追加してもまだ404だけど。@Configurationつけてなかった。
@ComponentScan("app.web")の下に、
@Configurationつけたらコントローラーが動くようにはなった。

### ECS上のAP変更手順

docker hubのイメージを変更し、タスクを再実行すれば変更される。

* bffというJavaプロジェクトを最新化する場合

1. githubを最新化
2. docker imageを最新化(DockerFileの中でgithubのソースをcloneする処理が書かれている前提)

  ```shell
  docker build --no-cache bff/ -t narushimas/bff:latest
  ```

  ```shell
  docker login
  ```

  ```shell
  docker push narushimas/bff:latest
  ```

3. AWSコンソールでECSタスクをstopする（ECSサービスが作成済で、それによりbffがタスクとして動く状態が前提）
   勝手にタスクが再実行されて、最新化される
   > タスクのupdateでは更新されないので注意
