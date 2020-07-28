# メニュー
1. システム構成
2. ネットワーク設定  
2.1. VPC  
2.2. サブネット  
2.3. Internet Gateway  
2.4. ルートテーブル  
2.5. セキュリティグループ
3. Bastionaサーバ
4. DBサーバ  
4.1. EC2インスタンス作成  
4.2. yumリポジトリファイル作成  
4.3. MongoDBインストール  
4.4. MongoDBパラメーター変更  
4.5. MongoDB起動  
5. Applicationサーバ  
5.1. EC2インスタンス作成  
5.2. Nodejsインストール
5.3. yarnインストール
5.4. gitインストール
5.5. Growiインストール
5.6. Growi自動起動設定
6. AMI作成
7. 起動テンプレート
8. ターゲットグループ
9. ロードバランサ
10. Auto Scalingグループ
11. 動作確認  
11.1. Growi  
11.2. Auto Scalingグループ


# システム構成
以下のようにAPPサーバがスケールするシステムを構築する

![システム構成](./img/system.png)

# ネットワーク設定
* VPC

  ネットワーク(CIDR): 10.0.0.0/16

* ネットワーク構成

  |種類             |ネットワーク(CDIR)|Route Table|Availability Zone|
  |:---------------|:---------------|:----------|:----------------|
  |Public  subnet 1|10.0.10.0/24    |Public     |us-east-1a       |
  |Public  subnet 2|10.0.20.0/24    |Public     |us-east-1b       |
  |Private subnet 1|10.0.30.0/24    |Private    |us-east-1a       |
  |Private subnet 2|10.0.40.0/24    |Private    |us-east-1b       |

* ルートテーブル

  |種類    |Destination|Target          |Note|
  |:------|:----------|:---------------|:----------------------------------|
  |Public |10.0.0.0/16|local           |                                   |
  |       |0.0.0.0/0  |Interbet Gateway|                                   |
  |Private|10.0.0.0/16|local           |                                   |
  |       |0.0.0.0/0  |NAT Gateway     |今回は、TargetにInternet Gatewayを指定|

* セキュリティグループ(SG)

  |種類         |Type     |Protocol|Port Range|Source                            |
  |:-----------|:---------|:-------|:---------|:---------------------------------|
  |Bastionサーバ|SSH       |TCP     |22        |BastionサーバにアクセスするPCのアドレス|
  |Webサーバ    |SSH       |TCP     |22        |BastionサーバのSG                  |
  |            |HTTP      |TCP     |80        |0.0.0.0/0                         |
  |APPサーバ    |SSH       |TCP     |22        |BastionサーバのSG                  |
  |            |Custom TCP|TCP     |3000      |WebサーバのSG                      |
  |DBサーバ     |SSH       |TCP     |22        |BastionサーバのSG                  |
  |            |Custom TCP|TCP     |27017     |APPサーバのSG                      |

## VPC作成
サービスメニューからVPCを選択して、VPCを作成する

1. 「VPCの作成」を押す

![VPC作成](./img/create_vpc1.png)

2. 以下の値を入力し、「作成」を押す
* 名前タグ: VPCの名前を指定（任意の値）
* IPv4 CIDR ブロック: VPCのネットワークアドレスを指定

![VPC作成](./img/create_vpc2.png)

## サブネット作成
サービスメニューからVPCを選択して、サブネットを4つ作成する
* Public subnet 1
* Public subnet 2
* Private subnet 1
* Private subnet 2

1. 「サブネットの作成」を押す

![サブネット作成](./img/create_subnet1.png)

2. 以下の値を入力し、「作成」を押す
* 名前タグ: サブネットの名前（任意の値）
* VPC: 「VPC作成」で作成したVPCを選択
* アベイラビリティーゾーン: アベイラビリティーゾーンを選択
* IPv4 CIDRブロック: サブネットのネットワークアドレスを指定

![サブネット作成](./img/create_subnet2.png)

## Internet Gateway作成
サービスメニューからVPCを選択して、Internet Gatewayを作成する

1. 「インターネットゲートウェイの作成」を押す

![インターネットゲートウェイ作成](./img/create_igw1.png)

2. 以下の値を入力し、「インターネットゲートウェイの作成」を押す
* 名前タグ: インターネットゲートウェイの名前（任意の値）

