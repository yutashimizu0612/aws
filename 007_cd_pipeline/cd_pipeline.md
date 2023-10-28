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
| | GitHub |

## Amazon ECRの設定

## CodeDeployの設定

## CodePipelineの設定