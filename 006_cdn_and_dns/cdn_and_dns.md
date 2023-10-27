## 独自ドメインを設定する

### お名前.comでドメインを取得する

お名前.comで`sy-test-domain.com`というテスト用ドメインを取得しました。

### Route 53でホストゾーンを作成する

AWSコンソールからRoute53のページに行きます。

「ホストゾーンを作成」を選択します。

* ホストゾーン設定
  * **ドメイン名**: `sy-test-domain.com`
  * **タイプ**: パブリックホストゾーン

### Route 53が提供するネームサーバをお名前.comに登録する

#### この作業の目的

多くの場合、ドメインを購入するとドメイン登録業者のDNSサーバがネームサーバとして設定されています。つまり、デフォルトではドメインの名前解決はその業者のDNSサーバによって行われるということです。Route 53をDNSサービスとして利用する場合、このネームサーバをドメイン登録業者のものからRoute 53のものに変更する必要があります。

この手順を実施することで、Route 53の提供するネームサーバが名前解決を行うようになります。

#### 登録方法

作成したホストゾーンの詳細ページで、ネームサーバ(NS)が確認できます。

ここに記載されたネームサーバは、Route 53が提供するDNSサーバであり、今回取得した独自ドメインの名前解決を担当するサーバです。

これらのネームサーバをドメインを取得したお名前.comに登録します。

トップページ > ネームサーバの設定 > 他のネームサーバを利用の流れで選択していくと、ネームサーバを入力するフォームが出てきます。ここに、Route 53で確認した4つのネームサーバを入力します。

### 独自ドメインとALBを紐づける

ホストゾーンにターゲットをALBとするAレコードを追加します。

* **レコード名**: 空白 
* エイリアスをonにする
* **トラフィックのルーテーティング先**: Application Load Balancer 
* **リージョン**: アジアパシフィック（東京） 
* **ロードバランサーを選択**: 事前に作成済みのALBを選択 

この手順によって、独自ドメインにアクセスすると、Route53を介してALBにルーティングされるようになります。

#### Aレコードとは

**ドメイン名をIPアドレスにマッピンするためのDNSレコード**のことです。ドメイン名に対応するIPv4アドレスを指定します。Route 53を使用する場合は、IPv4アドレスを直接指定するのではなく、ALBやCloudFrontなど特定のAWSリソースを直接指定することができます（エイリアスAレコード）。

エイリアスAレコードを使用すると、ALBやCloudFrontの動的なIPアドレスの変更にも自動で対応してくれます。

### アクセスできるかを確認する

`http://sy-test-domain.com`にアクセスすると、「Hello World!!」が表示されることが確認できました。

## HTTPSアクセスできるように設定する

続いて、`sy-test-domain.com`にhttpsでアクセスできるようにします。

### ACMでSSL/TLS証明書を取得する

#### SSL/TLS証明書のおさらい

以下の2つを役割を担う認証局から発行される電子証明書

* ウェブサイトの運営者の実在性を確認すること
* ブラウザとWebサーバ間で通信データの暗号化を行うこと

#### 証明書の取得方法

AWSコンソールから「AWS Private Certificate Authority」のページに行き、「AWS Certificate Manager」を選択します。その後、「証明書をリクエスト」を押します。

* **証明書タイプ**: パブリック証明書をリクエスト
* **ドメイン名**: `sy-test-domain.com`
* **検証方法**: DNS検証
* **キーアルゴリズム**: RSA 2048

リクエストを押すと、証明書リソースが作成されます。

#### CNAMEレコードをドメインのDNS設定に追加する

検証方法にDNS検証を選択した場合、ドメインのDNS設定にCNAMEレコードを追加する必要があります。証明書の詳細ページを見ると、CNAME名とCNAME値が確認できます。このCNAMEレコードをRoute53のホストゾーンのレコードから追加します。

Route 53 > ホストゾーン > sy-test-domain.comを開き、「レコードを作成」を押します。

* **レコード名**: 証明書のCNAME名(`sy-test-domain.com`部分を除く)
* **値**: 証明書のCNAME値

レコードを作成を押すと、CNAMEレコードが追加されました。

しばらくすると、証明書のステータスが「保留中の検証」から「成功」に変わります。10分程度掛かりました。

### ALBの設定を行う

ALB側で行う設定は以下の2つです。

* HTTPSリスナーを追加する
* セキュリティグループを更新する

#### HTTPSリスナーを追加する

ALBの詳細ページのリスナーセクションの「リスナーの追加」を押します。

* リスナーの詳細
  * **プロトコル**：HTTPS
    * HTTP/HTTPSのみ選択可能
  * **ポート**：443
