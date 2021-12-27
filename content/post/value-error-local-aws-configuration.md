+++
author = "Masataka Kashiwagi"
title = "ValueError: Must setup local AWS configurationの対処方法"
description = ""
date = 2021-12-26T17:16:07+09:00
draft = false
share = true
showLicense = false
tags = ["Dev", "AWS"]
+++

## はじめに
Step FunctionsでSageMakerのProceesingJobを使ってカスタムコンテナを実行した際に，その実行スクリプト内でExperimentAnalyticsのAPIを使用していたところ，`ValueError: Must setup local AWS configuration with a region supported by SageMaker.`というエラーが発生したので，その対処方法をメモしておきます．

結論から言うと，エラー内容にある通り`region`の指定を行うことで解決できます．

方法としては2つあり，1つは`boto_session`と`sagemaker_client`の`region_name`を指定する．もう1つは環境変数に`AWS_DEFAULT_REGION`を設定する．

方法は2種類ありますが，Step Functionsの設定ファイルに環境変数`AWS_DEFAULT_REGION`を1行追記するのが簡単かと思います．あと，個人的には実行するStep Functionsの実行regionを設定して欲しい気持ちもあります．

## Configuration Error
SageMaker Experimentsに保存されている実験結果は`sagemaker.analytics.ExperimentAnalytics`のAPI使うことで取得することができます．今回，Step FunctionsでSageMakerのProceesingJobを使ってカスタムコンテナを実行した際に，以下のエラーが発生しました．

> `ValueError: Must setup local AWS configuration with a region supported by SageMaker.`

Configurationは公式ドキュメントを見ると以下のことが書かれています．

> Configuration - Overview <br>
> Boto3 looks at various configuration locations until it finds configuration values. Boto3 adheres to the following lookup order when searching through sources for configuration values:
> - A Config object that's created and passed as the config parameter when creating a client
> - Environment variables
> - The ~/.aws/config file

上から順番に優先度が高いものになっていて，下位で設定している内容よりも上位で設定した内容が反映されます．

今回の場合は，1番目と2番目は特段設定していないので，3番目が採用されて問題ないかなと思っていましたが，上述のエラーが発生しました．AWSのクレデンシャル情報（config, credentials）はローカルの`~/.aws/`配下に置かれており，build時にローカルにあるファイルをvolumesマウントしてコンテナ内と同期しています．

この設定では上手くいかなかったので，1番目or2番目の設定を行ったところ正常に動作したので，こちらの方法を記載しておきます．もし，volumesマウントの方法で上手くいく方法があれば教えて下さい！

## Config objectをboto3 clientに渡す方法
公式ドキュメントに記載されている優先度が1番高い方法になります．`botocore.config.Config`をインスタンス化して使うことで解決する方法になりますが，今回はわざわざConfig objectを使わずに`region_name`を指定する方法を説明します．

```python
import boto3
import sagemaker

# sessionとclientの設定を行う
boto_session = boto3.session.Session(region_name="ap-northeast-1")
sagemaker_client = boto_session.client(service_name='sagemaker', region_name="ap-northeast-1")
sess = sagemaker.session.Session(boto_session=boto_session, sagemaker_client=sagemaker_client)

# ExperimentAnalyticsをインスタンス化する
trial_component_analytics = sagemaker.analytics.ExperimentAnalytics(
    experiment_name='sample-experiments01',
    sagemaker_session=sess
)

# データフレーム化
analytics_tables = trial_component_analytics.dataframe()
```

`boto3.session.Session`のregion_nameとこのsessionを使ったclientのregion_nameにそれぞれ該当するregionを指定します．boto_sessionとsagemaker_clientを作ったら，それを`sagemaker.session.Session`の対応する引数に渡し，出来上がったsessionをさらに，`sagemaker.analytics.ExperimentAnalytics`のsagemaker_sessionの引数に渡すことで実験結果を取得できます．

この場合はカスタムコンテナで実行するスクリプトの修正変更が必要になってきますが，次に説明する環境変数に渡す方法はこの辺りの修正は必要ないので，簡単かなと思います．

## 環境変数を設定する方法
この方法は，Step Functionsのワークフローを定義するjsonファイルの"Environment"変数に`AWS_DEFAULT_REGION`を設定するだけになります．

Step Functionsの定義ファイルのパラメータ部分は下記のような感じです．（今回はSageMakerのProcessingJobを使って実行しています）

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

スクリプトの中身は以下になります．sessionの設定が必要なくなります．

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
今回は，Step FunctionsでSageMakerのProceesingJobを使った際に発生したエラーの対処方法を備忘録として残したものになります．

Configurationの設定に優先順位があることを知ったので，この辺りは今回に限らず注意が必要だなと思いました．今回のエラーに対する対処方法は複数あるので，開発している状況に合わせて使い分けていければと思います．


## 参考
- [Boto3 Docs - Configuration](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html)
- [How to fix aws region error "ValueError: Must setup local AWS configuration with a region supported by SageMaker"](https://stackoverflow.com/questions/55869651/how-to-fix-aws-region-error-valueerror-must-setup-local-aws-configuration-with)