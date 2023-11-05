## GitHubリポジトリ

```
src
├── Dockerfile
├── index.html
buildspec.yml
```

```yml:buildspec.yml
version: 0.2

phases:
  pre_build:
    commands:
      - aws --version
      # 環境変数の設定
      - AWS_ACCOUNT_ID=086959954524 # TODO: CodeBuildの環境変数として設定する
      - ECR_ENDPOINT=${AWS_ACCOUNT_ID}.dkr.ecr.us-west-2.amazonaws.com
      - REPOSITORY_URI=${ECR_ENDPOINT}/test-repos # ECRに作成済みのリポジトリを指定
      # コミットハッシュの先頭7桁をタグに利用する
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
      # ECRへログイン
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin ${ECR_ENDPOINT}
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest ./src
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"hello-world","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
artifacts:
    files: imagedefinitions.json
```

## AWS CodeBuildの設定

### AWS CodeBuildとは

AWSの提供するビルド関連のフルマネージドサービスです。自動デプロイのプロセスの中でビルドの工程を担当するサービスとなります。

具体的には、

* ソースコードのコンパイル
* 単体テストの実行
* ビルドの成果物(アーティファクト)の生成

などを実施し、これらの処理を自動化できます。

ここで生成されたアーティファクトは、すぐにデプロイできる状態のものです。

https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/welcome.html

### ビルドプロジェクトの作成

AWSコンソールからCodeBuildページに行き、「ビルドプロジェクトを作成する」を押します。

| プロジェクトの設定 | |
| ---- | ---- |
| プロジェクト名| nginx-test-build |

ビルド対象となるリポジトリを選択します。今回はGitHubを使っているので、GitHubを選択しています。

| ソース | |
| ---- | ---- |
| ソースプロバイダ | GitHub |
| GitHub リポジトリ | 指定のリポジトリ |

| 環境 | | |
| ---- | ---- | ---- |
| 環境イメージ | カスタムイメージ | |
| 環境タイプ | Linux | 今回使用するイメージはAMD64アーキテクチャ向けのため、環境タイプはLinuxを選択します。|
| イメージレジストリ | Amazon ECR | |
| ECRアカウント | 自分のECRアカウント | |
| Amazon ECR レポジトリ | 任意のECRリポジトリ | |
| Amazon ECR イメージ | latest | |
| 🌟認証情報プルイメージ | AWS CodeBuild認証情報 | |
| 🌟特権付与 | チェックする |  |
| サービスロール | 新しいサービスロール | |
| ロール名 | codebuild-nginx-test-build-service-role(自動入力のまま) | |

| Buildspec | | |
| ---- | ---- | ---- |
| ビルド仕様 | buildspecファイルを使用する | buildspec.ymlにbuildコマンドを入れているため |
| Buildspec名 | 空欄 | buildspecファイルに`buildspec.yml`以外の名前とルートディレクトリ以外の場所を指定する場合はここで指定する |

| アーティファクト | | |
| ---- | ---- | ---- |
| タイプ | アーティファクトなし | |

上記の条件で、ビルドプロジェクトを作成します。

## CodeDeployの設定

### アプリケーションの作成

AWSコンソールからCodeDeployページに行き、「アプリケーションの作成」を押します。

| アプリケーションの設定 | | |
| ---- | ---- | ---- |
| アプリケーション名 | nginx-test | |
| コンピューティングプラットフォーム | Amazon ECS | |

上記の条件で、アプリケーションを作成します。

### 必要なロールの作成

デプロイグループの作成時に**サービスロール**というものを指定する必要があります。

ここには、**CodeDeployサービスが他のAWSサービスを操作するための権限を持つIAMロール**を指定します。

IAMページから簡単に作成できるので、事前に作っておきます。

IAM > ロール > ロールの作成を押します。

| 信頼されたエンティティを選択 | | |
| ---- | ---- | ---- |
| 信頼されたエンティティタイプ | AWSのサービス | |
| サービスまたはユースケース | CodeDeploy | |
| ユースケース | CodeDeploy - ECS | |

次へを押します。

| 許可ポリシー | | |
| ---- | ---- | ---- |
| ポリシー名 | AWSCodeDeployRoleForECS | 自動選択されています |

次へを押します。

| 名前、確認、および作成 | | |
| ---- | ---- | ---- |
| ロール名 | CodeDeployRoleForECS | 任意 |

このロールを引き受けるのはCodeDeployサービスなので、信頼ポリシーにもAWSCodeDeployが指定されています。

ロールを作成します。

### ターゲットグループの作成

| 基本的な設定 | | |
| ---- | ---- | ---- |
| ターゲットタイプの選択 | IPアドレス |  |
| ターゲットグループ名 | green-ecs-target-group |  |
| プロトコル | HTTP |  |
| ポート | 80 |  |
| IPアドレスタイプ | IPv4 |  |
| VPC | 対象となるECSが属するネットワークを選択 |  |
| プロトコルバージョン | HTTP1 |  |

次へを押します。

IPアドレスを追加する項目がありますが、空欄にします。Fargateを使用する場合、特定のIPアドレスを指定する必要がないからです。

### ALBのリスナーの追加

| リスナーの追加 | | |
| ---- | ---- | ---- |
| プロトコル | HTTPS |  |
| ポート | 444 |  |
| アクションのルーティング | ターゲットグループへ転送 |  |
| ターゲットグループ | green-ecs-target-group |  |

※ HTTPSにする場合は適切な証明書の選択が必要

#### セキュリティグループの変更

ALBと関連付けしているセキュリティグループを編集して、HTTPS:443 のトラフィック受信を許可します。

該当のセキュリティグループ > インバウンドルールから「インバウンドルールを編集」を押します。

以下のルールを追加します。

| | |
| ---- | ---- |
| タイプ | カスタムTCP |
| ポート | 444 |

### ECSサービスの作成

ECSサービスを作成します。

以前作成したECSサービスは、デプロイタイプが`ローリングアップデート`(デフォルト)となっています。

ブルーグリーンデプロイを行うためには、この設定を変更する必要があります。AWSコンソールからECSサービスのデプロイタイプを変更できないため、作成し直します。

| デプロイオプション | | |
| ---- | ---- | ---- |
| デプロイタイプ | ブルー/グリーンデプロイ(AWSCodeDeployを使用) | |
| デプロイ設定 | CodeDeployDefault.ECSAllAtOnce | デフォルトのまま |
| CodeDeployのサービスロール | CodeDeployRoleForECS | 上で作成したロールを指定します |

デプロイタイプを`ブルー/グリーンデプロイ(AWSCodeDeployを使用)`にすると、ロードバランシングのセクションでターゲットグループを2つ指定できるようになります。

### デプロイグループの作成

続いて、アプリケーションページからデプロイグループを作成します。

| | | |
| ---- | ---- | ---- |
| デプロイグループ名 | nginx-test-deploy | |
| サービスロール | CodeDeployRoleForECS | 上で作成したロールを指定します |
| ECS クラスター名 | nginx-dev-cluster | |
| ECS サービス名 | nginx-test-server | |

| Load balancer | | |
| ---- | ---- | ---- |
| Load balancer | sample-alb | |
| 本稼働リスナーポート | HTTP:80 | |
| ターゲットグループ | ecs-target-group | |

## CodePipelineの設定
