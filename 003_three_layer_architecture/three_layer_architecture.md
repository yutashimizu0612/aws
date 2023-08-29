## webサーバーとしてALBを作成する

### ALB用のサブネットの作成

* ALB用のサブネットを作成
  * サブネット名：alb-subnet
  * アベイラビリティーゾーン：ap-northeast-1c
  * 🌟IPv4 CIDRブロック：10.0.0.64/26

### セキュリティグループの作成

* セキュリティグループ名：alb-sg001
* VPC：subnetを作成したVPCと同じものを選択する
* インバウンドルール
  * タイプ：HTTP
  * ソース：0.0.0.0/0

### ALBの作成

HTTP/80番ポートを開放し、全ユーザーがアクセス可能な状態とします。

* EC2 > ロードバランサー > ロードバランサーの作成
* ロードバランサータイプ：Application Load Balancer
* 基本設定
  * ロードバランサー名：sample-alb
  * 🌟スキーム：インターネット向け
  * 🌟IPアドレスタイプ：IPv4
* ネットワークマッピング
  * VPC：subnetを作成したVPCと同じものを選択する
  * subnet：alb-subnet
* セキュリティグループ
  * 上で作成したalb-sg001を選択する
* リスナー
  * プロトコル：HTTP
  * ポート：80
  * デフォルトアクション：以下で作成するターゲットグループを選択する 

#### ターゲットグループの作成

* 基本設定
  * ターゲットタイプ：EC2インスタンス
  * ターゲットグループ名：app-target-group
  * プロトコル：HTTP / ポート：3030
    * 後で3030ポートを開けたnginxを起動する予定のため、3030を指定する
  * VPC：対象となるEC2インスタンスが属するネットワークを選択する
* ターゲットの登録
  * 対象とするEC2インスタンスを選択
    * このインスタンス内にnginxを立ち上げる

#### 🌟注意：ALB作成時に選択するサブネット

ALB作成時には最低2つのサブネットを選択する必要があります。
（ロードバランサーである以上、同じサブネットに所属していたのでは役割が果たせないから？）

また、この時、2つのサブネットは、異なるアベイラビリティゾーンに属する必要があります。

異なるアベイラビリティゾーンに属するサブネットが存在しない場合、サブネットが2つ選択できないため、ALBも作成できません。

#### 注意：インターネットゲートウェイへのルート

ALB用のサブネットがインターネットからトラフィックを受信できるようにする。

* alb-subnetのルートテーブルを選択
* ルートを編集 > ルートを追加
* 送信先：0.0.0.0/0 / ターゲット：対象VPCのインターネットゲートウェイを選択
* 変更を保存

## アプリケーションサーバーとしてEC2上にnginxコンテナを立ち上げる

### アプリケーションサーバ用のサブネットの作成

* アプリケーションサーバ用のサブネットを作成
  * サブネット名：app-subnet
  * アベイラビリティーゾーン：ap-northeast-1a
  * IPv4 CIDRブロック：10.0.0.0/26

### nginxコンテナを起動する

前提として、既に"app-ec2"というEC2インスタンスが存在します。

"app-ec2"内でnginxコンテナを起動します。

まず、"app-ec2"にSSH接続します。

`nginx`ディレクトリを作成し、そこに以下の`docker-compose.yml`を作成します。

```yml
version: '3'

services:
  nginx:
    image: nginx:latest
    ports:
      - "3030:80"
```

```bash
[ec2-user@ip-****** nginx]$ sudo docker-compose up -d
[+] Running 2/2
 ⠿ Network nginx_default    Created                                        0.1s
 ⠿ Container nginx-nginx-1  Started                                        0.9s
[ec2-user@ip-****** nginx]$ sudo docker-compose ps -a
NAME                COMMAND                  SERVICE             STATUS              PORTS
nginx-nginx-1       "/docker-entrypoint.…"   nginx               running             0.0.0.0:3030->80/tcp, :::3030->80/tcp
```

### セキュリティグループの作成

* セキュリティグループ名：sg001
* VPC：subnetを作成したVPCと同じものを選択する
* インバウンドルール
  * タイプ：カスタムTCP
  * ポート範囲：3030
  * ソース：ALBのセキュリティグループを設定（alb-sg001）

### nginx起動を確認

ブラウザから`http://[EC2_PUBLIC_IP_OR_DOMAIN]:3030`にアクセスして、nginxのデフォルトページが表示できるのを確認します。

先ほど作成した`sg001`に下記のルールを追加します。

* タイプ：カスタムTCP
* ポート範囲：3030
* ソース：0.0.0.0/0

nginxコンテナが起動しているのを確認の上、ブラウザからアクセスします。

下記のようなページが表示されれば成功です。