![インターネットゲートウェイ作成](./img/create_igw2.png)

3. 「VPCへアタッチ」を選択

![インターネットゲートウェイ作成](./img/create_igw3.png)

4. 以下の値を入力し、「インターネットゲートウェイのアタッチ」を押す

* 使用可能なVPC: 「VPC作成」で作成したVPCを選択

![インターネットゲートウェイ作成](./img/create_igw4.png)

## ルートテーブル作成
サービスメニューからVPCを選択して、ルートテーブルを2つ作成する
* public
* private

1. 「ルートテーブルの作成」を押す

![インターネットゲートウェイ作成](./img/create_rt1.png)

2. 以下の値を入力し、「作成」を押す
* 名前タグ: ルートテーブルの名前（任意の値）
* VPC: 「VPC作成」で作成したVPCを選択

![インターネットゲートウェイ作成](./img/create_rt2.png)

### ルートの編集
ルートを追加する

1. 作成したルートテーブルを選択して、「ルートの編集」を押す
![インターネットゲートウェイ作成](./img/create_rt3.png)

2. 「ルートの追加」を押して、以下を入力し、「ルートの保存」を押す
* 送信先: 0.0.0.0/0
* ターゲット: 通信の転送先を選択

![インターネットゲートウェイ作成](./img/create_rt4.png)

### サブネットの関連付け
ルートテーブルをサブネットに関連付ける  

1. 作成したルートテーブルを選択して、「サブネットの関連付けの編集」を押す

![インターネットゲートウェイ作成](./img/create_rt5.png)

2. ルートテーブルを関連付けるサブネットを選択肢、「保存」を押す

![インターネットゲートウェイ作成](./img/create_rt6.png)


## セキュリティグループ作成
サービスメニューからVPCを選択して、セキュリティグループを4つ作成する
* Bastionサーバ
* DBサーバ
* Appサーバ
* Webサーバ

1. 「セキュリティグループを作成」を押す

![セキュリティグループ作成](./img/create_sg1.png)

2. 以下を入力し、「セキュリティグループを作成」を押す
* セキュリティグループ名: セキュリティグループの名前（任意の値）
* 説明: セキュリティグループの説明（任意の値）
* VPC: 「VPC作成」で作成したVPCを選択
* インバウンドルール: 「ルールを追加」を押してインバウンドルールを追加

![セキュリティグループ作成](./img/create_sg2.png)

# Bastionサーバ作成
サービスメニューからEC2を選択して、Bastionサーバを作成する

1. 「インスタンスの作成」を押す

![Bastionサーバ作成](./img/create_bastion1.png)

2. Amazonマシンイメージ(AMI)の選択
「Amazon Linux 2 AMI」を選択

![Bastionサーバ作成](./img/create_bastion2.png)

3. インスタンスタイプの選択
「t2.micro」を選択し、「次のステップ: インスタンスの詳細の設定」を押す

![Bastionサーバ作成](./img/create_bastion3.png)

4. インスタンスの詳細の設定
以下を入力し、「次のステップ: ストレージの追加」を押す
* ネットワーク: 「VPC作成」で作成したVPCを選択
* サブネット: Public subnet 1を選択
* 自動割り当てパブリックIP: 「有効」を選択

![Bastionサーバ作成](./img/create_bastion4.png)

5. ストレージの追加
「次のステップ: タグの追加」を押す
__設定値は変更なし__

![Bastionサーバ作成](./img/create_bastion5.png)

6. タグの追加
「タグの追加」を押して、タグを追加し、「次のステップ: セキュリティグループの設定」を押す
* キー: Name
* 値: 任意の値

![Bastionサーバ作成](./img/create_bastion6.png)

7. セキュリティグループの設定
Bastionサーバ用のセキュリティグループを選択し、「確認と作成」を押す

![Bastionサーバ作成](./img/create_bastion7.png)

8. インスタンス作成の確認
値を確認して問題なければ、「起動」を押す

![Bastionサーバ作成](./img/create_bastion8.png)

9. キーペアの選択
キーペアを選択し、「インスタンスの作成」を押す

![Bastionサーバ作成](./img/create_bastion9.png)

