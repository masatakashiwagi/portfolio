+++
author = "Masataka Kashiwagi"
title = "ANN ライブラリである Annoy の build index 時に Illegal Instruction Error が発生した"
description = "Annoy で build index する際に発生したエラーについての対処方法"
date = 2023-08-09T15:56:37+09:00
draft = false
share = true
showLicense = false
support = true
tags = ["Dev", "Machine Learning"]
+++

## はじめに

[Annoy](https://github.com/spotify/annoy) という Spotify が開発している Python 製の ANN (Approximate Nearest Neighbors) のライブラリがあり，それを使ってレコメンドアイテムの類似度を計算する機会があったのですが，コンテナ化したものを Vertex AI Pipelines 上で動かしていたところ，`Fatal Python error: Illegal instruction` というエラーが発生して困っていたので，今回はこのエラーの対処方法について書いていこうと思います．

<!-- 結論としては，ローカルの M1 Mac と GitHub Actions 経由での docker build を両立するためには，Dockerfile に Annoy のライブラリをインストールする時に使用する `ANNOY_COMPILER_ARGS` という環境変数を追加することで解決できました🎉 -->

※ Annoy については，[こちらの ZOZO さんの記事](https://techblog.zozo.com/entry/annoy-explanation)が詳しく解説してくれています．

## docker build 時の環境による問題？

### 発生した事象

Annoy のライブラリを含んだ Docker イメージを作り，それを元に作成したコンポーネントを機械学習パイプラインである Vertex AI Pipelines で動かしたところ，`Fatal Python error: Illegal instruction` というエラーが発生しました．これは具体的には，`AnnoyIndex` を構築する際に発生しているように思われます．

Annoy を含む Python ライブラリは poetry で管理し，Dockerfile 内で `poetry install` しています．

### エラー原因

今回の問題は，Docker コンテナのイメージをビルドする時に指定するオプションが原因でした．

ローカル環境は Apple M1 Max (ARM アーキテクチャ) で開発していたため，イメージ作成時に以下のオプションを指定してビルドしていました．

```bash
docker build --platform linux/amd64 -t sample-recsys:latest -f ./Dockerfile .
```

AMD 環境でも動くように `--platform` フラグにターゲットプラットフォームである `linux/amd64` を指定してビルドしていました．このビルドしたイメージを GCP の Artifact Registry にプッシュし，そのイメージを使って Vertex AI Pipelines で各コンポーネントの検証を行っていました．

一方で，`--platform` フラグを付けずに M1 Mac からビルドしたイメージを使った場合には，`exec format error` というエラーが発生します．

検証時には，ローカル環境から直接イメージをビルド & プッシュし，そのイメージを使ったコンポーネントを Vertex AI Pipelines 上で動かして検証を行っていました．その際に使用していたスクリプトをそのまま GitHub Actions での CI/CD 構築時に使用したことで，今回の `Fatal Python error: Illegal instruction` という事象が発生することになります．

`Illegal instruction` というエラーが発生するという事象はいくつか Issue が上がっていました．

- [Error: illegal hardware instruction (core dumped) #492](https://github.com/spotify/annoy/issues/492)
- ["Illegal instruction: 4" when trying to build index (Python 3.7, Annoy version 1.17.0, macOS 10.14.6) #513](https://github.com/spotify/annoy/issues/513)

`--platform` フラグを付けてイメージをビルドしても問題ないライブラリも多数あるのですが，Annoy では `linux/amd64` を指定したのを GitHub Actions の Runner ([ubuntu-latest](https://github.com/actions/runner-images/blob/main/images/linux/Ubuntu2204-Readme.md)) で動かしたのがどうも上手く行かなかったみたいです．

- [Initializing an AnnoyIndex crashes on AMD processors #472](https://github.com/spotify/annoy/issues/472)

## 解決方法

僕の例では，[Initializing an AnnoyIndex crashes on AMD processors #472](https://github.com/spotify/annoy/issues/472) の Issue に記載されている方法で解決することができました．

Annoy のライブラリをインストールする際に，コンパイルパラメータである `ANNOY_COMPILER_ARGS` を Dockerfile 内に環境変数として指定することで解決することができました．[annoy/setup.py](https://github.com/spotify/annoy/blob/main/setup.py) を見ると良いかもしれません．

```dockerfile
ENV ANNOY_COMPILER_ARGS -D_CRT_SECURE_NO_WARNINGS,-DANNOYLIB_MULTITHREADED_BUILD,-mtune=native
```

この環境変数を Dockerfile にセットすると，`--platform` に `linux/amd64` を付け Docker イメージを GitHub Actions 経由でビルドしたものを Vertex AI Pipelines で使用しても，エラー無く処理が完了しました．

あとは，そもそもこれはローカルの M1 Mac と GitHub Actions の両方で同一のスクリプトを使用したいが為に行っている対応策なので，スクリプトを別々にすれば，GitHub Actions でビルドする際には，上記の `ANNOY_COMPILER_ARGS` を設定することなく，シンプルに以下のビルドコマンドだけでいけます．

```bash
docker build -t sample-recsys:latest -f ./Dockerfile .
```

また，，`--platform` のオプションを付けてビルドすると時間が余計にかかるみたいなので，無駄なものは付けない方が良さそうですね．

## おわりに

今回は思いもよらないエラーで悩んでいたので，こうして無事？エラーの原因と解決策が分かって良かったです．全然このことに気づかなかったので，Annoy をやめて Faiss を使用することも考えていました（Faiss でも同じことになっていたかもです笑）．

コンテナイメージをビルドする際のローカルとプロダクション適用時のマシンスペックが違うことも考えた上で，再現性を意識したコードを書かなければと改めて感じたので，良い教訓となりそうです．

## 参考

- [近傍探索ライブラリ「Annoy」のコード詳解](https://techblog.zozo.com/entry/annoy-explanation)
