+++
author = "Masataka Kashiwagi"
title = "スプレッドシートからBigQueryへDigdagを使ったデータ連携"
description = "teamayaプロジェクトのデータ連携をした話"
date = 2021-12-22T18:00:00+09:00
draft = false
share = true
showLicense = false
tags = ["Dev", "Data"]
+++

## はじめに
今回は，teamayaという個人プロジェクトで進めている<span class="marker_yellow">**データ連携**</span>の話になります．コードは以下のリポジトリに置いてあるので，ご自由に使用下さい！

<iframe class="hatenablogcard" style="width:100%;height:155px;margin:15px 0;max-width:680px;" title="masatakashiwagi/teamaya: This is a repository for Data integration and Machine Learning Pipeline." src="https://hatenablog-parts.com/embed?url=https://github.com/masatakashiwagi/teamaya" frameborder="0" scrolling="no"></iframe>

具体的には，手元にあるスプレッドシートのデータをBigQueryの特定のテーブルに連携するまでの話になります．

普段家計簿のデータをスプレッドシートに手入力で管理してるんですが，そのデータをBigQueryに集めて色々と検証できると良いなーという思いから，データ連携を始めました．加えて，可視化も良くしたい思いもあり，Data Studioでダッシュボードを作ったりもしています．

データ連携をするだけであれば，Embulkを単体実行するのでこと足りますが，今回はサーバーモードで立ち上げたDigdag UIを使ってワークフローを実行しています．スケジュール実行，ワークフロー管理や履歴管理などがUIからだとしやすく，使い心地などを知るためにも使用しました．

また，Docker環境で実行できるように構成しています．Docker化することで，簡単に別環境に持っていくことができますし，スクラップ&ビルドがしやすいのもあります．

今回は以下の2種類の方法でデータを連携する方法を紹介します．内容的には既に技術記事に書かれているものが多いと思いますが，今回はDockerコンテナで各タスクが実行できるようにしているので，その辺りを参考頂けたらと思います．

1. <span class="marker_yellow">**Dockerコンテナ内からEmbulkを直接実行して，データを転送する方法**</span>
2. <span class="marker_yellow">**Digdag UIからワークフローを実行して，データを転送する方法**</span>
    - こちらも裏側では，Embulkが実行されます．

今回のデータ連携フローのアーキテクチャーは以下のような感じです．

![データ連携フローのアーキテクチャー](../../img/teamaya-data-workflow.png "データ連携フローのアーキテクチャー")

アンドリュー・カーネギーの以下の名言にもあるように，機械学習エコシステムを自分で作っていくために，一歩一歩進めています！
> **最も高い目標を達成するには、一歩一歩進むしかないという事実を、頭に入れておかなければならない。**

