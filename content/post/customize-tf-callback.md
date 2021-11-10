+++
author = "Masataka Kashiwagi"
title = "TensorflowのCallback関数をカスタマイズ"
description = "TensorflowのCallback関数をカスタマイズした話"
date = 2021-11-10T00:14:33+09:00
draft = false
share = true
showLicense = false
tags = ["dev", "machine learning"]
+++

## はじめに
AWSのSageMaker上でSageMaker Python SDKを使用して独自の機械学習モデルを作成することができますが，その際に学習や評価が行える**[Estimator](https://sagemaker.readthedocs.io/en/stable/api/training/estimators.html)**というSageMakerのinterfaceがあります．<br>
一方で，SageMaker Experimentsで実験管理を行いたい場合には，このEstimatorに色々と渡してあげる必要があります．

その中でも学習時に出力されるlossの値や評価メトリクスを記録するためには，Estimatorのmetric_definitionsに正規表現を記述してログから上手く取得する必要があります．<br>
これをより簡単にするために，CustomCallback関数を作成した話になります．

## Callback関数のカスタマイズ
Tensorflow，厳密にはKerasのCallback関数をカスタマイズします．`tf.keras.callbacks.Callback`クラスを継承した`CustomCallBack(tf.keras.callbacks.Callback)`クラスを作成します．この作成したクラスを`model.fit`時に引数のcallbacksに渡してやることで使用することができます．

今回はSageMaker Experimentsで使うことを想定したもので，Estimatorのmetric_definitionsに渡すRegexとして，以下のようなログが出力されて欲しいとします．（メトリクスはRMSEとした場合を想定）
MetricDefinitionsはこちらが参考になります→[Define Metrics](https://docs.aws.amazon.com/sagemaker/latest/dg/automatic-model-tuning-define-metrics.html)

```python
sagemaker.estimator.Estimator(
    ...,
    metric_definitions={
        {'Name': 'Train Loss', 'Regex': 'train_loss: (.*?);'},
        {'Name': 'Validation Loss', 'Regex': 'val_loss: (.*?);'},
        {'Name': 'Train Metrics', 'Regex': 'train_root_mean_squared_error: (.*?);'},
        {'Name': 'Validation Metrics', 'Regex': 'val_root_mean_squared_error: (.*?);'},
    }
)
```

しかしながら，Kerasでモデルを作成する際のデフォルトでは，学習時のLossは`loss`，メトリクスは`root_mean_squared_error`でprefixが無い状態になります．これをCallback関数をカスタマイズすることでprefixに`train_`を付けて，Regexで簡単に取得したいという気持ちです．

Tensorflow公式ドキュメントの[Writing your own callbacks](https://www.tensorflow.org/guide/keras/custom_callback?hl=en)や[tf.keras.callbacks.Callback](https://www.tensorflow.org/api_docs/python/tf/keras/callbacks/Callback)を参考に作成しました．

```python
import tensorflow as tf

class CustomCallback(tf.keras.callbacks.Callback):
    def on_train_begin(self, logs=None):
        """Called at the beginning of training.
        """

    def on_train_end(self, logs=None):
        """Called at the end of training.
        """

    def on_epoch_begin(self, epoch, logs=None):
        """Called at the start of an epoch.
        """

    def on_epoch_end(self, epoch, logs=None):
        """Called at the end of an epoch.
        """

    def on_train_batch_begin(self, batch, logs=None):
        """Called at the beginning of a training batch in fit methods.
        """

    def on_train_batch_end(self, batch, logs=None):
        """Called at the end of a training batch in fit methods.
        """
```

上記コードの中で必要なものを修正すれば大丈夫です．今回は学習の開始終了とエポックの終了時に呼ぶように修正しました．

```python
import tensorflow as tf
import datetime

class CustomCallBack(tf.keras.callbacks.Callback):
    def on_train_begin(self, logs=None):
        """Called at the beginning of training.
        """
        print(f"Start training - {str(datetime.datetime.now())}")

        # get parameters
        self.epochs = self.params['epochs']

    def on_train_end(self, logs=None):
        """Called at the end of training.
        """
        print(f"Finish training - {str(datetime.datetime.now())}")

    def on_epoch_end(self, epoch, logs=None):
        """Called at the end of an epoch.
        """
        keys = list(logs.keys())
        metrics_values_list = []

        for key in keys:
            if key.startswith('val_'):
                metrics_values_list.append(f"{key}: {logs.get(key):.4f};")
            else:
                metrics_values_list.append(f"train_{key}: {logs.get(key):.4f};")
        values = ' - '.join(metrics_values_list)

        print(f"Epoch {epoch+1}/{self.epochs} - {values}")

# fit時にcallbacksに作成したカスタマイズクラスをインスタンス化したものを渡す
model.fit(
    ...,
    callbacks=[CustomCallBack()],
)
```

出力は以下のような感じになります．今回は`print`文で出力していますが，loggerを用意して`logger.info`を使うのも良いかと思います．
```python
Start training - 2021-11-09 23:48:12.787257
Epoch 1/10 - train_loss: 4.6889; - train_root_mean_squared_error: 2.1654; - val_loss: 11.1416; - val_root_mean_squared_error: 3.3379;
...
Finish training - 2021-11-09 23:48:15.095133
```

## おわりに
今回はSageMaker Experimentsで実験管理を行う上でログ出力の形を修正したいという動機からCallBack関数をカスタマイズしました．Callback関数の中身を知るためにソースコードを読んだりして勉強になりました．Tensorflowのフレームワークは拡張性があり，カスタマイズの方法もドキュメントに整備されているので，比較的容易に修正できると思います．今回は時間経過や予測時間の表示は省いてしまったので，余裕があればログにこれらを出力するようにしていきたいです．


## 参考
- [Sagemaker Training APIs - Estimator](https://sagemaker.readthedocs.io/en/stable/api/training/estimators.html)
- [Writing your own callbacks](https://www.tensorflow.org/guide/keras/custom_callback?hl=en)
- [tf.keras.callbacks.Callback](https://www.tensorflow.org/api_docs/python/tf/keras/callbacks/Callback)