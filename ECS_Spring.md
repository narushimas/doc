# AWS ECS上に構築するSpringアプリケーション

第4回から順にやっていく (初心者の館(3))

## 第4回

https://news.mynavi.jp/itsearch/article/devsoft/4354

* availability zone :
  AWSのデータセンタ。 availability zoneを越えてデータを保存することで可用性を高められる。

* VPC :
  virtual private cloud。
  VPCの中に複数のサブネットを作って構築する。
  VPCのなかに、複数のavailability zoneを使うように作れる。

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
        | Public subnet's IPv4 CIDR:* | 10.2.20.128/25 | こうすることで、128以上がpublicになる |
        | Private subnet's IPv4 CIDR:* | 10.2.20.0/25| 1~127がpublicになる |
        | Elastic IP Allocation ID:* | 上で取得したElastic IP| - |