# DBサーバ作成
## EC2インスタンス作成
* Amazon Machine Image: Amazon Linux 2 AMI
* Instance Type: t2.small
* ネットワーク: 「VPC作成」で作成したVPC
* サブネット: Private subnet 1
* 自動割り当てパブリックIP: 有効  
__「無効化」を選択してインターネットへのアクセスはNATゲートウェイを経由するのが正しいが、使用している環境ではNATゲートウェイが作れないため、「有効」を選択すること__

* タグ: キー: Name、値: Database Serverの名前（任意の値）
* セキュリティグループ: Database Server用のセキュリティグループ

## これ以降の作業はBastionサーバからDBサーバにSSHでログインして実施する
[Bastionサーバから他のEC2インスタンスにSSHログインする方法](./HowToLoginEC2.md)

## yumリポジトリファイル作成
以下のyumリポジトリファイルを作成する  
__※ sudoで実施する__

* ファイル  
/etc/yum.repos.d/mongodb-org-3.6.repo

* ファイルの内容
```
[mongodb-org-3.6]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/amazon/2013.03/mongodb-org/3.6/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.6.asc
```

## MondoDBインストール
以下のコマンドでMongoDBをインストールする

```
$ sudo yum install -y mongodb-org
```

## MongoDBパラメータ変更
他のサーバからアクセスできるようにBind IP Addressを変更する

* ファイル  
/etc/mongod.conf

* 変更内容
  * 変更前
    ```
    # network interfaces
    net:
      port: 27017
      bindIp: 127.0.0.1  # Listen to local interface only, comment to listen on all interfaces.
    ```

  * 変更後
    ```
    # network interfaces
    net:
      port: 27017
      bindIp: 0.0.0.0  # Listen to local interface only, comment to listen on all interfaces.

    ```

## MongoDB起動
以下のコマンドを実行してMongoDBを起動する

1. systemdの定義ファイル再読み込み
  ```
  sudo systemctl daemon-reload
  ```

2. MongoDBを起動
  ```
  $ sudo systemctl start mongod
  ```

3. MongoDB自動起動設定
 ```
 $ sudo systemctl enable mongod
 ```

# APPサーバ作成
## EC2インスタンス作成
* Amazon Machine Image: Amazon Linux 2 AMI
* Instance Type: t2.small
* ネットワーク: 「VPC作成」で作成したVPC
* サブネット: Private subnet 1
* 自動割り当てパブリックIP: 有効  
__「無効化」を選択してインターネットへのアクセスはNATゲートウェイを経由するのが正しいが、使用している環境ではNATゲートウェイが作れないため、「有効」を選択すること__

* タグ: キー: Name、値: Application Serverの名前（任意の値）
* セキュリティグループ: Application Server用のセキュリティグループ

## これ以降の作業はBastionサーバからAppサーバにSSHでログインして実施する
[Bastionサーバから他のEC2インスタンスにSSHログインする方法](./HowToLoginEC2.md)

## Nodejsインストール
以下のコマンドを実行してnodejsをインストールする

```
$ curl -sL https://rpm.nodesource.com/setup_12.x | sudo bash -
```
```
$ sudo yum install -y nodejs
```

以下のコマンドを実行してバージョンが正しく表示されればインストール成功
```
$ node -v
v12.18.3
```

## yarnインストール
以下のコマンドを実行してyarnをインストールする

```
$ curl -sL https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
```
```
$ sudo yum install -y yarn
```

以下のコマンドを実行してバージョンが正しく表示されればインストール成功
```
$ yarn -v
1.22.4
```

## gitインストール
以下のコマンドを実行してgitをインストールする

```
$ sudo yum install -y git
```

以下のコマンドを実行してバージョンが正しく表示されればインストール成功

```
$ git --version
git version 2.23.3
```

## Growiインストール
1. ソースコード取得  
  以下のコマンドを実行してGithubからGrowiのソースコードを取得する

    ```
    $ sudo mkdir -p /opt
    $ cd /opt
    $ sudo git clone https://github.com/weseek/growi.git
    ```

2. 最新バージョンをチェックアウト  
以下のコマンドを実行して最新バージョンを確認する

    ```
    $ git tag -l
    ```

    以下のコマンドを実行して最新バージョンをチェックアウト
    ```
    $ sudo git checkout -b vバージョン refs/tags/バージョン
    ```

    以下のコマンドを実行してバージョンが正しく表示されればチェックアウト成功

    ```
    git branch
      master
    * バージョン
    ```

