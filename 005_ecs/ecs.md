## まとめ

### ワードの理解

#### タスク

タスク定義の内容に従って、実際に実行されるコンテナのインスタンス

#### タスク定義

* コンテナとその設定を定義するもの
* docker-compose.ymlで定義することとほぼ同じ
* 複数のコンテナを定義できる

#### サービス

タスク定義を用いて、タスクの実行と管理を行う役割を担う。

#### 🌟クラスター

他のECSリソースを束ねる論理的なエンティティ？

## Amazon ECR(Amazon Elastic Container Registry)とは

AWSが提供するコンテナイメージを管理保管できるサービスです。

https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/what-is-ecr.html

## Dockerイメージを作成する

Amazon ECRへプッシュするために、今回使用するコンテナのイメージをビルドします。

### イメージをビルドする

```Dockerfile
FROM nginx:latest

COPY index.html /usr/share/nginx/html

EXPOSE 80
```

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>Title</title>
</head>
<body>
  <h1>Hello World!!</h1>
</body>
</html>
```

上記の2つのファイルを作成し、下記コマンドを実行します。

```bash
docker build -t nginx-test .
```

## ECRにリポジトリを作成する

AWSコンソールからリポジトリのページに行き、「リポジトリを作成」を押します。

* **リポジトリ名**: test-repos
* 他の設定はデフォルトのまま

## ECRにプッシュする

### AWS CLIのインストールと初期設定

### Docker CLIを使って、ECRにログインする

ECRはDockerイメージを管理するリポジトリです。そこにDockerイメージをプッシュするために、Docker CLIからECRにログインする必要があります。

```bash
aws ecr get-login-password --region ap-northeast-1 --profile ECRLoginAndPushRoleProfile | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.ap-northeast-1.amazonaws.com
```

### Dockerイメージにタグ付けする

```bash
$ docker tag nginx-test:latest <your-account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/test-repos:latest
```

```-
$ docker images
REPOSITORY                                                     TAG       IMAGE ID       CREATED       SIZE
<your-account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/test-repos              latest    57289*******   7 days ago    192MB
nginx-test                                                     latest    57289*******   6 days ago    192MB
```

#### タグ付けとは何か？

タグ付けは、**既存のイメージに新しい名前(タグ)を付けること**を指します。イメージが新しく作成されたようにも見えますが、ローカルに存在するイメージに新しい名前(タグ)が増えただけです。そのため、"IMAGE ID"を見ると、全く一緒であることが分かります。

```bash
$ docker tag 元となるイメージ[:TAG] タグ付けされた後のイメージ名[:TAG]
```

#### 何のためにタグ付けするのか？

前提として、ECRにプッシュするには`<your-account-id>.dkr.ecr.<region>.amazonaws.com/<repository-name> `のような名前である必要があります。この形式に従っていないイメージはそもそもプッシュできません。

ただ、ローカルでビルドを行う際に、最初からECRの形式に従った長々しい名前を付けるのは微妙です。そこで、最初にビルドする時は汎用的な名前を付けておき、実際にECRにプッシュする前にタグ付けをして、ECRにプッシュ可能な名前としています。

[Docker イメージをプッシュする](https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/docker-push-ecr-image.html)

### DockerイメージをECRにプッシュする

今タグ付けしたDockerイメージをECRにプッシュします。

```bash
$ docker push <your-account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/test-repos:latest
The push refers to repository [<your-account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/test-repos]
626d2ef86498: Pushed 
e005f3c090e0: Pushed 
3ccc534e961c: Pushed 
def14911cf6a: Pushed 
78cdcd2ba4bb: Pushed 
8cc0ce26d320: Pushed 
dcb816d1345e: Pushed 
1c3daa065742: Pushed 
latest: digest: sha256:***** size: 1985
```

## ALB

### ターゲットグループを作成する

AWSコンソールからターゲットグループのページに行き、ターゲットグループを作成します。

ターゲットグループは、**ALBがトラフィックを振り分ける対象先**のことを指します。今回、ALBにトラフィックが到達したら、nginxコンテナを起動するECSにトラフィックを振り分けてもらいたいです。そのため、ECS用のターゲットグループを作成します。

* 基本的な設定
  * **ターゲットタイプの選択**： IPアドレス
  * **ターゲットグループ名**： ecs-target-group
  * **プロトコル**： HTTP
  * **ポート**： 80
  * **IPアドレスタイプ**： IPv4
  * **VPC**： 対象となるECSが属するネットワークを選択
  * **プロトコルバージョン**： HTTP1
* IPアドレス
  * 特に何も指定しない

#### ターゲットタイプの選択

ECS、特にFargateを使用する場合、ターゲットタイプは**IPアドレス**を選択する必要があります。

#### ターゲットグループ作成時に指定するプロトコルとポートについて

ターゲットグループを作成するときのプロトコルとポートは、ALBがトラフィックを転送する先のターゲットのプロトコルとポートを指します。今回のケースでは、ECSタスクのFargateインスタンス内のコンテナのプロトコルとポート(nginxコンテナがリッスンしている80番ポート)です。

#### IPアドレス

IPアドレスを手動で入力する箇所がありますが、ここは空欄で問題ありません。Fargateを使用する場合、特定のIPアドレスを指定する必要がないからです。

タスクが起動するとタスク内で起動するコンテナ(今回はnginxコンテナ)に動的にIPアドレスが割り当てられるのですが、ECSサービスが自動でターゲットグループにIPアドレスを登録してくれます。そのため、ターゲットグループには手動でIPアドレスを指定する必要はないのです。

ただし、ECSサービスを作成するときに、ターゲットタイプがIPアドレスのターゲットグループを指定する必要はあります。後ほどECSサービスを作成するときに、ここで作成した`ecs-target-group`を指定します。

### ターゲットグループとロードバランサを関連付けする

* 対象のロードバランサを選択
* 「リスナーとルール」タブを選択
* リスナーの追加
  * プロトコル / ポート： HTTP 80
  * アクションの種類： ターゲットグループへ転送
  * ターゲットグループの選択： ecs-target-group

以下の要領でリスナーを追加します。

## ECS

### タスク定義の作成

AWSコンソールのECSのページに行き、タスク定義を選択します。

#### インフラストラクチャの要件

* **タスク定義ファミリー**： nginx-test
* **起動タイプ**： AWS Fargate
* **オペレーティングシステム/アーキテクチャ**： Linux/X86_64
* **タスクサイズ**
    * **CPU**： .5vCPU
    * **メモリ**： 1GB
* **タスクロール**： なし
* **タスク実行ロール**： 新しいロールの作成

nginxで「Hello world!」を表示するだけなので、最小構成とします。

##### タスクロールとは？

タスクロールは、**タスク内のコンテナがAWSリソースと通信する際に使用するロール**です。

起動した後のロール

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task-iam-roles.html

##### タスク実行ロールとは？

コンテナ起動時のロール

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task_execution_IAM_role.html

#### コンテナの定義

* **コンテナの詳細**
    * **名前**： nginx
    * **イメージURI** <your-account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/test-repos
* **ポートマッピング**
    * **コンテナポート**： 80
    * **プロトコル**： TCP
    * **ポート名**: nginx-80-tcp(自動設定されたもの)
    * **アプリケーションプロトコル**: HTTP

イメージURIには、先ほどプッシュした`<your-account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/test-repos`を指定します。コンテナポートにはnginxデフォルトの80番を指定します。

ポートマッピングと言いつつ、ホスト側ポートを指定する箇所がありません。これは、**Fargateがホストポートを動的に割り当てる**ためです。Fargateを使用すると、コンテナのインスタンスをユーザー側で管理する必要はなく、ポートの衝突を心配する必要もないようです。

* **リソース割り当て制限**
    * **CPU**： .5
    * **メモリのハード制限**： 1
    * **メモリのソフト制限**： 1

コンテナ1つ1つレベルでのリソース制限の設定です。nginxで「Hello world!」を表示するだけなので、最小構成とします。

作成します。

### ECSサービスを作成する

#### クラスターを作成する

AWSコンソールのECSページに行き、サイドバーからクラスターを選択します。

* クラスター設定
  * **クラスター名**： nginx-dev-cluster
* インフラストラクチャ
  * AWS Fargate (サーバーレス)にチェック

作成します。

#### サービスを作成する

作成したクラスターのページに行くと、サービスという項目があります。
サービス作成ボタンを押します。

* 環境
  * **既存のクラスター**： nginx-dev-cluster(さっき作ったクラスター)
  * **コンピューティングオプション**： 起動タイプにチェック
  * **起動タイプ**： FARGATEを選択
  * **プラットフォームのバージョン**： LATEST
* デプロイ設定
  * **アプリケーションタイプ**： サービス
  * **タスク定義**：
    * **ファミリー**： nginx-test(作成したタスク定義を選択)
    * **リビジョン**： 1(最新)
    * **サービス名**： nginx-test-server
  * **サービスタイプ**： レプリカにチェック
  * **必要なタスク**： 1
* ネットワーキング
  * **VPC**： ALBと同じVPC
  * **サブネット**： ALBとは異なる別のサブネットを選択
  * セキュリティグループ
    * 新しいセキュリティグループの作成
    * **セキュリティグループ名**： ecs-sg
    * セキュリティグループのインバウンドルール
      * **タイプ**： カスタムTCP
      * **ポート範囲**： 80
      * **ソース**： Source group
      * **値**： ALBに紐付けているセキュリティグループを選択
* ロードバランシング
  * **ロードバランサーの種類**： Application Load Balancer
  * 既存のロードバランサーを使用にチェック
  * **ロードバランサー**： sample-alb
  * **ロードバランス用のコンテナの選択**： nginx 80:80
  * **リスナー**
    * 「既存のリスナーを使用」にチェック
    * ポート 80 / プロトコル HTTP
  * ターゲットグループ
    * 「既存のターゲットグループを使用」にチェック
    * **ターゲットグループ名**： ecs-target-group

#### サービス作成時のエラー対応

サービスを作成しようとすると、`exec /docker-entrypoint.sh: exec format error`というエラーが発生しました。これは、**イメージをビルドしたローカルマシンとECSのアーキテクチャ**が異なることが原因です。

Apple SiliconのMacでは、デフォルトでARMベースのイメージがビルドされますが、多くのクラウドサービスやサーバーの環境はAMD64（x86_64）アーキテクチャ上で実行されています。このアーキテクチャ(ARM64 or AMD64)が合っていないと、コンテナ起動時にエラーが出ます。

対応として、`--platform linux/amd64`を付けてイメージをビルドし直して、再度ECRにプッシュします。

```bash
# AMD64で再ビルド
docker build -t nginx-test . --platform linux/amd64
# ログイン
aws ecr get-login-password --region ap-northeast-1 --profile ECRLoginAndPushRoleProfile | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.ap-northeast-1.amazonaws.com
# タグ付け
docker tag nginx-test:latest <your-account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/test-repos:latest
# 再プッシュ
docker push <your-account-id>.dkr.ecr.ap-northeast-1.amazonaws.com/test-repos:latest
```

※ 2023/10/08現在、タスク定義のOS/アーキテクチャは`Linux/ARM64`を選択できます。AWS側をARM64にしても良いですが、ここではAMD64（x86_64）で統一することとします。

#### コンテナの起動を確認

上記のエラー対応の後、再度サービスを作成します。

すると、サービスとその中のタスクが起動していることが確認できます。

ALBのドメインをブラウザに打ち込むと、Hello World!!が表示されることが確認できました。

#### 起動中のタスクを停止

起動し続けていると課金されるので、以下の手順でタスクを停止しておきます。

* クラスタのサービスタブを開きます。
* 停止したいサービスにチェックを入れて、更新ボタンを押します
* 必要なタスクを`0`にします
* 更新ボタンを押します

これによって、タスクが停止され、新しいタスクも起動されなくなります。
