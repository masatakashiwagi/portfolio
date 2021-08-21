+++
author = "Masataka Kashiwagi"
title = "CUDA error: Device-side assert triggeredの解決方法"
date = 2021-02-01T16:14:09+09:00
description = "CUDA error: Device-side assert triggeredの解決方法"
tags = ["dev"]
showLicense = false
+++

## RuntimeError: CUDA error: device-side assert triggeredの解決方法
Pytorchでモデルを作成していた際に，`RuntimeError: CUDA error: device-side assert triggered`が発生して，原因がよくわからなかったので，調べたことをメモしておきます．

## エラー発生の原因
調べてみると，原因としては
- ライブラリのVersionが違う
- ラベル/クラスの数とネットワークの入出力のshapeが異なる
- loss関数の入力が正確でない

などなど...

よくあるのが，下2つかなと思います．

### ラベル/クラスの数とネットワークの入出力のshapeが異なる
想定しているラベルもしくはクラス数とネットワークの出力のクラス数が異なる場合，この場合は`nn.Linear(input, num_class)`で合わせてやる必要がある．

### loss関数の入力が正確でない
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


## 参考
https://towardsdatascience.com/cuda-error-device-side-assert-triggered-c6ae1c8fa4c3
