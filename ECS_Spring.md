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

dockerビルドで作るイメージ名はgitのリポジトリ名と揃えてみた

```shell
docker build backend/ -t narushimas/backend:latest
```

-t でイメージ名とタグ名を指定できる

