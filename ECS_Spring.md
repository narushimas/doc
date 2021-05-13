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

    1. amazonコンソールに、narushimasユーザでログイン

    1. 上の検索窓から、`VPC`で検索

    1. まずElastic IPアドレスを取得する

        | 項目 | 設定値 |
        | --- | --- |
        | tag | Name: `ma-narushima-elasticip` |

       特に指定しなかったけど、以下になった。
       `35.74.10.18`

    1. 以下の情報を入力して、作成。

        | 項目 | 設定値 | 備考 |
        | --- | --- | --- |
        | IPv4 CIDR block:* | `10.2.20.0/24` | - |
        | Name | `ma-narushima-vpc` | - |
        | Public subnet's IPv4 CIDR:* | 10.2.20.128/25 | こうすることで、128以上がpublicになる； |
        | Private subnet's IPv4 CIDR:* | 10.2.20.0/25| 1~127がpublicになる |
        | Elastic IP Allocation ID:* | 上で取得したElastic IP| - |
