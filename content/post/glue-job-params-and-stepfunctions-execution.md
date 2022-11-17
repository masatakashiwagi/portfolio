+++
author = "Masataka Kashiwagi"
title = "Step FunctionsからGlueのジョブパラメータを指定して実行する方法"
description = "Step Functions経由でGlueにジョブパラメータを渡して実行したい時にどうすればいいか"
date = 2022-11-16T18:00:00+09:00
draft = false
share = true
showLicense = false
support = true
tags = ["AWS", "Dev"]
+++

## はじめに
Glueを使ってデータ連携する際に，例えばデータ連携したい期間を変えたり，環境情報を渡したり，などのパラメータを与えて実行したい場合の備忘録です．特に，Step Functions (SFn) 経由でGlueを実行する場合に，インプットパラメータに必要な情報を渡してそれをGlueにどう紐付けるかに関する内容になってます．

今回の内容は，実務で発生した検索エンジンのインデックスに簡単にデータを流せるように，SFnのパイプラインのインプットにパラメータを渡すだけで実行できるようにしたかったので，その時に実施したお話です．

## Glueのジョブパラメータ設定
ジョブパラメータはGlue実行時に渡すことができるパラメータで，デフォルトでもいくつか用意されています（参考：[Job parameters used by AWS Glue](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-glue-arguments.html)）．

これに自前で用意したパラメータを受け取りたい場合，`getResolvedOptions`の第二引数にリストに渡されてくるパラメータを定義します．これは後ほどのSFnのインプットに与えてGlueで受け取るものです．

```python
# JobName: sample_glue_job
import sys
from awsglue.utils import getResolvedOptions


# 動的に切り替えたい or 環境により変化する部分をパラメータとして受け取る
args = getResolvedOptions(
    sys.argv,
    ["period_date", "index_name"]
)

# データ連携したい期間
period_date = int(args["period_date"])
# データを投入する検索エンジンのインデックス名
index_name = args["index_name"]

...(実際の処理が続く)
```

**パラメータは文字列のみ**しか受け付けないので，期間を数値で使いたい場合は，上記コードのように数値型に変換する必要があります．

## Step Functionsのinputにデータを渡してGlueで使う方法
Glueジョブが作成できたら，それをStep Functionsで動かしていきます．

以下にStep FunctionsのState Machineのサンプルコードを載せていますが，ポイントは`glue:startJobRun.sync`アクションのParametersにあるArgumentsの設定です．inputパラメータを渡す場合は，`$`をkeyの末尾とvalueの先頭に付与する必要があります（参考：[Pass Parameters to a Service API](https://docs.aws.amazon.com/step-functions/latest/dg/connect-parameters.html)）．

また，keyの先頭に`--`を付与しないとパラメータと認識されずにエラーになるので，注意が必要です．

```json
{
  "Comment": "Glueでデータ連携を行うステートマシン",
  "StartAt": "Glue-Job",
  "States": {
    "Glue-Job": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "sample_glue_job",
        "Arguments": {
          "--period_date.$": "$.period_date",
          "--index_name.$": "$.index_name"
        }
      },
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "FailState"
        }
      ],
      "Comment": "データ連携用のGlueジョブ",
      "Next": "Success"
    },
    "Success": {
      "Type": "Succeed"
    },
    "FailState": {
      "Type": "Fail",
      "Cause": "Error",
      "Error": "Error"
    }
  }
}
```

SFnのコンソールから実行する場合，下図のように実行前の画面でJSON形式で必要なパラメータを渡すことで，SFnで定義したGlueのジョブパラメータに値がセットされます．

今回の場合だと，`period_date`と`index_name`をパラメータとしてSFnのinputから渡して，Glueでそれらを受け取りETL処理を実行していきます．

![SFnのコンソール実行画面](../../img/glue-job-params-sfn-img1.png "SFnのコンソール実行画面")

## おわりに
今回はStep Functionsのinputパラメータを変更することで，簡単にGlueのジョブパラメータに値を渡してパイプラインを実行する方法を紹介しました．

Glue Studioからコードを直接変更することでもできますが，毎回コードを変更するのはバグが混入する可能性もあるので，パイプライン実行時のinputで制御できた方が汎用性があり，シンプルかなと思い試してみました．

## 参考
- [Job parameters used by AWS Glue](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-glue-arguments.html)
- [Pass Parameters to a Service API](https://docs.aws.amazon.com/step-functions/latest/dg/connect-parameters.html)