* セキュアリスナーの設定
  * Default SSL/TLS server certificate
    * **Certificate source**: ACMから
    * **Certificate (from ACM)**: 先ほど作成した証明書を選択

追加を押すと、リスナーとルールにHTTPS:443が追加されました。

到達不可能と表示されてますが、これはALBと関連付けしてあるセキュリティグループがHTTPS:443のトラフィックを許可していないためです。

#### セキュリティグループを更新する

ALBと関連付けしているセキュリティグループを編集して、HTTPS:443のトラフィック受信を許可します。

該当のセキュリティグループ > インバウンドルールから「インバウンドルールを編集」を押します。

以下のルールを追加します。

* **タイプ**: HTTPS 
* **ソース**: 0.0.0.0/0

「ルールを保存」を押すと、インバウンドルールにHTTPSが追加されます。

### HTTPSでアクセスできることを確認する

これまでの設定で`sy-test-domain.com`にHTTPSでアクセスできるようになりました。

`https//sy-test-domain.com`にアクセスして確認します。

期待通り「Hello World!!」が表示され、証明書も確認できました。

## CloudFrontをALBの前段に設置する

### CloudFront用のSSL/TLS証明書を取得する

CloudFrontはグローバルサービスのため、米国東部(バージニア北部)リージョンで発行されたACMのみ使用できます。そのため、ALBに紐付けた証明書と同内容の証明書を米国東部(バージニア北部)リージョンで再度作成します。

* **証明書タイプ**: パブリック証明書をリクエスト
* **ドメイン名**: `sy-test-domain.com`
* **検証方法**: DNS検証
* **キーアルゴリズム**: RSA 2048

CNAME名とCNAME値はALB証明書の時と同内容です。既にCNAMEレコードにこの名前と値は登録済みですので、CNAMEレコードの追加は不要です。しばらくすると証明書のステータスが成功になりました。

### CloudFrontディストリビューションを作成する

AWSコンソールのCloudFrontの「CloudFrontディストリビューションを作成」を押します。

* **オリジンドメイン**: ALBを選択します
* **プロトコル**: HTTPSのみ
* **オブジェクトを自動的に圧縮**: Yes
* **ビューワープロトコルポリシー**: HTTPS only
* **キャッシュキーとオリジンリクエスト**: `Cache policy and origin request policy (recommended)`にチェック
* **キャッシュポリシー**: CachingOptimized
* **ウェブアプリケーションファイアウォール (WAF)**: セキュリティ保護を有効にしない（WAFは有料のため）
* **料金クラス**: 北米と欧州のみを使用
* **代替ドメイン名(CNAME)** : `sy-test-domain.com`
* **カスタム SSL 証明書**: 米国東部(バージニア北部)リージョンで作成した証明書

「ディストリビューションを作成」を押します。

### Route 53のエイリアスレコードを更新する

現在、ALBが指定されているAレコードを修正し、CloudFrontに変更します。

これによって、`sy-test-domain.com`にアクセスすると、ALBではなくCloudFrontにルーティングされるようになります。結果として、CloudFront > ALB > ECSという順番でルーティングされ、nginxによる「Hello World!!」が表示されるはずです。

### CloudFront設定後の動作確認

`sy-test-domain.com`にアクセスして、「Hello World!!」が表示されることを確認します。

#### 502エラー

```
CloudFront wasn't able to connect to the origin. We can't connect to the server for this app or website at this time. There might be too much traffic or a configuration error. Try again later, or contact the app or website owner.
If you provide content to customers through CloudFront, you can find steps to troubleshoot and help prevent this error by reviewing the CloudFront documentation.
```

https://zenn.dev/devcamp/articles/f488e3d22ff63e

#### 403エラー

```
Bad request. We can't connect to the server for this app or website at this time. There might be too much traffic or a configuration error. Try again later, or contact the app or website owner.
If you provide content to customers through CloudFront, you can find steps to troubleshoot and help prevent this error by reviewing the CloudFront documentation.
```

https://ueyama.blog/?p=594

* デフォルトルートオブジェクトに`index.html`を追加します。
* キャッシュキーとオリジンリクエストのオリジンリクエストポリシーに`AllViewer`を選択して、保存します。

#### キャッシュと圧縮の確認

CloudFrontを経由することで、本当にキャッシュと圧縮がされているかを確認します。

開発者ツールのNetworkでResponseヘッダを確認し、`X-Cache: Hit from cloudfront`があることを確認します。これがあればキャッシュされています。

圧縮を示す内容が見当たらないため、Disable cacheにチェックを付けて再度読み込みます。すると、`Content-Encoding: br`が確認できました。