3. ビルド  
以下のコマンドを実行してGrowiをビルドする

    ```
    $ cd /opt/growi
    $ sudo yarn
    ```

    __Warningメッセージは無視して良い__

4. 起動確認  
以下のコマンドを実行してGrowiを起動する

    ```
    $ sudo MONGO_URI=mongodb://DatabaseサーバのPrivate IP Address:27017/growi npm start
    ```

    __Warningメッセージは無視して良い__

    以下のメッセージが表示されれば起動成功
    ```
    INFO: growi:crowi/1356 on ip-10-0-30-131.ec2.internal: [production] Express server is listening on port 3000
    ```

    起動が成功したらCTRL + CでGrowiを停止する

## Growi自動起動設定
1. ユニットファイルを作成  
以下のようにユニットファイルを作成する  
__ ※sudoで実行する__

* ファイル  
`/etc/systemd/system/growi.service`

* ファイルの内容
  ```
  [Unit]
  Description=Growi
  After=network.target mongod.service
  
  [Service]
  WorkingDirectory=/opt/growi
  EnvironmentFile=/etc/sysconfig/growi
  ExecStart=/usr/bin/npm start
  
  [Install]
  WantedBy=multi-user.target
  ```

2. コンフィグファイル作成  
以下のようにコンフィグファイルを作成する  
__ ※sudoで実行する__

* ファイル  
`/etc/sysconfig/growi`

* ファイルの内容  
  ```
  NODE_ENV=production
  PASSWORD_SEED="`openssl rand -base64 128 | head -1`"
  MONGO_URI="mongodb://Private IP Address of Database Server:27017/growi"
  FILE_UPLOAD=local
  ```

3. サービス起動と自動起動設定  
以下のコマンドを実行してGrowiの起動と自動起動を設定する

    ```
    $ sudo systemctl daemon-reload
    ```

    ```
    $ sudo systemctl start growi
    ```

    ```
    $ sudo systemctl enable growi
    ```

    以下のコマンドを実行して以下のメッセージが表示されれば起動成功
    ```
    $ sudo systemctl status -l growi
    ```
    ```
    Jul 26 06:17:57 ip-10-0-30-131.ec2.internal npm[1531]: [2020-07-26T06:17:57.945Z]  INFO: growi:crowi/1777 on ip-10-0-30-131.ec2.internal: [production] Express server is listening on port 3000
    ```

# AMI作成
サービスからEC2を選択し、Amazon Machine Imageを作成する

1. 「イメージの作成」を選択
![AMI作成](./img/create_ami1.png)

2. 以下を入力し、「イメージの作成」を押す
* イメージ名: AMIの名前（任意の値）
![AMI作成](./img/create_ami2.png)

# 起動テンプレート作成
サービスからEC2を選択し、起動テンプレートを作成する

1. 「起動テンプレートを作成」を押す

2. 以下を入力し、「起動テンプレートを作成」を押す
* 起動テンプレート名と説明
![AMI作成](./img/create_lc1.png)

* Amazonマシーンイメージ（AMI）  
「AMI作成」で作成したAMIを選択
  ![AMI作成](./img/create_lc2.png)

* インスタンスタイプ  
「t2.small」を選択
  ![AMI作成](./img/create_lc3.png)

* キーペア  
作成済みのキーペアを選択
  ![AMI作成](./img/create_lc4.png)

* セキュリティグループ  
Applicationサーバのセキュリティグループを選択
  ![AMI作成](./img/create_lc5.png)

* リソースタグ  
EC2インスタンスにつけるタグを設定
  ![AMI作成](./img/create_lc6.png)

# ターゲットグループ作成
サービスメニューからEC2を選択し、ターゲットグループを作成する

1. 「ターゲットグループの作成」を押す
![ターゲットグループ作成](./img/create_tg1.png)

2. 以下を入力し、「次へ」を押す
* 基本的な設定
  * ターゲットタイプの選択: 「インスタンス」を選択
  * ターゲットグループ名: ターゲットグループの名前（任意の値）
  * プロトコル: 「HTTP」を選択
  * ポート: 3000
  * VPC: 「VPC作成」で作成したVPCを選択

    ![ターゲットグループ作成](./img/create_tg2.png)

