+++
author = "Masataka Kashiwagi"
title = "AirbyteでスプレッドシートのデータをBigQueryに連携"
description = "OSSのELツールであるAirbyteでデータ連携"
date = 2022-09-29T15:37:53+09:00
draft = false
share = true
showLicense = false
support = true
tags = ["Dev", "Data"]
+++

## はじめに
以前から気になっていたOSSの <span class="marker_yellow">**[Airbyte](https://airbyte.com/)**</span> というELに特化したData Integrationツールを使ってみたかったので，今回はこれを使って以前Embulkで実装していたスプレッドシートからBigQueryへのデータ同期処理と同じことができるか試してみた話になります．

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" title="スプレッドシートからBigQueryへDigdagを使ったデータ連携" src="https://hatenablog-parts.com/embed?url=https://masatakashiwagi.github.io/portfolio/post/integrate-spreadsheets-to-bigquery-with-digdag/" width="300" height="150" frameborder="0" scrolling="no"></iframe>

Airbyteは良い感じのUIがあるので，UIをポチポチしながら設定していきます．

## Airbyteとは？
![Airbyte](../../img/airbyte-img1.png "Airbyte")

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" title="airbytehq/airbyte" src="https://hatenablog-parts.com/embed?url=https://github.com/airbytehq/airbyte" width="300" height="150" frameborder="0" scrolling="no"></iframe>

AirbyteはOSSのETLツールですが，特にExtractとLoadに注力しているツールになります．豊富なデータソース（[Source](https://airbyte.gitbook.io/airbyte/integrations/sources)）とターゲットソース（[Destination](https://airbyte.gitbook.io/airbyte/integrations/destinations)）に対応していて，これらを設定することでデータを簡単に連携することができます．Transform部分は内部的にはdbtを使ってハンドリングしているみたいです．（[Transformations with SQL (Part 1/3)](https://airbyte.gitbook.io/airbyte/operator-guides/transformation-and-normalization/transformations-with-sql)）

提供形態としては，OSSとマネージドサービス（クラウド版: 有料）があり，ローカルをはじめ，AWS/GCP/Azureと各種クラウドサービスでデプロイすることができます（参考: [Deploying Airbyte Open Source](https://airbyte.gitbook.io/airbyte/deploying-airbyte)）．

Connectorは既に用意されているもの（[airbyte/airbyte-integrations/connectors/](https://github.com/airbytehq/airbyte/tree/master/airbyte-integrations/connectors)）もあれば，独自で作成することもできます．

## スプレッドシートからBigQueryに連携
基本的には，Tutorialに沿って進めていきます．まずは`git clone`してUIを立ち上げます．

```bash
git clone https://github.com/airbytehq/airbyte.git
cd airbyte
docker compose up
```

`docker compose`を実行したら，[http://localhost:8000](http://localhost:8000) でUIにアクセスすることができます．

ここからはUIの世界で全て完結することができます！

### Source
案内に従って進めると，まずプルダウンから今回のデータソースであるGoogle SheetsをSource typeとして選択します．右側にSetup Guideがあるので，見ながら設定できて親切設計だなと感じました．

![Google Sheets](../../img/airbyte-img2.png "Google Sheets")

以下の設定を埋めていきます．
- **Source name**
    - 「Google Sheets」としています
- **Authentication**
    - GCPのサービスアカウントを設定します．JSONファイルの中身をコピーして貼り付けます
    - `cat ~/.gcp/hoge_service_account.json | pbcopy`でクリップボードにコピーするとやりやすいです
- **Row Batch Size**
    - デフォルトの200にしています
- **Spreadsheet Link**
    - 対象となるスプレッドシートのURLを設定します
    - 事前にサービスアカウントでのアクセスを許可しておく必要があります

### Destination
次に，プルダウンからターゲットソースであるBigQueryをDestination typeとして選択します．

![BigQuery](../../img/airbyte-img3.png "BigQuery")

以下の設定を埋めていきます．
- **Destination name**
    - 「BigQuery」としています
- **Project ID**
    - GCPにアクセスして現在使っているプロジェクトIDを設定します
- **Dataset Location**
    - 「US」で良さそう？
- **Default Dataset ID**
    - BigQueryのデータセットとして作成されるIDになります
- **Loading Method**
    - Standard InsertsとGCS Stagingの2種類あります
    - Standard Inserts
        - SQL INSERTで直接アップロードする方法で，非効率的なためGCS Stagingを推奨しています（今回はこちらを選択）
    - GCS Staging
        - ファイルにレコードを書き込み，そのファイルをGCSにアップロードし，その後COPY INTOテーブルを使用してファイルをアップロードする方法（GCSのバケットなどの情報が必要になります）
- **Service Account Key JSON (Required for cloud, optional for open-source)**
    - GCPのサービスアカウントを設定します．JSONファイルの中身をコピーして貼り付けます
- **Transformation Query Run Type (Optional)**
    - interactiveとbatchの2種類あります
- **Google BigQuery Client Chunk Size (Optional)**
    - デフォルトの15にしています

### Connection
最後に，Connectionのセットアップを行います．設定したSourceとDestinationをセットし，Replication frequency（同期頻度）, Destination NamespaceやPrefixなどを決めていきます．

- **Transfer**
    - Replication frequency
        - 手動実行やスケジュール実行を選択できる
- **Streams**
    - Destination Namespace
        -  Mirror source structure, Destination default, Custom formatの3種類あります
    - Destination Stream Prefix (Optional)
        - 必要に応じて付与します
- **Normalization & Transformation**
    - Raw data (JSON)
    - Normalized tabular data
    - 上記どちらかを選択しますが，Raw dataだとJSONのままデータが格納されるので，Normalized tabular dataで良いと思います

個人的に同期するスプレッドシートの各シートがそれぞれ表示されて，どれを同期するか選択して決められるというのが感動しました 🎉

今回は1シートだけ連携することにし，Custom Transformの処理はせずに単純にデータをそのまま連携していきます．

![Connection](../../img/airbyte-img4.png "Connection")

実行結果のログは以下のような感じ`Sync Succeeded`となっています．

![Execution Logs](../../img/airbyte-img5.png "Execution Logs")

Airbyte側は大丈夫そうなので，BigQueryの方も確認してみると，ちゃんと入ってるので問題なさそうです！

![BigQuery Logs](../../img/airbyte-img6.png "BigQuery Logs")

## おわりに
今回は，OSSのAirbyteを使ったスプレッドシートからBigQueryへのデータ同期を行ってみました．AirbyteはUIが用意されていて，直感的に操作できる+設定も簡単でデータ同期の体験としてとても良かったです！データソースが豊富なのもメリットとして大きいと思います．

Extract & Loadのみしか使えていないので，次はdbtを理解してTransformも追加して処理を実行してみたいと思います．あとは他のデータパイプラインツールとの比較とかも出来たら楽しそうかなと思いました．
