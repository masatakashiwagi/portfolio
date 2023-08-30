+++
author = "Masataka Kashiwagi"
title = "RuntimeError: CUDA error: Device-side assert triggered の解決方法"
date = 2021-02-01T16:14:09+09:00
lastmod = 2023-08-30T14:28:59+09:00
description = "RuntimeError: CUDA error: Device-side assert triggered の解決方法"
tags = ["Dev", "Machine Learning"]
showLicense = false
share = true
+++

## はじめに

Pytorch でモデルを作成していた際に，「**RuntimeError: CUDA error: device-side assert triggered**」が発生し，原因がよくわからなかったので，調べたことをメモしておきます．

## エラー発生の原因

調べてみると，原因としては以下のようなものがあります．

- ライブラリの Version が違う
- ラベル/クラスの数とネットワークの入出力の shape が異なる
- Loss 関数の入力が正確でない

などなど...

よくあるのが，下2つかなと思います．

### ラベル/クラスの数とネットワークの入出力の shape が異なる

想定しているラベルもしくはクラス数とネットワークの出力のクラス数が異なる場合，この場合は FC 層の最後に `nn.Linear(input, num_class)` を入れて調整する必要があります．

### Loss 関数の入力が正確でない

僕が遭遇したのはこちらのパターンになります．

例えば，BCELoss を考えた場合，計算するためには値としては0~1を取る必要があります．そのため普通は最終出力に **Sigmoid関数** or **Softmax関数** を入れるかと思います．

それ以外にも Loss の設計で以下のようにしておくと良いかと思います．

```python
class BCELoss(nn.Module):
    def __init__(self):
        super().__init__()
        self.bce = nn.BCELoss()

    def forward(self, input, target):
        input = torch.where(torch.isnan(input), torch.zeros_like(input), input)
        input = torch.where(torch.isinf(input), torch.zeros_like(input), input)
        input = torch.where(input>1, torch.ones_like(input), input)  # 1を超える場合には1にする

        target = target.float()

        return self.bce(input, target)
```

### 他の解決方法

他にも調べていると解決方法として CUDA の設定を以下にすると良いなどもありましたが，解決するかどうかはよくわからないです．

```bash
CUDA_LAUNCH_BLOCKING=1
```

## おわりに

今回は，Pytorch でのモデル作成時に発生したエラーについて整理しました，モデル作成時にはモデルの In/Out や Loss 関数の定義をきちんと理解し把握しておく必要があると改めて感じました．同様のエラーが起きた場合には，この辺りをまずは調べてみるのが良さそうです．

## 参考

- [CUDA error 59: Device-side assert triggered](https://towardsdatascience.com/cuda-error-device-side-assert-triggered-c6ae1c8fa4c3)
