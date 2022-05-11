+++
author = "Masataka Kashiwagi"
title = "Excel Binary WorkbookをPythonで処理"
date = 2019-10-05T15:16:44+09:00
description = "EBWを処理するために"
tags = ["Dev"]
showLicense = false
share = true
+++

## はじめに
今回は，先日初めて見たファイル形式のExcel Binary Workbook (xlsb)に関して，pythonでcsvにパースする話です．

`.xlsx`はよくあるExcelファイルの形式ですが，それのバイナリー形式である`.xlsb`に関しての話です．（今まで見たことなかった拡張子です笑）

Excelの闇やExcelとの格闘は色々ありますが，今回はそこをグッと堪えて進めたいと思います笑

## .xlsbとは
[Weblio辞書](https://www.weblio.jp/content/.xlsb)によると以下のように記載されています．
> .xlsbとは，Excel 2007で作成したブックを「XML形式でないバイナリブック」として保存する際に用いられる拡張子である．.xlsbでブックを保存した場合はファイル全体がバイナリ形式で保存され，XMLベースである.xlsxなどのファイル形式で保存した場合と比べて，ファイルサイズを数分の1程度に抑えることができる．

受け取ったファイルは`.xlsb`形式でも100MBぐらいあったため，`.xlsx`形式だとかなり容量が大きく，ファイルを開くと処理が重たくなることが想像できるので，圧縮したのだと考えられます．

## Excelを扱えるpythonライブラリ
### openpyxl
定番の[openpyxl](https://openpyxl.readthedocs.io/en/stable/)です．

上記ページにも記載されてますが，Excelファイルの拡張子である`.xlsx`を扱うことができます．
> openpyxl is a Python library to read/write Excel 2010 xlsx/xlsm/xltx/xltm files.

いつものようにこのライブラリで処理しようとしたところ，下記のようなエラーが発生しました．
```bash
openpyxl.utils.exceptions.InvalidFileException: openpyxl does not support binary format .xlsb, please convert this file to .xlsx format if you want to open it with openpyxl
```
もう一度openpyxlの説明を見ると，確かに扱える拡張子は`xlsx/xlsm/xltx/xltm`となっているので、`.xlsb`は扱えないのが分かります．

そこで，`.xlsb`の拡張子が扱えるライブラリを調べたところ，pyxlsbというのがあるみたいなので，それを使うことにしました．

### pyxlsb
[pyxlsb](https://github.com/wwwiiilll/pyxlsb)は公式の説明にあるように，xlsb形式を扱えるpythonライブラリになります．
> pyxlsb is an Excel 2007-2010 Binary Workbook (xlsb) parser for Python.

インストールはpipですることができます．
```python
pip install pyxlsb
```

公式のサンプルコードを記載しておきます．
```python
import csv
from pyxlsb import open_workbook

with open_workbook('Book1.xlsb') as wb:
    for name in wb.sheets:
        with wb.get_sheet(name) as sheet, open(name + '.csv', 'w') as f:
            writer = csv.writer(f)
            for row in sheet.rows():
                writer.writerow([c.v for c in row])
```

もしpandasのデータフレームに変換したい場合は，参考ページのコードで可能となります．

ただし，時刻変換に関して少し注意が必要なので，記載しておくと公式にもある通り，日付は**float**に変換されてしまうため，**convert_date関数**を使う必要があります．
> Note that dates will appear as floats. You must use the convert_date(date) method from the corresponding Workbook instance to turn them into datetime.

なので，元のファイルに時刻が入っている場合には上記変換をコードの中に入れて処理する必要がありますのでご注意下さい．
```python
print(wb.convert_date(41235.45578))
>>> datetime.datetime(2012, 11, 22, 10, 56, 19)
```

今回は個人的に嵌ってしまった`.xlsb`形式のファイルを扱う方法を紹介しましたが，出来ればデータ分析をするようなデータをExcelファイルで扱いたくないのが本音です．

もちろん簡単なデータの可視化とか表計算とかExcelが活躍する場面は多々あると思うので，使い分けていきたいとは思います．

## 参考
- [Stack Overflow - Read XLSB File in Pandas Python](https://stackoverflow.com/questions/45019778/read-xlsb-file-in-pandas-python)

## 追記
- 2020/01/05: 更新

pandasの「**version=1.0.0**」で`.xlsb`ファイルをロードできるようになったみたいです．方法は`pd.read_excel`の引数で`engine="pyxlsb"`と指定するだけです．

```python
# Returns a DataFrame
pd.read_excel("path_to_file.xlsb", engine="pyxlsb")
```

参考: https://pandas.pydata.org/docs/user_guide/io.html#io-xlsb