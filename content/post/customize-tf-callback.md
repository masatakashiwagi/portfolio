+++
author = "Masataka Kashiwagi"
title = "Tensorflow の Callback 関数をカスタマイズ"
description = "Tensorflow の Callback 関数をカスタマイズした話を書きました"
date = 2021-11-10T00:14:33+09:00
draft = false
share = true
showLicense = false
tags = ["DEV", "ML"]
+++

## はじめに

AWS の SageMaker 上で SageMaker Python SDK を使用して独自の機械学習モデルを作成することができますが，その際に学習や評価が行える **[Estimator](https://sagemaker.readthedocs.io/en/stable/api/training/estimators.html)** という SageMaker の interface があります．<br>
一方で，SageMaker Experiments で実験管理を行いたい場合には，この Estimator に色々と渡してあげる必要があります．

その中でも学習時に出力される loss の値や評価メトリクスを記録するためには，Estimator の metric_definitions に正規表現を記述してログから上手く取得する必要があります．<br>
これをより簡単にするために，CustomCallback 関数を作成した話になります．

## Callback 関数のカスタマイズ

Tensorflow，厳密には Keras の Callback 関数をカスタマイズします．`tf.keras.callbacks.Callback` クラスを継承した `CustomCallBack(tf.keras.callbacks.Callback)` クラスを作成します．この作成したクラスを `model.fit` 時に引数の callbacks に渡してやることで使用することができます．

今回は SageMaker Experiments で使うことを想定したもので，Estimator の metric_definitions に渡す Regex として，以下のようなログが出力されて欲しいとします．（メトリクスは RMSE とした場合を想定）
MetricDefinitions はこちらが参考になります → [Define Metrics](https://docs.aws.amazon.com/sagemaker/latest/dg/automatic-model-tuning-define-metrics.html)

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

しかしながら，Keras でモデルを作成する際のデフォルトでは，学習時の Loss は「**loss**」，メトリクスは「**root_mean_squared_error**」で prefix が無い状態になります．これを Callback 関数をカスタマイズすることで prefix に「**train_**」を付けて，Regex で簡単に取得したいという気持ちです．

Tensorflow 公式ドキュメントの [Writing your own callbacks](https://www.tensorflow.org/guide/keras/custom_callback?hl=en) や [tf.keras.callbacks.Callback](https://www.tensorflow.org/api_docs/python/tf/keras/callbacks/Callback) を参考に作成しました．

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
        # the epoch when training is stopped
        self.stopped_epoch = 0
        # initialize the best loss as infinity
        self.best_loss = np.Inf
        # list of best metrics values
        self.best_metrics_values_list = []

    def on_train_end(self, logs=None):
        """Called at the end of training.
        """
        if self.stopped_epoch > 0:
            best_values = ' - '.join(self.best_metrics_values_list)
            print(f"Epoch {self.stopped_epoch + 1}: early stopping")
            print(f'Final results: {best_values}')

        print(f'Finish training - {str(datetime.datetime.now())}')

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

        current_loss = logs.get('val_loss')
        if np.less(current_loss, self.best_loss):
            self.best_loss = current_loss
            self.best_metrics_values_list = metrics_values_list
        else:
            self.stopped_epoch = epoch

# fit時にcallbacksに作成したカスタマイズクラスをインスタンス化したものを渡す
model.fit(
    ...,
    callbacks=[CustomCallBack()],
)
```

出力は以下のような感じになります．今回は `print` 文で出力していますが，logger を用意して `logger.info` を使うのも良いかと思います．

```python
Start training - 2021-11-09 23:48:12.787257
Epoch 1/10 - train_loss: 4.6889; - train_root_mean_squared_error: 2.1654; - val_loss: 11.1416; - val_root_mean_squared_error: 3.3379;
...
Finish training - 2021-11-09 23:48:15.095133
```

## おわりに

今回は SageMaker Experiments で実験管理を行う上でログ出力の形を修正したいという動機から CallBack 関数をカスタマイズしました．Callback 関数の中身を知るためにソースコードを読んだりして勉強になりました．Tensorflow のフレームワークは拡張性があり，カスタマイズの方法もドキュメントに整備されているので，比較的容易に修正できると思います．今回は時間経過や予測時間の表示は省いてしまったので，余裕があればログにこれらを出力するようにしていきたいです．

## 追記

- 2022/02/24: 更新

earlystopping に対応する形式に CallBack 関数を修正しました．`current_loss = logs.get('val_loss')` で logs から get する value は `callbacks.EarlyStopping(monitor='val_loss')` で `monitor` に指定している値になります．

## 参考

- [Sagemaker Training APIs - Estimator](https://sagemaker.readthedocs.io/en/stable/api/training/estimators.html)
- [Writing your own callbacks](https://www.tensorflow.org/guide/keras/custom_callback?hl=en)
- [tf.keras.callbacks.Callback](https://www.tensorflow.org/api_docs/python/tf/keras/callbacks/Callback)
