+++
author = "Masataka Kashiwagi"
title = "teamayaという個人プロジェクトをはじめてみた"
description = "家計簿のデータをBigQueryに連携して遊ぶプロジェクト"
date = 2021-12-12T16:55:14+09:00
draft = false
share = true
showLicense = false
tags = ["Dev", "Poem"]
+++

## はじめに
最近，データエンジニアリングとMLOpsの領域への関心が個人的に高まっていて，何か個人プロジェクトで出来ないかと考えていたところ，普段スプレッドシートで記録している家計簿のデータを使って，データエンジニアリングとMLOpsの技術検証をしようと言う考えに至りました．

技術スタックとしては，普段BigQuery以外は業務でもあまり触れる機会がない[Google Cloud Platform](https://cloud.google.com/) (GCP) のサービスとOSSを組み合わせてできることをしようと考えています．

## プロジェクト名: teamaya
プロジェクトを始めるにあたりプロジェクト名を決めるとワクワクするので，最初に決めることにしました．沖縄好きなので，沖縄の言葉を組み合わせて何か名前を付けたいと考えていて，沖縄県と東京都のハーフである妻にも相談しながら今回の「**<span class="marker_yellow">teamaya</span>**」と言う名前にしました．

githubのリポジトリは以下になります．（スター貰えると泣いて喜びます&#x1f602;）

<iframe class="hatenablogcard" style="width:100%;height:155px;margin:15px 0;max-width:680px;" title="masatakashiwagi/teamaya: This is a repository for Data integration and Machine Learning Pipeline." src="https://hatenablog-parts.com/embed?url=https://github.com/masatakashiwagi/teamaya" frameborder="0" scrolling="no"></iframe>

**teamaya**は"**team**"と"**maya**"を組み合わせた造語です．teamは「〔動物に引かせて～を〕運ぶ、運搬する」という意味があり，maya（マヤー）は沖縄の方言で猫という意味があるので，<span class="marker_yellow">猫にデータを運んで貰う</span>という意味を込めてこの名前にしました．

ちなみに，リポジトリにあるアイコンのハイビスカス&#x1f33a;を付けた猫&#x1f638;は妻に書いて貰いました！

## どういったプロジェクトなのか
今のところ考えているのは，大きく2つになります．
1. **<span class="marker_yellow">データ連携</span>**
    1. ローカルのデータをクラウド上に連携
    2. 連携したデータをELTツールで変換 & データの品質管理
    3. データの可視化
2. **<span class="marker_yellow">機械学習システムの構築</span>**
    1. パイプライン構築
    2. 実験管理
    3. モニタリング & 通知

まず，データ連携については．ローカルからクラウドへの連携，特に特定のデータソースからBigQueryに連携する部分を実施して，データウェアハウス（DWH）を作ろうと思っています．そこからELTツールでデータマート（DM）を作成し，BIツール（Data Studioなど）でデータの可視化を行うといった流れを検証しようと考えています．

個人的には，[Dataform](https://dataform.co/)や[dbt](https://www.getdbt.com/)といったサービスでELTパイプラインを作って，データ変換やデータの品質管理を一通り試してみたいと思っています．

機械学習システムの構築に関しては，機械学習モデル作成のためのパイプラインや実験管理などを行いつつ，コードのテストやデータ・予測結果のモニタリングなどをできるようにしたいのと，[Feast](https://feast.dev/)などのfeature storeを試したりもしたいなと思っています．

現状では上記2つを構築していきたいと思ってますが，進めていく中で興味が湧いたものを適宜取り入れていきたいなと思っています．

## おわりに
今回はプロジェクトを新しく始めたので，その紹介になります．ローカルからクラウドにデータ連携する部分については，既に作成しているので別途内容を紹介したいと思います．

試行錯誤しながら面白そうなものは色々と取り入れて進めて行きたいと思うので，もし面白いツールなどがあればTwitterで教えて頂ければ嬉しいです．

## 参考
- [MLOps.toys](https://mlops.toys/)