## BigQueryとスプレッドシートの設定
GCPのアカウント登録方法は割愛しますが，gmailがあれば簡単に登録できます．登録が完了したら，適当な[プロジェクトを作成](https://cloud.google.com/resource-manager/docs/creating-managing-projects)して下さい．

### Google Sheets APIの有効化を行う

「APIとサービス」 → 「ライブラリ」と画面遷移し，検索窓に「**Google Sheets API**」と入力して検索すると，スプレッドシートのAPIを有効化できる画面に遷移するので，有効化を行います．ここで有効化しておかないと，この後スプレッドシートを使用したデータ連携が出来ないので注意下さい！

![Google Sheets API](../../img/teamaya-google-sheets-api.png "Google Sheets API")

### サービスアカウントの作成

それが終わったら，サービスアカウントを作成します．「IAMと管理」 → 「サービスアカウント」へアクセスした後，必要な情報を入力し，キーの作成からJSONを選択してキーの作成を行います．そうすると，サービスアカウントのJSONファイルがダウンロードされるので，これを`~/.gcp`配下に置いておきます．

### サービスアカウントのメールアドレスをスプレッドシートに登録

データ連携したいスプレッドシートを開き，右上の共有ボタンから「ユーザーやグループを追加」の枠にダウンロードしたサービスアカウントのメールアドレスをコピー&ペーストして，送信をクリックします．そうすることで，このスプレッドシートのデータを登録したサービスアカウントで転送することができます．

ここで，メールアドレスの許可をしていない場合，Embulk実行時に以下のエラーが発生します．
> Error: (ClientError) forbidden: The caller does not have permission

### BigQueryにデータセットを作成

データを格納するために，事前にデータセットを作成しておく必要があるので，データセットIDを適当に決めて，データセットの作成を行っておきます．

## 1. Embulkを直接実行してデータ転送を行う場合
この方法は，Dockerコンテナ内からEmbulkを直接実行して，スプレッドシートのデータをBigQueryのテーブルに転送する方法になります．

[Embulk](https://www.embulk.org/)の細かい説明は割愛しますが，簡単に言えば，バルクデータローダーの役割としてBigQueryなどのデータレイク/データウェアハウスにデータ転送を行うことができます．

データ転送を行うために用意するものとしては，以下になります．

1. <span class="marker_yellow">**Gemfile**</span>
2. <span class="marker_yellow">**Liquidファイル**</span>
3. <span class="marker_yellow">**Dockerfile & docker-compose.ymlファイル**</span>

### Gemfileを用意する
EmbulkのプラグインをGemfile/Gemfile.lockでバージョン管理するために用意します．Embulkには，データのInput/Outputのプラグインがあリ，これを使うことで様々なデータソースからターゲットにデータを連携することができます．

- Input: [embulk-input-google_spreadsheets](https://github.com/medjed/embulk-input-google_spreadsheets)
- Output: [embulk-output-bigquery](https://github.com/embulk/embulk-output-bigquery)

```ruby
source 'https://rubygems.org/'

# No versions are specified for 'embulk' to use the gem embedded in embulk.jar.
# Note that prerelease versions (e.g. "0.9.0.beta") do not match the statement.
# Specify the exact prerelease version (like '= 0.9.0.beta') for prereleases.
gem 'embulk'

# input spreadsheets plugin
gem 'embulk-input-google_spreadsheets'
# ouput bigquery plugin
gem 'embulk-output-bigquery'
gem 'tzinfo-data'
```

### Liquidファイルを作成する
Embulkの設定ファイルとしてLiquidファイルを作成します．YAMLファイルに設定を記述することもできますが，以下のメリットでLiquidファイルを使用しています．

- 変数を設定することができる
- 同じ設定内容を共通ファイルとして使うことができる
- etc...

BigQueryとスプレッドシートの情報は，`.env`ファイルを作成して，環境変数として管理しています．これらの変数をLiquidファイルで使用しています．

```yaml
in:
  type: google_spreadsheets
  auth_method: service_account
  {% comment %} GCPのサービスアカウントのJSONファイルパス {% endcomment %}
  json_keyfile: {{ env.GCP_SERVICE_JSON }}
  {% comment %} スプレッドシートのURL {% endcomment %}
  spreadsheets_url: {{ env.SPREADSHEETS_TABLE }}
  default_timezone: 'Asia/Tokyo'
  {% comment %} スプレッドシートのワークシートタイトル {% endcomment %}
  worksheet_title: year_purchase_amount_2019
  {% comment %} headerを指定している場合は2行目からとなる {% endcomment %}
  start_row: 2
  {% comment %} カラム名と型を指定する {% endcomment %}
  columns:
    - {name: id, type: long}
    - {name: date, type: timestamp, format: '%Y/%m/%d', timezone: 'Asia/Tokyo'}
    - {name: category, type: string}
    - {name: purchaser, type: string}
    - {name: purchase_amount, type: long}
    - {name: memo, type: string}

out:
  type: bigquery
  mode: replace
  auth_method: service_account
  json_keyfile: {{ env.GCP_SERVICE_JSON }}
  {% comment %} BigQueryのプロジェクト名 {% endcomment %}
  project: {{ env.BIGQUERY_PROJECT }}
  {% comment %} BigQueryのデータセット名 {% endcomment %}
  dataset: {{ env.BIGQUERY_PURCHASE_AMOUNT_DATASET }}
  {% comment %} BigQueryのテーブル名 {% endcomment %}
  table: daily_purchase_amount
  auto_create_table: true
  source_format: NEWLINE_DELIMITED_JSON
  default_timezone: 'Asia/Tokyo'
  default_timestamp_format: '%Y-%m-%d'
  formatter: {type: jsonl}
  encoders:
    - {type: gzip}
  retries: 3
```

envファイルの説明も補足でしておきます．適宜設定している環境に合わせて修正します．

```bash
SPREADSHEETS_TABLE=<該当するスプレッドシートのURL: https://docs.google.com/spreadsheets/d/hogehoge>
BIGQUERY_PROJECT=<BigQueryのプロジェクト名>
BIGQUERY_PURCHASE_AMOUNT_DATASET=<BigQueryのデータセット名>
GCP_SERVICE_JSON=/root/.gcp/hoge.json
```

### Dockerfile & docker-compose.ymlファイルを作成する
今回はdocker環境から実行するので，Dockerfileとdocker-compose.ymlファイルを作成します．（後ほど使うDigdagの内容も記載されています）

```dockerfile
FROM openjdk:8-alpine
LABEL MAINTAINER=masatakashiwagi

ENV DIGDAG_VERSION="0.9.42"
ENV EMBULK_VERSION="0.9.23"

RUN apk --update add --virtual build-dependencies \
    curl \
    tzdata \
    coreutils \
    bash \
    && curl --create-dirs -o /bin/digdag -L "https://dl.digdag.io/digdag-${DIGDAG_VERSION}" \
    && curl --create-dirs -o /bin/embulk -L "https://dl.embulk.org/embulk-$EMBULK_VERSION.jar" \
    && chmod +x /bin/digdag \
    && chmod +x /bin/embulk \
    && cp /usr/share/zoneinfo/Asia/Tokyo /etc/localtime \
    && apk del build-dependencies --purge

ENV PATH="$PATH:/bin"

# Install libc6-compat for Embulk Plugins to use JNI
# cf: https://github.com/jruby/jruby/wiki/JRuby-on-Alpine-Linux
# https://github.com/classmethod/docker-embulk
RUN apk --update add libc6-compat

# Copy Embulk configuration
COPY ./embulk/task /opt/workflow/embulk/task

# Make bundle
WORKDIR /opt/workflow/embulk
RUN embulk mkbundle bundle

# Copy Gemfile file
# This is the workaround, because jruby directory is not created
COPY ./embulk/bundle/Gemfile /opt/workflow/embulk/bundle
COPY ./embulk/bundle/Gemfile.lock /opt/workflow/embulk/bundle
WORKDIR /opt/workflow/embulk/bundle

# Install Embulk Plugins
RUN embulk bundle

# Set up Digdag Server
COPY ./digdag /opt/workflow/digdag
# ADD https://github.com/ufoscout/docker-compose-wait/releases/download/2.9.0/wait /bin/wait
# RUN chmod +x /bin/wait
WORKDIR /opt/workflow

CMD ["tail", "-f", "/dev/null"]
```

bundle辺りでまわりくどいやり方をしていますが，`RUN embulk bundle`を実行した際に上手くインストールできなかったため，ワークアラウンドとして，`mkbundle`した後にGemfile/Gemfile.lockをdockerコンテナ内に`COPY`した上で，`RUN embulk bundle`を行っています．

### Embulkをdockerコンテナで実行する方法
では，実際にEmbulkの実行を行うために，まずはEmbulkのコンテナサービスを立ち上げます．

```bash
# コンテナの立ち上げ
docker compose up -d embulk
```

バックグラウンドでコンテナを起動しておきます．その後，dry-runを行うためにpreviewコマンドを実行します．

```bash
# dry-run
docker exec embulk sh /bin/embulk preview -b embulk/bundle embulk/task/spreadsheet/export_hab_purchase_amount.yml.liquid
```

dry-run実行後の結果を載せておきます．

![Embulk dry-run](../../img/teamaya-exec-img1.png "Embulk dry-run")

dry-runが大丈夫だった場合，本番実行を行います．本番はrunコマンドを実行します．

```bash
# production-run
docker exec embulk sh /bin/embulk run -b embulk/bundle embulk/task/spreadsheet/export_hab_purchase_amount.yml.liquid
```

本番実行が上手くいくとBigQueryにデータが入っていることを確認できます．

![Embulk production-run](../../img/teamaya-bigquery-img1.png "Embulk production-run")

## 2. Digdag UIからワークフローを実行してデータ転送を行う場合
次に説明するこちらの方法は，digdag serverを立ち上げて，UI上からワークフローを実行して，スプレッドシートのデータをBigQueryのテーブルに転送する方法になります．（裏でEmbulkが動いています）

[Digdag](https://www.digdag.io/)の細かい説明は割愛しますが，簡単に言えば，設定ファイルでバッチのワークフロー実行を定義・管理できるワークフローエンジンになります．スケジュール実行や失敗時の通知などを行うことができます．

Digdagのワークフローからデータ転送を行うために用意するものとしては，以下になります．

1. <span class="marker_yellow">**digファイル**</span>
2. <span class="marker_yellow">**serverのpropertiesファイル**</span>
3. <span class="marker_yellow">**docker-compose.ymlファイル**</span>

上記に加えて，1で説明したEmbulk実行に必要なファイルも用意します．

### digファイルを用意する
個別のワークフローを単発実行する場合は，`digdag run hoge.dig`で良いのですが，今回はUIから実行したいので，厳密にはdigファイルの中身が要ります．記述する内容としては，ワークフローで実行していくタスクをコードに落としていきます．

Digdagでは，エラーの場合や成功した場合に，どういった処理をするのかを書くことができるので，例えば，それぞれの場合でslackに通知を行えたりもできます（今回はslack通知は実装できていませんが，今後実装していきたいです！）．リトライ回数の設定やスケジュール実行の設定もここでできます．

```bash
+task:
    _retry: 1
    sh>: /bin/embulk run -b /opt/workflow/embulk/bundle /opt/workflow/embulk/task/spreadsheet/export_hab_purchase_amount.yml.liquid
    _error:
        echo>: workflow error...
+success:
    echo>: workflow success!
```

今回は以下のワークフローになっています．
- task:
    - リトライ回数: 1回
    - Embulkの実行
    - エラーの場合は`workflow error...`をechoする
- success:
    - `workflow success!`をechoする

successはtaskが正常に終了した場合に，実行されることになります．

### serverのpropertiesファイルを用意する
サーバーモードで起動するために，引数にオプションを指定する必要があるのですが，それらの設定を`server.properties`ファイルに集約しています．設定内容は色々とあるので詳しくは[公式ドキュメント: server-mode-commands](https://docs.digdag.io/command_reference.html#server-mode-commands)を参照下さい．

このファイルには，サーバーの情報やデータベースの情報を記載しています．

```bash
server.bind = 0.0.0.0
server.port = 65432
server.admin.bind = 0.0.0.0
server.admin.port = 65433
server.access-log.pattern = json
server.access-log.path = /var/log/digdag/access_logs
log-server.type = local

# database情報
# database.type = memory
database.type = postgresql
database.user = digdag
database.password = digdag
database.host = postgres
database.port = 5432
database.database = digdag
database.maximumPoolSize = 32
```

サーバーモードで起動すると，ワークフローの情報を保存するために，データベースの設定が必要になってきます．今回はPostgreSQLを別コンテナで立てて，Digdagコンテナと接続することにしています．ちなみに，これらの情報をインメモリで保存することもできます（この場合は，`database.type = memory`として下さい）．

### docker-compose.ymlファイルを作成する
色々と試して上手くいかなかったのですが，最終的には以下の内容で落ち着きました．service共通の定義は`x-template`にまとめています．serviceは3つありますが，Digdagを使う場合はdigdagとpostgresのみを立ち上げて使います．

dockerの`depends_on`は依存関係（起動順序）を指定できますが，DB起動後のアプリ起動までを制御できるわけではないので，postgresの起動完了前にdigdagがアクセスしてしまい起動失敗する事があります．
- [Control startup and shutdown order in Compose](https://docs.docker.com/compose/startup-order/)

このため，`condition: service_started`と設定することで，postgresが起動後にdigdagが立ち上がるようにしています．

```yaml
depends_on:
  postgres:
    condition: service_started
```

また，`tty`をtrueに設定しているのは，コンテナが正常終了して止まらないようにするためになります．

```yaml
version: '3.8'

# Common definition
x-template: &template
  volumes:
      - ~/.gcp:/root/.gcp:cached
      - /tmp:/tmp
  env_file:
      - .env

services:
  digdag:
    container_name: digdag
    build: .
    tty: true
    ports:
      - 65432:65432
      - 65433:65433
    volumes:
        - /var/run/docker.sock:/var/run/docker.sock
    command: ["java", "-jar", "/bin/digdag", "server", "-c", "digdag/server.properties", "--log", "/var/log/digdag/digdag_server.log", "--task-log", "/var/log/digdag/task_logs"]
    depends_on:
      postgres:
        condition: service_started
    <<: *template

  postgres:
    image: postgres:13.1-alpine
    container_name: postgres
    ports:
      - 5432:5432
    environment:
      POSTGRES_DB: digdag
      POSTGRES_USER: digdag
      POSTGRES_PASSWORD: digdag
    volumes:
      - /tmp/data:/var/lib/postgresql/data
    tty: true
    <<: *template

  embulk:
    container_name: embulk
    build: .
    <<: *template

networks:
  default:
    external:
      name: teamaya
```

### Digdagをdockerコンテナで実行する方法
では，Digdag UIを使うためにDigdagをサーバーモードで起動します．

```bash
# コンテナの立ち上げ
docker compose up -d digdag
```

バックグラウンドでコンテナを起動しておきます．`docker ps`コマンドでコンテナが立ち上がっていることを確認したら，`http://localhost:65432/`でUIにアクセスします．以下の画面が表示されたらOKです．

![Digdag UI](../../img/teamaya-digdag-ui-img1.png "Digdag UI")

New projectからNameを設定したら，Add fileをクリックして，digファイルのコードをコピー&ペーストします．貼り付けたら，Saveで内容を保存します．そしたら，Workflowsのタブを選択し，先ほど追加したワークフローが表示されているのでクリックします．右上のRunボタンを押して実行が完了すると，下図のような結果になります．

![Digdag UI](../../img/teamaya-digdag-ui-img2.png "Digdag UI")

StatusがSuccessになっていれば，正常終了でBigQueryにデータが入っていると思うので，確認してみて下さい．

## おわりに
今回は，手元にあるスプレッドシートのデータをDigdag/Embulkを用いてBigQueryに連携するを紹介しました．

やっぱり，UIで直感的に状況や情報がわかるのはメリットだなと思います．複数のワークフローが動作したりする環境だとそれらも管理できるので良いと思います．スケジュール実行やエラーや成功時の通知設定なども出来るので活用したいと思います．

また今回は，データ連携にDigdagを用いましたが，他にもApache AirflowやそのマネージドサービスであるCloud Composerを使っても同等のことができると思います．この辺りも別途試していきたいと思います．

個人的な次のステップとしては，Dataformやdbtを使った「<span class="marker_yellow">**データの品質管理**</span>」に興味があるので，データをTransformしたり，テストしたりして考えていきたいです．また，データレイク/データウェアハウス/データマートの設計などを併せて学習していきたいと思います！

これとは別軸で機械学習パイプラインの設計もしていこうかなと考えています！

## 参考
- [Digdag 入門](https://recruit.gmo.jp/engineer/jisedai/blog/introduction-to-digdag/)
- [EmbulkとDigdagとデータ分析基盤と](https://www.slideshare.net/ToruTakahashi4/embulkdigdag)
- [digdagをDockerizeしてECS上で運用することにしました - 雑なメモ](https://yukiyan.hatenablog.jp/entry/2017/01/23/102411)
- [Docker Compose の depends_on の使い方まとめ](https://gotohayato.com/content/533/)
- [DockerのTTYって何?](https://zenn.dev/hohner/articles/43a0da20181d34)