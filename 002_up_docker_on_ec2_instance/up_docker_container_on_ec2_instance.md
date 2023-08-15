## ポートマッピング

dockerホスト（DockerEngineが動作しているマシン）のポートとコンテナのポートをマッピングすること。これにより、外部からDockerコンテナにアクセスできるようになる。

```yml
    # EC2で公開するポート番号 ： コンテナのポート番号
    ports:
      - '8080:8080'
```

Dockerコマンドで実行する場合は、`docker run -p 8080:80 ...`など`-p`オプションで指定可能。

## ボリュームマウントとバインドマウント

### 前提

* コンテナ内のデータは、コンテナが削除されると失われる。

### dockerにおけるvolumeとは

* 永続的なデータ保持や複数コンテナでのデータ共有のためのデータストレージ
  * コンテナが削除されても、ボリュームはそのまま（コンテナとライフサイクルが別）

### ボリュームマウント

* docker engineの管理下にあるvolumeをコンテナにマウント(取り付ける)やり方
* コンテナと異なるライフサイクルのため、コンテナが削除されてもボリュームは削除されない。このため、データ永続化に使われる

```yml
services:
  db:
    volumes:
      # MySQLのデータを永続化する(ボリュームマウント)
      # コンテナ外のDockerVMのvolumeのdb-dataという領域にデータが保持される
      - 'db-data:/var/lib/mysql'
      - './docker/db/config/my.cnf:/etc/mysql/conf.d/mysql.cnf'
      - './docker/db/initial_db:/docker-entrypoint-initdb.d'
volumes:
  db-data:
```

### バインドマウント

* ローカルPCなど、docker engineが管理していないディレクトリやファイルにコンテナをマウントするやり方

```yml
  client:
    volumes:
      - ./client/:/usr/src/app
  api:
    volumes:
      - ./server:/usr/src/app
```

### 無名ボリューム

* ホストのディレクトリを指定しない方法
  * 指定したディレクトリやファイルをホストのファイルシステムではなく、docker engineが管理するボリュームにマウントする
  * バインドマウントを使ってアプリケーションをコンテナにマウントするとき、特定のディレクトリを除外したいケースに使う

```yml
    volumes:
      - ./server:/usr/src/app
      - /usr/src/app/node_modules
```

上の例では、`/usr/src/app`配下のディレクトリとファイルはホストの`./server`の内容と全く同じになるが、`/usr/src/app/node_modules`だけはコンテナ側の（ボリュームに保存されている）データを参照する。

## ブリッジネットワーク

* Dockerのネットワークモードの1つ
* デフォルトでは、コンテナはこのネットワークに接続されている
* ホスト上のネットワークとは別物のプライベートなネットワーク
  * ポートマッピングをすることで、ホスト側からアクセスが可能となる
* ネットワーク内部の各コンテナは、それぞれ疎通が可能
  * ブリッジネットワークでは、他のホスト上で起動するコンテナと通信することはできない

### 名前解決の方法

```yml
services:
  api:
    image: node:14.15.4-alpine3.10
    networks:
      - sample_bridge
  db:
    image: mysql:5.7
    networks:
      - sample_bridge
networks:
  sample_bridge:
    driver: bridge
```

上記では、`sample_bridge`というカスタムブリッジネットワークを作成し、apiとdbコンテナがそのネットワークに接続するように設定している。この2つのコンテナは、互いのサービス名(api,db)で通信できる。

```js
app.get('/db', async (req, res) => {
  try {
    // apiコンテナ内で、dbというコンテナ名で通信できる（同じsample_bridgeというカスタムブリッジネットワークだから）
    const response = await axios.get('http://db:3306');
    res.send(response.data);
  } catch (error) {
    res.status(500).send('Failed to fetch from database service');
  }
});
```

### dockerにおける他のネットワークの種類

* Host:ホスト
  * コンテナがHost上のネットワークにそのまま接続する
  * ホストのポートをそのまま使うため、衝突することがある
  * NATを介さないため、高速
* None
  * ネットワーク接続不可

## dockerとdocker-composeの使い分け

* 複数のコンテナを管理する場合にはdocker-composeが便利
  * アプリケーション全体が一目で把握できる
  * 1つのコマンドで全てのコンテナを起動・停止できる
* イメージをカスタマイズして使う場合は、DockerFileを使う
  * docker hubにあるイメージをそのまま使う場合はDockerFileはなくても問題ない

## DockerFileの書き方

```DockerFile
# ベースとなるイメージを指定する
FROM node:14.15.4-alpine3.10

# 以降のコマンドは、ここで指定したディレクトリで実行される
WORKDIR /usr/src/app

# ホストのファイルやディレクトリをコンテナにコピーする
# 詳細に言うと、ホストのファイルやディレクトリがイメージの新しいレイヤーにコピーされる。そのイメージからコンテナが起動すると、コンテナにも指定したホストファイルやディレクトリが存在する
COPY ./package.json ./
COPY ./yarn.lock ./

# シェルコマンドの実行
RUN yarn install

# コンテナのポートの宣言
# これを指定してもポートが公開されるわけではなく、ポートを公開するためには`docker run -p`や`docker-compose.yml`での指定が必要
# あくまでもドキュメントとしての役割
EXPOSE 8080

# コンテナが起動したときに実行されるコマンド
CMD [ "yarn", "start" ]
```
