# まとめ

## IAMユーザー、 IAMロール、IAMポリシーについて理解する

### IAMユーザー

* AWSサービスにアクセスする際に使用するユーザーのこと
* 永続的に使用されるユーザー名やパスワードなどの資格情報を持つ
* ユーザーやアプリケーションがAWSのコンソールやAPIにアクセスする際に使用される

### IAMロール

* AWSリソースに一時的にアクセス権限を付与するためのもの
* IAMユーザーと異なり、ユーザー名やパスワードなどの永続的な資格情報を持たない
  * 代わりに、必要なタイミングで一時的に使用するセキュリティトークンが発行される

### IAMポリシー

* AWSリソースに対するアクセス権限を定義したもの
* IAMユーザー、IAMロール、IAMグループに関連付けして使用する
* 複数のポリシーを意味のある単位でまとめたものがIAMロールと考えることができる
  * ポリシーを一つも持たないIAMロールは何の権限も持たないため、実質的には意味がない

## IAMポリシーのJSON ポリシー

下記の例が定義していることをまとめると、

* bucket-for-ec2というS3バケットとその中のオブジェクトに対する(Resource)
* 書き込み、読み取り、一覧表示というアクションを(Action)
* 許可する(Effect)

ということになる。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::bucket-for-ec2/*",
                "arn:aws:s3:::bucket-for-ec2"
            ]
        }
    ]
}
```

### Effect

次の項目以降で指定するアクションを"許可する(Allow)"のか"拒否する(Deny)"のかを指定する項目

### Action

AWSサービスのアクションのリスト。
例えば、

* ec2:RunInstances: インスタンスを起動する。
* ec2:DescribeSecurityGroups: セキュリティグループの詳細情報を取得する。
* s3:ListBucket: S3バケットの一覧表示をする

など無数のアクションがある。

### Resource

ここで定義するポリシーが適用されるAWSリソース(Amazon Resource Name)のリスト。
今回の例だと、書き込み、読み取り、一覧表示が許可される先のS3バケットとS3バケット内のオブジェクトが指定されている。

## IAMロールのAssume Role

IAMに関連するアクションの1つで、AWSリソース(EC2やLamdaなど)が特定のIAMロールを一時的に引き受ける(assume)アクション。EC2などのリソースは、IAMロールをAssumeすることで、ロールに紐づく権限を取得できる。

### 信頼ポリシー

IAMロールを引き受ける(assume)ことができるリソースを指定する**信頼ポリシー**というものがある。これは、IAMロールごとに設定するもの。当然ながら、どんなリソースでもIAMロールをAssumeできてしまうのはセキュリティ的に問題があるので、この信頼ポリシーを設定しなければこのIAMロールは誰も引き受けることはできない。

下記は、特定のEC2インスタンス(id-aaaaaaabbbbbccccc)のみがIAMロールを引き受けられるように設定した信頼ポリシー。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "ec2:InstanceID": "id-aaaaaaabbbbbccccc"
        }
      }
    }
  ]
}
```

# 構築手順

## S3バケットを作成する

### S3バケットとは

S3バケットは、Amazon S3(Simple Storage Service)上でデータを保存するためのコンテナのことです。S3では、ファイルやデータのようなオブジェクトをバケット内に保存できます。バケットはデータを整理するフォルダのようなイメージです。

Azureで言うと、Blob Storageサービスに該当します。

### 作成手順

Amazon S3ページから「バケットを作成」を選択します。

* 一般的な設定
  * **バケット名**: bucket-for-ec2
  * **AWS リージョン**: アジアパシフィック（東京）ap-northeast-1
* オブジェクト所有者
  * ACL無効(推奨)に✅
* このバケットのブロックパブリックアクセス設定
  * パブリックアクセスを全てブロックに✅
* バケットのバージョニング
  * **バケットのバージョニング**: 無効にするに✅
* 🌟デフォルトの暗号化
  * **暗号化タイプ**: Amazon S3 マネージドキーを使用したサーバー側の暗号化 (SSE-S3)
  * **バケットキー**: 有効にする

「バケットを作成」ボタンを押します。

#### 暗号化のタイプ

暗号化のタイプの項目で、**(SSE-S3)**と**SSE-KMS**が選択できます。SSE-S3は、AmazonS3側でキーの管理を行なってくれるタイプです。一方で、SSE-KMSは、AWSのキーマネジメントサービス(**KMS**)を使って、ユーザー自身でキーを管理するタイプです。

#### 🌟バケットキーの有効化・無効化

バケットキーを有効化すると、**バケット全体で1つのKMSキーを使用**するようになります。一方で、無効化すると、各S3オブジェクト（ファイル）ごとにKMSキーが使用されます。バケットキーを無効化した状態で大量データをS3バケットに保存する場合、KMSキーが大量に作られ、KMSへのリクエストも大量に発生します。

## EC2からS3バケットに対する操作を行う

今回、EC2からS3バケットに対して、**一覧表示・データ取得・書き込み**の操作ができるようにします。これを実現するには、次の3ステップを踏む必要があります。

1. 特定のS3バケットに一覧表示・データ取得・書き込みする権限を持つカスタムポリシーを作成する
2. IAMロールを作成し、1で作成したポリシーを関連付けする
3. 2のIAMロールをEC2インスタンスに関連付けする

この手順を踏むことで、EC2インスタンスからS3バケットへの**一覧表示・データ取得・書き込み**の操作ができるようになります。

では、実際にやっていきます。

### ポリシーを作成する

AWSコンソールからIAMページに移動し、ポリシーを選択します。

* ポリシーを作成を押す
* アクセス許可を指定
  * **サービスを選択**: S3を選択
  * **アクセスレベル**: 以下の3つを選択
    * リスト: ListBucket
    * 読み取り: GetObject
    * 書き込み: PutObject
* リソース
  * 特定を選択
  * ARNを追加
    * **bucket**
      * 任意のbucket name: bucket-for-ec2(先ほど作成したS3バケット)
      * 任意のリソース: bucket-for-ec2
    * **object**
      * 任意のbucket name: bucket-for-ec2
      * 任意のobject name: *（bucket-for-ec2の全てのオブジェクトを指定）
* ポリシー名: PolicyForTestBucket

ポリシーを作成します。カスタマー管理のポリシーが追加されているのが分かります。

image

アクセス許可で**何をできるか**を指定しています。一方、リソースでは**どのリソースに対して**を指定しています。このケースでは、先ほど作成したbucket-for-ec2というS3バケットを指定しました。このポリシーは、**bucket-for-ec2というS3バケットに対して、リスト表示、読み取り、書き込みができる**ことを示しています。

#### ポリシーをJSONで確認する

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::bucket-for-ec2/*",
                "arn:aws:s3:::bucket-for-ec2"
            ]
        }
    ]
}
```

### EC2用のIAMロールを作成する

AWSコンソールからIAMページに移動し、ロールを選択します。

* ロールを作成を押す
* 信頼されたエンティティタイプ： AWSのサービス
* ユースケース
  * **サービスまたはユースケース**: EC2
  * 🌟**ユースケース**: EC2
* 許可を追加
  * 許可ポリシー: PolicyForTestS3Bucket（先ほど作成したカスタムのポリシー）
* 名前、確認、および作成
  * ロール名: RoleForTestS3Bucket

ロールが作成できました。

### EC2がS3に対する権限がないことを確認する

EC2インスタンスにSSH接続して、S3バケット内のオブジェクトの一覧表示を試みます。

```bash
[ec2-user@ip-** ~]$ aws s3 ls s3://bucket-for-ec2
Unable to locate credentials. You can configure credentials by running "aws configure".
```

ご覧の通り、EC2からS3バケットを操作する権限がないことが分かります。

### EC2にIAMロールをアタッチする

EC2のインスタンスのページに移ります。

* IAMロールを紐付けたいEC2インスタンスにチェックを付けます
* アクション > セキュリティ > IAMロールを変更
* **IAMロール**: RoleForTestS3Bucket（先ほど作成したIAMロール）

IAMロールの更新を押します。

EC2インスタンスの詳細ページのセキュリティタブから関連付けされたIAMロールが確認できます。

image

## （仮）EC2からS3への操作を行う

テスト用のファイルを作成して、S3バケットにアップロードできることを確認します。

```bash
[ec2-user@ip-** ~]$ touch test.txt
[ec2-user@ip-** ~]$ aws s3 cp test.txt s3://bucket-for-ec2/documents/test.txt
upload: ./test.txt to s3://bucket-for-ec2/documents/test.txt
```

続いて、一覧表示できることを確認します。

```bash
[ec2-user@ip-** ~]$ aws s3 ls s3://bucket-for-ec2
[ec2-user@ip-** ~]$ aws s3 ls s3://bucket-for-ec2/documents/
2023-09-23 02:34:44          0 test.txt
```

先ほどアップロードしたファイルがあることが確認できました。

最後に、ファイルをダウンロードします。

```bash
[ec2-user@ip-** ~]$ aws s3 cp s3://bucket-for-ec2/documents/test.txt test_downloaded.txt
download: s3://bucket-for-ec2/documents/test.txt to ./test_downloaded.txt
[ec2-user@ip-** ~]$ ls
test.txt  test_downloaded.txt
```

S3バケットにある`test.txt`をEC2ローカルに`test_downloaded.txt`としてダウンロードされたことが確認できました。