* ヘルスチェック
  * ヘルスチェックプロトコル: 「HTTP」を選択
  * ヘルスチェックパス: 「/」
  * 成功コード: 302
    ![ターゲットグループ作成](./img/create_tg3.png)

3. 「ターゲットグループの作成」を押す  
値を変更せずに「ターゲットグループの作成」を押す
![ターゲットグループ作成](./img/create_tg4.png)

# Load Balancer作成
サービスからEC2を選択し、ロードバランサーを作成する
1. 「ロードバランサーの作成」を押す
![ターゲットグループ作成](./img/create_lb1.png)

2. 「HTTP/HTTPS」の「作成」を押す
![ターゲットグループ作成](./img/create_lb2.png)

3. 以下の値を入力し、「次の手順: セキュリティ設定の構成」を押す
* 名前: ロードバランサーの名前（任意の値）
* スキーム: 「インターネット向け」を選択（デフォルト）
* IPアドレスタイプ: 「ipv4」を選択（デフォルト）
* ロードバランサーのプロトコル: 「HTTP」を選択
* ロードバランサーのポート: 80
* VPC: 「VPC作成」で作成したVPCを選択
* アベイラビリティーゾーン: 表示されているアベイラビリティーゾーンを選択、Public subnet 1,2を選択
![ターゲットグループ作成](./img/create_lb3.png)

4. 「次の手順: セキュリティグループの設定」を押す
![ターゲットグループ作成](./img/create_lb4.png)

5. 以下の値を入力し、「次の手順: ルーティングの設定」を押す
* セキュリティグループの割り当て: 「既存のセキュリティグループを選択する」を選択
* セキュリティグループ: Webサーバ用のセキュリティグループを選択
![ターゲットグループ作成](./img/create_lb5.png)

6. 以下の値を入力し、「次の手順: ターゲットの登録」を押す
* ターゲットグループ: 「既存のターゲットグループ」を選択
* 名前: 「ターゲットグループ作成」で作成したターゲットグループを選択
![ターゲットグループ作成](./img/create_lb6.png)

7. 「次の手順: 確認」を押す
![ターゲットグループ作成](./img/create_lb7.png)

8. 「作成」を押す
![ターゲットグループ作成](./img/create_lb8.png)

# Auto Scalingグループ作成
1. 「Auto Scalingグループの作成」を押す
![ターゲットグループ作成](./img/create_asg1.png)

2. 起動テンプレートを選択  
「起動テンプレート作成」で作成した起動テンプレートを選択し、「次のステップ」を押す
![ターゲットグループ作成](./img/create_asg2.png)

3. Auto Scalingグループ設定
以下を入力し、「次の手順: スケーリングポリシーの設定」を押す
* グループ名: Auto Scalingグループの名前（任意の値）
* ネットワーク: 「VPC作成」で作成したVPCを選択
* サブネット: 「サブネットの作成」で作成したPublic subnet 1,2を選択
* ロードバランシング: 「ひとつまたは複数のロードバランサーからトラフィックを受信する」にチェック
* ターゲットグループ: 「ターゲットグループ作成」で作成したターゲットグループを選択
* ヘルスチェックのタイプ: 「ELB」を選択
![ターゲットグループ作成](./img/create_asg3.png)

4. スケーリングポリシー設定
以下を入力し、「確認」を押す
* メトリクスタイプ: 「CPUの平均使用率」を選択
* ターゲット値: 50
![ターゲットグループ作成](./img/create_asg4.png)

5. 確認、作成
値を確認し、問題がなければ「Auto Scalingグループの作成」を押す
![ターゲットグループ作成](./img/create_asg5.png)

# 動作確認
## Growi
WebブラウザからロードバランサーのDNS名でアクセスしてGrowiの画面が表示されることを確認する
* ロードバランサーのDNS名  
以下の画面で確認する
![ターゲットグループ作成](./img/lb_dns.png)

## Auto Scalingグループ
1. BastionサーバからApplicationサーバのインスタンスにSSHログインする

2. Growiを停止する  
    ```
    $ sudo systemctl stop growi
    ```

3. インスタンスが入れ替わることを確認
