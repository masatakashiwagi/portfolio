+++
author = "Masataka Kashiwagi"
title = "データフレームでリストを ndarray にした値を csv ファイルで保存する場合の注意"
description = "ndarray を csv ファイルで保存する時は注意が必要だという話を書きました"
date = 2023-07-31T13:39:35+09:00
draft = false
share = true
showLicense = false
support = true
tags = ["DEV", "DATA"]
+++

## はじめに

Python のデータフレームを使ってレコメンドリストを生成した場合に遭遇した内容で，備忘録がてら筆を取ってます．内容としては，リストを ndarray に変換して，それらの値が入ったデータフレームを csv ファイルとして保存し，再度ロードする場合に発生する事象への対応方法になります．

結論は，データフレームにリスト形式のデータを保存したい場合は，str 型に変換してから保存するようにしましょう！ということです．これは csv のカンマ区切りでの保存による影響を受けてしまうからです．

## 発生した事象

例えば，各ユーザーに対してアイテムをレコメンドしたい場合を考えてみます．各ユーザーに対して用意されたレコメンドリストがあるとします．10人ユーザーが存在するとしてサンプルデータを作って考えてみると，

```python
import random
import pandas as pd
import numpy as np

user_id = [random.randint(1, 99999) for _ in range(10)]
recommend_items = [np.array([random.randint(1, 99999) for _ in range(3)]) for _ in range(10)]

# recommend_itemsのカラムには，リストをndarrayに変換したデータが入っている
df_rec = pd.DataFrame({"user_id": user_id, "recommend_items": recommend_items})

print(df_rec['recommend_items'][0])
> array([4775, 44953, 85874])  # 数値は適当に入れてます

print(type(df_rec['recommend_items'][0]))
> numpy.ndarray
```

データフレームの中身を見ると，以下のようなデータが入っています．

|  | user_id | recommend_items |
| :---: | ---: | ---: |
| 0 | 8482 | [4775, 44953, 85874] |
| 1 | 83899 | [96090, 53639, 82456] |
| 2 | 93606 | [8355, 76477, 58666] |

recommend_items のカラムのデータの実態は ndarray になっているのですが，見た目はリスト型のデータが入っていると勘違いが起きたりします．特に色々な処理を行っていくと，途中で型が変わっていることに気づいていない場合もあるかと思います．

ここで，各ユーザーのレコメンドリストをデモ的に ndarray に変換していますが，

- BigQuery に配列で格納されているデータをロードした場合
- 処理過程で，Numpy 配列に変換した方が高速に処理でき，結果をそのまま格納した場合
- etc...

などの場合で生じることがあるかなと思います．

このデータフレームを csv ファイルに保存して，ロードすると，

```python
# csvファイルで結果を保存
df_rec.to_csv("./tmp_recommend_results1.csv", index=False)

# 保存したcsvファイルをデータフレームとしてロードする
tmp_df_rec = pd.read_csv("./tmp_recommend_results1.csv")

print(tmp_df_rec['recommend_items'][0])
> [ 4775 44953 85874]
```

要素を1つ取り出してみると，カンマが取れて右詰めのような形式で表示されます．

## ndarray を csv ファイルに保存する前にリストに変換する

ndarray のまま csv ファイルに保存してしまうと，ロード時に意図しない形式で処理ができなくなってしまうので，簡単な方法としては，事前にリストに変換してからデータフレームに入れることが望ましいです．

```python
recommend_items_list = [list(arr) for arr in df_rec['recommend_items']]
df_rec['recommend_items'] = recommend_items_list

print(type(df_rec['recommend_items'][0]))
> list

df_rec.to_csv("./tmp_recommend_results2.csv", index=False)
tmp_df_rec = pd.read_csv("./tmp_recommend_results2.csv")

print(type(tmp_df_rec['recommend_items'][0]))
> <class 'str'>
```

リストに変換してから csv ファイルで保存すると，値は文字列になります．リストとして再度使いたい場合は，以下のように組み込み関数の `eval()` か，`ast.literal_eval()` を使って文字列をリストに変換するのが良いです．

```python
import ast

# 以下のどちらかを使うと良い
tmp_df_rec['recommend_items'] = tmp_df_rec['recommend_items'].apply(eval)
tmp_df_rec['recommend_items'] = tmp_df_rec['recommend_items'].apply(ast.literal_eval)

print(type(type(tmp_df_rec['recommend_items'][0])))
> list
```

`eval()` と `ast.literal_eval()` の違いは，`ast.literal_eval()` はリテラルのみを含む式を評価するのに対して，`eval()` はリテラルに加え変数およびそれらの演算を含んだ式を評価することができます．

これでリストとして元の値を使うことができます．

## おわりに

今回の事象に全然気づかずにそのままデータを保存していて，いざ分析しようとした時に，あれ？なんかおかしいぞとなり発覚しました．正規表現などを使えば無理やりできるのかもしれないですが，あまり好ましいやり方ではないと思うので，今回のように忘れずにリストに変換するのが良いと思います．

あとはそもそも csv ファイルに保存せずに別のファイル形式で保存するやり方もあると思います．が，なんやかんや分析する場合は csv ファイルが楽だったりもします笑

でも，最近 csv ファイル絡みでの予期せぬことに遭遇することが多くて嫌になりつつあります...😅

## 参考

- [Pythonのast.literal_eval()で文字列をリストや辞書に変換](https://note.nkmk.me/python-ast-literal-eval/)
