+++
author = "Masataka Kashiwagi"
title = "ValueError: Must setup local AWS configuration の対処方法"
description = "local AWS configuration の設定方法を備忘録として書いた"
date = 2021-12-26T17:16:07+09:00
draft = false
share = true
showLicense = false
support = true
tags = ["Dev", "AWS"]
+++

## はじめに

Step Functions で SageMaker ProceesingJob を使ってカスタムコンテナを実行した際に，その実行スクリプト内で ExperimentAnalytics の API を使用していたところ，「<span class="marker_yellow">**ValueError: Must setup local AWS configuration with a region supported by SageMaker.**</span>」というエラーが発生したので，その対処方法をメモしておきます．

結論から言うと，エラー内容にある通り「<span class="marker_yellow">**region**</span>」の指定を行うことで解決できます．

方法としては2つあります．

- 「**boto_session** と **sagemaker_client**」の「**region_name**」を指定する
- 環境変数に「**AWS_DEFAULT_REGION**」を設定する

2つ目の方法の Step Functions の定義ファイルに環境変数: `AWS_DEFAULT_REGION` を1行追記するのが簡単かと思います．

## Configuration Error

SageMaker Experiments に保存されている実験結果は `sagemaker.analytics.ExperimentAnalytics` の API 使うことで取得することができます．今回，Step Functions で SageMaker ProceesingJob を使ってカスタムコンテナを実行した際に，以下のエラーが発生しました．

> ValueError: Must setup local AWS configuration with a region supported by SageMaker.

Configuration は公式ドキュメントを見ると以下のことが書かれています．

> Configuration - Overview <br>
> Boto3 looks at various configuration locations until it finds configuration values. Boto3 adheres to the following lookup order when searching through sources for configuration values:
> - A Config object that's created and passed as the config parameter when creating a client
> - Environment variables
> - The ~/.aws/config file

上から順番に優先度が高いものになっていて，下位で設定している内容よりも上位で設定した内容が反映されます．

今回の場合は，1番目と2番目は特段設定していないので，3番目が採用されて問題ないかなと思っていましたが，上述のエラーが発生しました．AWS のクレデンシャル情報（config, credentials）はローカルの `~/.aws/` 配下に置かれており，build 時にローカルにあるファイルを volumes マウントしてコンテナ内と同期しています．

この設定では上手くいかなかったので，1番目 or 2番目の設定を行ったところ正常に動作したので，こちらの方法を記載しておきます．もし，volumes マウントの方法で上手くいく方法があれば教えて下さい！

## Config object を boto3 client に渡す方法

公式ドキュメントに記載されている優先度が1番高い方法になります．`botocore.config.Config` をインスタンス化して使うことで解決する方法になりますが，今回はわざわざ Config object を使わずに `region_name` を指定する方法を説明します．

```python
import boto3
import sagemaker

# sessionとclientの設定を行う
boto_session = boto3.session.Session(region_name="ap-northeast-1")
sagemaker_client = boto_session.client(service_name='sagemaker', region_name="ap-northeast-1")
sagemaker_session = sagemaker.session.Session(boto_session=boto_session, sagemaker_client=sagemaker_client)

# ExperimentAnalyticsをインスタンス化する
trial_component_analytics = sagemaker.analytics.ExperimentAnalytics(
    experiment_name='sample-experiments01',
    sagemaker_session=sagemaker_session
)

# データフレーム化
analytics_tables = trial_component_analytics.dataframe()
```

コードの流れは以下になります．

- boto_session と sagemaker_client を作成する
- `sagemaker.session.Session` の引数にそれぞれを渡す
- sagemaker_session を `sagemaker.analytics.ExperimentAnalytics` の引数に渡す
- 取得した実験結果をデータフレーム化する

ここで，最初の「boto_session と sagemaker_client を作成する」部分で，「`boto3.session.Session` の region_name」と，この session を使った「client の region_name」に該当する region を指定する必要があります．この2つをセットしておくことで，今回発生したエラーを回避することができます．

この場合はカスタムコンテナで実行するスクリプトの修正変更が必要になってきますが，次に説明する環境変数に渡す方法はこの辺りの修正は必要ないので，簡単かなと思います．

## 環境変数を設定する方法

Step Functions のワークフローを定義する json ファイルの Environment 変数に「<span class="marker_yellow">**AWS_DEFAULT_REGION**</span>」を設定する方法になります．

Step Functions の定義ファイルのパラメータ部分は下記のような感じです．（今回は SageMaker ProcessingJob を使って実行しています）

```json
"Parameters": {
  "AppSpecification": {
    "ImageUri": "hogehoge.dkr.ecr.ap-northeast-1.amazonaws.com/mlops-experiments:latest",
    "ContainerEntrypoint": [
    "python3",
    "/opt/ml/code/get_experiments.py"
    ]
  },
  "Environment": {
    "AWS_DEFAULT_REGION": "ap-northeast-1"
  }
}
```

スクリプトの中身は以下になります．session の設定が必要なくなります．

```python
import sagemaker

# ExperimentAnalyticsをインスタンス化する
trial_component_analytics = sagemaker.analytics.ExperimentAnalytics(
    experiment_name='sample-experiments01'
)

# データフレーム化
analytics_tables = trial_component_analytics.dataframe()
```

## おわりに

今回は，Step Functions で SageMaker ProceesingJob を使った際に発生したエラーの対処方法を備忘録として残したものになります．

Configuration の設定に優先順位があることを知ったので，この辺りは今回に限らず注意が必要だなと思いました．今回のエラーに対する対処方法は複数あるので，開発している状況に合わせて使い分けていければと思います．

あと，個人的には実行している Step Functions の region をセットして欲しい気持ちもありますが，まーこれは状況次第なので，なんとも言えない気もします...

## 参考

- [Boto3 Docs - Configuration](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html)
- [How to fix aws region error "ValueError: Must setup local AWS configuration with a region supported by SageMaker"](https://stackoverflow.com/questions/55869651/how-to-fix-aws-region-error-valueerror-must-setup-local-aws-configuration-with)
