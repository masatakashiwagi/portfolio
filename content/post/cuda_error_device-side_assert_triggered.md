+++
author = "Masataka Kashiwagi"
title = "RuntimeError: CUDA error: Device-side assert triggeredの解決方法"
date = 2021-02-01T16:14:09+09:00
description = "RuntimeError: CUDA error: Device-side assert triggeredの解決方法"
tags = ["Dev"]
showLicense = false
share = true
+++

## はじめに
Pytorchでモデルを作成していた際に，`RuntimeError: CUDA error: device-side assert triggered`が発生して，原因がよくわからなかったので，調べたことをメモしておきます．

## エラー発生の原因
調べてみると，原因としては
- ライブラリのVersionが違う
- ラベル/クラスの数とネットワークの入出力のshapeが異なる
- Loss関数の入力が正確でない

などなど...

よくあるのが，下2つかなと思います．

### ラベル/クラスの数とネットワークの入出力のshapeが異なる
想定しているラベルもしくはクラス数とネットワークの出力のクラス数が異なる場合，この場合は`nn.Linear(input, num_class)`で合わせてやる必要がある．

### Loss関数の入力が正確でない
僕が遭遇したのはこちらのパターンになります．

例えば，BCELossを考えた場合，計算するためには値としては0~1を取る必要があります．そのため普通は最終出力に`Sigmoid, Softmax関数`を入れるかと思います．

それ以外にもLossの設計で以下のようにしておくと良いかと思います．

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
他にも調べていると解決方法としてCUDAの設定を以下にすると良いなどもありましたが，解決するかどうかはよくわからないです．

```bash
CUDA_LAUNCH_BLOCKING=1
```

## おわりに
今回は，Pytorchでのモデル作成時に発生したエラーについて整理しました，モデル作成時にはモデルのIn/OutやLoss関数の定義をきちんと理解し把握しておく必要があると改めて感じました．<br>
同様のエラーが起きた場合には，この辺りをまずは調べてみるのが良さそうです．

## 参考
- [CUDA error 59: Device-side assert triggered](https://towardsdatascience.com/cuda-error-device-side-assert-triggered-c6ae1c8fa4c3)