追加したルールは削除しておきます。

## RDSを作成・起動する

### DB用のサブネットの作成

* DB用のサブネットを作成
  * サブネット名：db-subnet
  * アベイラビリティーゾーン：ap-northeast-1a
  * IPv4 CIDRブロック：10.0.0.128/26
* 2つ目のDB用のサブネットを作成
  * サブネット名：db-subnet2
  * アベイラビリティーゾーン：ap-northeast-1c
  * IPv4 CIDRブロック：10.0.0.192/26

RDSを作成するにあたっては、異なるアベイラビリティゾーンに属する複数のサブネットを持つサブネットグループを選択する必要があります。

そのため、作成する2つのサブネットでは異なるアベイラビリティゾーンを選択します。

### セキュリティグループの作成

事前にDB用のセキュリティグループを作成しておきます。

このセキュリティグループをこの後作成するDBに関連付けることによって、"app-ec2"のみがDBにアクセスできるようになります。

* セキュリティグループ名：db-sg001
* VPC：subnetを作成したVPCと同じものを選択する
* インバウンドルール
  * タイプ：MySQL/Aurora
  * ソース：sg001
    * "app-ec2"に関連付けしたセキュリティグループを指定します
    * "app-ec2"のIPアドレスも指定できますが、セキュリティグループを指定するのが一般的です

### サブネットグループの作成

* サブネットグループの詳細
  * 名前：db-subnet-group
  * VPC：subnetを作成したVPCと同じものを選択する
* サブネットを追加
  * アベイラビリティゾーン：ap-northeast-1a,ap-northeast-1c
  * サブネット：db-subnet,db-subnet2

### RDSの作成

* サービスメニューからRDSを選択
* データベースの作成ボタンを押す
  * データベース作成方法を選択：標準作成
  * エンジンのオプション
    * エンジンのタイプ：MySQL
    * エディション：MySQL Community（デフォルト）
    * エンジンのバージョン：MySQL 8.0.33（デフォルト）
  * テンプレート：無料利用枠
  * 可用性と耐久性：無料利用枠の場合は選択できない
  * 設定
    * DBインスタンス識別子：任意の名前（AWSリージョン内で一意の必要あり）
    * マスターユーザー名：admin
    * マスターパスワード：任意
  * インスタンスの設定
    * db.t2.micro（無料枠内）
  * ストレージ
    * テスト用なので、共に最小値を選択
      * 汎用SSD(gp2)
      * ストレージ割り当て：20
    * ストレージの自動スケーリング：無効
* 接続
  * コンピューティングリソース：EC2コンピューティングリソースに接続しない
    * ※ EC2に接続を選択した場合、自動的にセキュリティグループの設定が変更される。今回は既に手動設定済だったため接続しないを選択
  * VPC：subnetを作成したVPCと同じものを選択する
  * DB サブネットグループ：事前に作成済みの"db-subnet-group"を選択
  * パブリックアクセス：なし
  * 既存のVPCセキュリティグループ：事前に作成済みの"db-sg001"を選択
* 追加設定
  * データベースの選択肢
    * 最初のデータベース名：users
    * ※ ここを設定すると、RDSインスタンス起動時に自動的にデータベースを作成してくれる

## NATゲートウェイ

### NATゲートウェイの作成

NATゲートウェイを作成します。

NATを配置するサブネットは、ALBのあるパブリックサブネットとします。

* VPC < NATゲートウェイを作成を開く
* 名前：nat-gateway（任意）
* サブネット：alb-subnet
* 接続タイプ：パブリック
* Elastic IPの割り当てを押す

### ルートテーブルの紐付けを変更

アプリケーションサーバがNATゲートウェイを介してインターネットにアクセスできるようにするため、app-subnet(EC2インスタンスが属するプライベートサブネット)のルートテーブルを変更します。

* "app-subnet"のルートテーブル"app-rtb"を選択
* ルートを編集
  * 送信先：0.0.0.0/0
  * ターゲット：nat-gateway（上で作成したNAT）を選択

### MySQLクライアントをインストールし、DBへアクセスする

```bash
[ec2-user@ip-10-0-0-7 nginx]$ sudo docker exec -it コンテナ名 /bin/sh 
$ sudo apt update
All packages are up to date.
$ apt install default-mysql-client
$ mysql -V
mysql  Ver 15.1 Distrib 10.11.3-MariaDB, for debian-linux-gnu (x86_64) using  EditLine wrapper

$ mysql -h db-three-layer-sample.cjpmmmg6bimt.ap-northeast-1.rds.amazonaws.com -u admin -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 294
Server version: 8.0.33 Source distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases
    -> ;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| users              |
+--------------------+
5 rows in set (0.012 sec)
```
