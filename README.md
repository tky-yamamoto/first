# Truck

## 目次
- [システム概要](#システム概要)
- [アクセス情報(URL,アカウント,CloudFrontのドメイン)](#アクセス情報urlアカウントcloudfrontのドメイン)
- [初期設定手順](#初期設定手順)
  - [リソースの取得](#リソースの取得)
  - [設定ファイルの反映](#設定ファイルの反映)
  - [インストール(truckadmin,truckedidevユーザーは不要)](#インストールtruckadmintruckedidevユーザーは不要)
  - [初回ビルドの実施(truckadmin,truckedidevユーザーは不要)  ](#初回ビルドの実施truckadmintruckedidevユーザーは不要)
- [デプロイ手順](#デプロイ手順)
  - [本番環境](#本番環境)
    - [公開コンテンツを更新する場合](#公開コンテンツを更新する場合)
    - [サーバーライブラリーの更新](#サーバーライブラリーの更新)
    - [データベースの更新](#データベースの更新)
    - [`crontab`の更新](#crontabの更新)
  - [ステージ環境](#ステージ環境)
    - [_gateway_, _preview_, _admin_ のビルド・再起動](#_gateway_-_preview_-_admin_-のビルド・再起動)
    - [_jr-server_ のビルド・再起動](#_jr-server_-のビルド・再起動)
    - [クライアント(管理)のビルド(複数台構成の場合は全台同時実行)](#クライアント管理のビルド複数台構成の場合は全台同時実行)
    - [クライアント(下見)のビルド(複数台構成の場合は全台同時実行)](#クライアント下見のビルド複数台構成の場合は全台同時実行)
    - [ステージング環境(_botannabe_)から _S3_ への静的ファイルアップロード](#ステージング環境_botannabe_から-_s3_-への静的ファイルアップロード)
    - [`crontab`の更新](#crontabの更新-1)
    - [_gateway_, _preview_, _admin_, _jr-server_ の一括再起動  ](#_gateway_-_preview_-_admin_-_jr-server_-の一括再起動)


## システム概要
株式会社オークネット（トラック推進室）様向けのトラック共有在庫・下見／管理システムである。


## アクセス情報(URL,アカウント,CloudFrontのドメイン)
それぞれ以下サーバー内の _truckdev_ ユーザー ホームディレクトリー直下の _site-info.txt_ を参照のこと

* ステージ環境： _botannabe_
* 本番環境： _genpeinabe_


## 初期設定手順
EC2インスタンスにミドルウェアインストール後、サービス提供出来るまでのコマンド手順を以下に記す。  
尚、注意書きが無い限りは全ユーザーで実施するものとする。

**前提**    
以下ファイル,ディレクトリーが既に配備済みであること
* _truckdev_ ユーザー
    * ~/.aws/config
    * ~/.aws/credentials
    * ~/.cloudfront.info
    * ~/.rabbitmq.info
* _truckadmin_ ユーザー
    * ~/.aws/config
    * /var/lib/redis (参照権限)
    * /opt/manage-aws
* _truckedidev_ ユーザー  
    前提条件なし


### リソースの取得

    _GitHub_ より`clone`実施
    ```
    $ git clone git@github.com:a-ibs/truck.git ~/truck-apps
    ```

### 設定ファイルの反映

    _alias_ 反映
    ```bash
    $ ln ~/truck-apps/os-settings/alias ~/.alias \
        --symbolic \
        && source ~/.bashrc
    ```

    _bash-profile_ 反映
    ```bash
    $ ln ~/truck-apps/os-settings/bash_profile ~/.bash_profile \
        --force \
        --symbolic \
        && source ~/.bash_profile
    ```

###  インストール(truckadmin,truckedidevユーザーは不要)

    `nvm`と _Node.js_ をインストール
    ```bash
    $ curl https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh \
            --output - \
            | bash \
        && source ~/.bashrc \
        && nvm install v6.9.1
    ```

    _Node.js_ パッケージのインストール
    ```bash
    $ npm install pm2@2.0.18 --global \
        && npm install coffee-script@1.11.1 --global \
        && npm install grunt-cli@v1.2.0 --global
    ```

    _CSS_ 生成フレームワークのインストール
    ```bash
    $ gem install sass compass \
        --install-dir ~/local
    ```

    帳票サービスモジュールコンパイル用 _Maven_ ライブラリーのダウンロード
    ```bash
    $ mkdir ~/.apache-maven-3.2.1 \
        && wget https://archive.apache.org/dist/maven/binaries/apache-maven-3.2.1-bin.tar.gz \
            --output-document=- \
        | tar --extract \
            --gzip \
            --directory ~/.apache-maven-3.2.1 \
            --strip-components 1
    ```

### 初回ビルドの実施(truckadmin,truckedidevユーザーは不要)  

    初回用 _Grunt_ タスクの実行
    ```bash
    $ cd ~/truck-apps/ \
        && ./setup.sh \
        && grunt deploy
    ```

    公開コンテンツビルド用のパッケージインストール
    ```bash
    $ cd ~/truck-apps/client/admin \
        && npm install \
        && cd ~/truck-apps/client/preview \
        && npm install
    ```

    _S3_ への`sync`用ディレクトリー作成
    ```bash
    $ mkdir ~/truck-apps/client/{admin,preview}/sync_dir/static \
        --parents
    ```

    帳票サービスのコンパイル/パッケージ化に使用する _Maven_ のインストール
    ```bash
    $ cd ~/truck-apps/jr-server \
        && mvn install
    ```

## デプロイ手順
更新対象に合わせて手順を分けて以下に記す。

### 本番環境

対象ホスト
* _genpeinabe_
* _tounyuunabe_
* _motsunabe_

**本番環境デプロイに関する補足**  
現状では3ホストそれぞれのローカルにindex.htmlを含めたファイルが配備されており、個別にデプロイを実施する設計となっている。
_HTML_ ファイル内に記述された _JavaScript_ や _CSS_ へのパスはデプロイ時の年月日を持ってユニークにしているが、これらがサーバーごとに異なる年月日とならないよう留意すること。
また、コマンドを実行するユーザーと対象サーバーについては、それぞれの括弧内参照。


#### 公開コンテンツを更新する場合

    サービスの停止(truckdev@全サーバー)
    ```bash
    $ pm2 stop all \
        && cd ~/truck-apps/jr-server \
        && ./jrsrv-ctrl.sh stop
    ```

    登録済みジョブの無効化(truckdev@全サーバー,truckadmin@genpeinabe,truckedidev@tounyuunabe)
    ```
    $ crontab -r
    ```

    ソース最新化(truckdev@全サーバー)
    ```bash
    $ cd ~/truck-apps \
        && git pull
    ```

    管理サイト クライアントファイルの退避(truckdev@全サーバー ※初期設定時不要)
    ```bash
    $ cp ~/truck-apps/client/admin/release{,.$(date '+%Y%m%d')} \
          --preserve \
          --recursive \
        && ln ~/truck-apps/{client/admin/release.$(date '+%Y%m%d'),gateway-server/public/admin} \
          --symbolic \
          --force \
          --no-dereference
    ```

    公開コンテンツのビルド(truckdev@全サーバー ※要同時実行)
    ```bash
    $ cd ~/truck-apps/client/admin \
        && grunt \
        && cd ~/truck-apps/client/preview \
        && grunt
    ```

    ビルド時のファイル名一致確認(truckdev@全サーバー ※一致していない場合はビルドからやり直す)
    ```bash
    $ ls ~/truck-apps/client/preview/release/js/index*
    ```

    1世代前のreleaseディレクトリーを除き削除(truckdev@全サーバー ※初期設定時不要)
    ```bash
    $ ls ~/truck-apps/client/admin/release.* \
            --directory \
         | sort --reverse \
         | tail --lines=+3 \
         | xargs rm \
            --recursive \
            --force
    ```

    ```bash
    $ ls ~/truck-apps/client/preview/release.* \
            --directory \
         | sort --reverse \
         | tail --lines=+3 \
         | xargs rm \
            --recursive \
            --force
    ```

    静的ファイルのアップロード(truckdev@genpeinabe)
    ```bash
    $ rsync --archive \
          --delete \
          ~/truck-apps/client/preview/release/{js,css,images,lib} \
          ~/truck-apps/client/preview/sync_dir/static \
          && aws s3 sync \
              ~/truck-apps/client/preview/sync_dir/static \
              s3://truck-production/static
    ```

    _CloudFront_ へのアップロードしたコンテンツ `sync` 確認(truckdev@genpeinabe)  
    ※レスポンスコード200が応答されればOK
    ```
    $ watch --interval=5 \
        "curl https://d2vpuq2hegsmy2.cloudfront.net/static/js/index.$(date '+%Y%m%d').min.js \
            --write-out '%{http_code}\n' \
            --silent \
            --output /dev/null"
    ```

    下見サイト ドキュメントルート差し替え(truckdev@全サーバー ※要同時実行)
    ```bash
    $ cp ~/truck-apps/client/preview/release{,.$(date '+%Y%m%d')} \
          --preserve \
          --recursive \
        && ln ~/truck-apps/{client/preview/release.$(date '+%Y%m%d'),gateway-server/public} \
        --symbolic \
        --force \
        --no-dereference
    ```

    ジョブの再登録(truckdev@全サーバー,truckadmin@genpeinabe,truckedidev@tounyuunabe)
    ```bash
    $ crontab ~/truck-apps/os-settings/crontab/$(whoami)@${HOSTNAME%.auctruck.local}
    ```

#### サーバーライブラリーの更新

    ELBから _genpeinabe_ と _tounyuunabe_ の切り離し(truckadmin@genpeinabe)
    ※truckadmin@tounyuunabeでも可
    ```bash
    $ ~/truck-apps/tools/manage-aws/deregister.exs genpeinabe tounyuunabe
    ```

    _genpeinabe_ , _tounyuunabe_ のサービス停止(truckdev@genpeinabe,truckdev@tounyuunabe)
    ```
    $ pm2 stop all
    ```

    _genpeinabe_ , _tounyuunabe_ のデプロイ(truckdev@genpeinabe,truckdev@tounyuunabe)
     ```bash
    $ cd ~/truck-apps \
        && grunt deploy
    ```

    _genpeinabe_ , _tounyuunabe_ のサービス起動(truckdev@genpeinabe,truckdev@tounyuunabe)
    ```
    $ pm2 start all
    ```

    _genpeinabe_ , _tounyuunabe_ の帳票用サービス停止(truckdev@genpeinabe,truckdev@tounyuunabe)
    ```bash
    $ cd ~/truck-apps/jr-server \
        && ./jrsrv-ctrl.sh stop
    ```

    _genpeinabe_ , _tounyuunabe_ の帳票用サービス コンパイル/パッケージ化(truckdev@genpeinabe,truckdev@tounyuunabe)
    ```bash
    $ mvn compile \
        && mvn package
    ```

    _genpeinabe_ , _tounyuunabe_ の帳票用サービス起動(truckdev@genpeinabe,truckdev@tounyuunabe)
    ```
    $ cd ~/truck-apps/jr-server \
        && ./jrsrv-ctrl.sh start
    ```

    ELBに _genpeinabe_ と _tounyuunabe_ の復帰(truckadmin@genpeinabe)
    ※truckadmin@tounyuunabeでも可
    ```bash
    $ ~/truck-apps/tools/manage-aws/register.exs genpeinabe tounyuunabe
    ```

    ELBから _motsunabe_ の切り離し(truckadmin@motsuabe)
    ```bash
    $ ~/truck-apps/tools/manage-aws/deregister.exs motsunabe
    ```

    _motsunabe_ のサービス停止(truckdev@motsunabe)
    ```
    $ pm2 stop all
    ```

    _motsunabe_ のデプロイ(truckdev@motsunabe)
    ```bash
    $ cd ~/truck-apps \
        && grunt deploy
    ```

    _motsunabe_ のサービス起動(truckdev@motsunabe)
    ```
    $ pm2 start all
    ```

    _motsunabe_ の帳票用サービス停止(truckdev@motsunabe)
    ```
    $ cd ~/truck-apps/jr-server \
        && ./jrsrv-ctrl.sh stop
    ```

    _motsunabe_ の帳票用サービス コンパイル/パッケージ化(truckdev@motsunabe)
    ```bash
    $ mvn compile \
        && mvn package
    ```

    _motsunabe_ の帳票用サービス起動(truckdev@motsunabe)
    ```
    $ cd ~/truck-apps/jr-server \
        && ./jrsrv-ctrl.sh start
    ```

    _ELB_ に _motsunabe_ を復帰(truckadmin@motsunabe)
    ```bash
    $ ~/truck-apps/tools/manage-aws/register.exs motsunabe
    ```

#### データベースの更新

    _RDS_ へファンクションの更新(truckdev@genpeinabe)
    ```bash
    $ psql-aws < ~/truck-apps/data/function/fnc_current_meter_check.sql
    ```

    ローカル _RDS_ へファンクションの更新(truckdev@全サーバー ※要同時実行)
    ```bash
    $ psql-local < ~/truck-apps/data/function/fnc_current_meter_check.sql
    ```

#### `crontab`の更新

    アイオーク連携の実行ファイルに実行権限付与(truckdev@tounyuunabe)
    ```bash
    $ chmod +x ~/truck-apps/renkei/renkei-server/bin/iauc.sh
    ```

    設定の最新化(truckdev@全サーバー,truckadmin@genpeinabe,truckedidev@tounyuunabe)
    ```bash
    $ ln ~/truck-apps/os-settings/crontab/.crontab.production.inc ~/.crontab.inc \
          --symbolic \
          --force \
      && crontab ~/truck-apps/os-settings/crontab/$(whoami)@${HOSTNAME%.auctruck.local}
    ```

---

### ステージ環境

対象ホスト
* _botannabe_
* _kamonabe_ (現在停止中)
* _shamonabe_ (現在停止中)

ステージ環境ではデプロイ対象によって以下のいずれかを使用する

#### _gateway_, _preview_, _admin_ のビルド・再起動
    ```bash
    $ cd ~/truck-apps \
        && grunt deploy \
        && pm2 restart all
    ```

#### _jr-server_ のビルド・再起動
    ```bash
    $ cd ~/truck-apps/jr-server \
        && ./jrsrv-ctrl.sh stop \
        && mvn compile \
        && mvn package \
        && ./jrsrv-ctrl.sh start
    ```

#### クライアント(管理)のビルド(複数台構成の場合は全台同時実行)
    ```bash
    $ cd ~/truck-apps/client/admin && grunt
    ```

#### クライアント(下見)のビルド(複数台構成の場合は全台同時実行)
    ```bash
    $ cd ~/truck-apps/client/preview && grunt
    ```

#### ステージング環境(_botannabe_)から _S3_ への静的ファイルアップロード
    ```bash
    $ rsync --archive \
          --delete \
          ~/truck-apps/client/preview/release/{js,css,images,lib} \
          ~/truck-apps/client/preview/sync_dir/static \
          && aws s3 sync \
              ~/truck-apps/client/preview/sync_dir/static \
              s3://truck-stage/static
    ```

#### `crontab`の更新
    ```bash
    $ ln ~/truck-apps/os-settings/crontab/.crontab.stage.inc ~/.crontab.inc \
          --symbolic \
          --force \
      && crontab ~/truck-apps/os-settings/crontab/$(whoami)@${HOSTNAME%.auctruck.local}
    ```

#### _gateway_, _preview_, _admin_, _jr-server_ の一括再起動  
    ```bash
    $ ~/truck-apps/tools/.startall
    ```
