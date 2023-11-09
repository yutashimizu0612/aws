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
      - ECR_ENDPOINT=${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com
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
| 環境イメージ | マネージド型イメージ | |
| コンピューティング | EC2 | |
| オペレーティングシステム | Amazon Linux | |
| ランタイム | Standard | |
| イメージ | amazonlinux2-x86_64-standard:5.0 | |
| 🌟特権付与 | チェックする |  |
| サービスロール | 新しいサービスロール | |
| ロール名 | codebuild-nginx-test-build-service-role(自動入力のまま) | |

| Buildspec | | |
| ---- | ---- | ---- |
| ビルド仕様 | buildspecファイルを使用する | buildspec.ymlにbuildコマンドを入れているため |
| Buildspec名 | 空欄 | buildspecファイルに`buildspec.yml`以外の名前とルートディレクトリ以外の場所を指定する場合はここで指定する |

| アーティファクト | | |
| ---- | ---- | ---- |
| タイプ | アーティファクトなし | なしを選択するとS3に保存されません。CodePipelineを使う場合は次のステージに自動的に成果物(アーティファクト)が渡されるためS3に保存する必要はありません。 |

上記の条件で、ビルドプロジェクトを作成します。

### IAMポリシーの作成とロールへの追加

ビルドプロジェクト作成時に併せて作成した`codebuild-nginx-test-build-service-role`に**ECRへのログインとイメージをプルするポリシー**を追加します。これによって、CodeBuildはECRにログインしてイメージをプルできるようになります。

IAM > ポリシーからポリシーの作成を押します。

JSONを選択し、下記を入力します。

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "ECRPullPolicy",
			"Effect": "Allow",
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability"
            ],
			"Resource": "arn:aws:ecr:ap-northeast-1:{your_account_id}:repository/{your_ecr_name}"
		},
        {
            "Sid": "GetAuthToken",
            "Effect": "Allow",
            "Action": "ecr:GetAuthorizationToken",
            "Resource": "*"
        }
	]
}
```

ポリシー名を`TestReposECRPullPolicy`とし、ポリシーを作成します。

`codebuild-nginx-test-build-service-role`のページに行き、許可を追加 > ポリシーのアタッチを選択します。

### ビルドのテスト

必要なポリシーをCodeBuildに持たせたので、今作成したCodeBuildが正しく動作するかを確認します。

PROVISIONINGで失敗しました。**ECRのイメージをプルできていない**ことが原因のようです。

```
BUILD_CONTAINER_UNABLE_TO_PULL_IMAGE: Unable to pull customer's container image. CannotPullContainerError: Error response from daemon: pull access denied for {my_account_id}.dkr.ecr.ap-northeast-1.amazonaws.com/test-repos, repository does not exist or may require 'docker login': denied: User: CodeBuild
```

#### 🌟原因

ビルドプロジェクト作成時に環境イメージに**カスタムイメージ**を選択し、自分でECRリポジトリやイメージを指定していたのが原因でした。

環境イメージを**マネージド型イメージ**に変更して、再度ビルドすると成功しました。

生成されたアーティファクトを確認してみようと思いましたが、どこにもありませんでした。

ビルドプロジェクト作成時に**アーティファクトなし**を選択しているため、S3バケットにはアーティファクトは保存されないようです。

## CodeDeployの設定

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

### 必要なロールの作成

この後ECSサービスを作成する際に、**サービスロール**というものを指定する必要があります。

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

この条件で、ECSサービスを作成すると、CodeDeployのアプリケーションとデプロイグループが自動で作成されます。

## CodePipelineの設定

AWSコンソールからCodePipelineページに行き、「パイプラインを作成する」を押します。

| パイプラインの設定を選択する | | |
| ---- | ---- | ---- |
| パイプライン名 | nginx-test-deploy | |
| 🌟パイプラインタイプ | V2 | パイプライントリガーをGitタグにするにはV2である必要があるため、V2を選択 |
| サービスロール | 新しいサービスロール | |
| ロール名 | AWSCodePipelineServiceRole-ap-northeast-1-nginx-test-deploy | 自動入力 |

| ソースステージを追加する | | |
| ---- | ---- | ---- |
| ソースプロバイダー | GitHub(バージョン2) | |
| リポジトリ名 | 該当のGitHubのリポジトリ | |
| パイプライントリガー | Gitタグ | |
| タグ > 含める | v* | `v1.0.1`などの形式で指定した場合のみにパイプラインを発動させるために指定 |
| タグ > 除外する | 空欄 | |
| デフォルトブランチ | main | |
| 🌟出力アーティファクト形式 | CodePipelineのデフォルト | |