# AWS ECS上に構築するSpringアプリケーション

第4回から順にやっていく (初心者の館(3))

## 第4回

https://news.mynavi.jp/itsearch/article/devsoft/4354

`初めに`

AWSは（多分）慣れるまでネットワークが難しい。単語を覚えるのと、それぞれの役割を正しく理解することを意識して取り組みたい。

わかりやすい気がした動画($https://www.youtube.com/watch?v=6muiXYF6CjE)

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

## 第5回