+++
author = "Masataka Kashiwagi"
title = "Kaggle-Shopeeコンペの振り返りとソリューション"
date = 2021-05-11T22:11:08+09:00
description = "コンペの振り返りとソリューション"
tags = [
    "kaggle",
]
showLicense = false
+++

## Kaggle-Shopeeコンペの振り返り
2021/03/09~2021/05/11まで開催していた[Shopeeコンペ](https://www.kaggle.com/c/shopee-product-matching)の振り返りになります．

2週間程度しか手を動かせなかったですが，久しぶりに参加したので備忘録として記録を残しておきます．

最終的な結果は179th/2464で銅メダルで，特に凝ったことは何もしていなかったので，妥当かなと思います．

このコンペは上位10チーム中7チームが日本人チームで，日本人のレベルの高さを改めて実感できるコンペでした！

## 概要
コンペの内容は簡単に言うと，画像とテキスト情報を用いて、2つの画像の類似性を比較し，どのアイテムが同じ商品であるかを予測するコンペになります．

- 開催期間: 2021/03/09 ~ 2021/05/11
- 参加チーム数: 2464
- 予測対象: `posting_id`列にマッチする全てのposting_idを予測する．ただし，posting_idは必ずself-matchし，グループの上限は50個となっている．
- データ: 投稿ID，画像，商品のタイトル，画像の知覚ハッシュ(perceptual hash)，ラベルグループID
- 評価指標: F1-Score
- その他: コードコンペ

## My Solution
![My Solution](../../img/kaggle-shopee-my_solution.png "my solution")

- 画像特徴量・テキスト特徴量・画像のphash値をconcatして結果をユニーク化したものを最終的な予測値としました．
- 何も複雑なことはしていないモデルですが，結果的に銅メダルを取ることができました．

### Image Model
- eca_nfnet_l0
    - loss: ArcFace
    - pooling: AdaptiveAvgPool2d
    - scheduler: CosineAnnealingLR
    - loss(criterion): CrossEntropyLoss
    - size: 512*512
- eca_nfnet_l1
    - loss: CurricularFace
    - pooling: MAC
    - scheduler: CosineAnnealingLR
    - loss(criterion): FocalLoss
    - size: 512*512
- efficientnet_b3
    - loss: ArcFace
    - pooling: GeM
    - scheduler: CosineAnnealingLR
    - loss(criterion): FocalLoss
    - size: 512*512
- swin_small_patch4_window7_224
    - loss: CurricularFace
    - scheduler: CosineAnnealingLR
    - loss(criterion): FocalLoss
    - size: 224*224
- common
    - augmentation by albumentations.
        - HorizontalFlip
        - VerticalFlip
        - Rotate
        - RandomBrightness
    - optimizer
        - Adam

#### 少し工夫したポイント
- 画像特徴量を抽出する部分で，いくつか工夫した点をあげます．  
    - [CNN Image Retrieval](http://cmp.felk.cvut.cz/cnnimageretrieval/)を参考にしました．
1. lossに`ArcFace`と`CurricularFace`を用いた
    - ArcFaceを使っている人が多かったが，CurricularFaceも使いました．スコア的にはCurricularFaceの方がよかったです．
2. いくつかのモデルでpooling層に`GeM`と`MAC`を用いた
    - Google Landmark Retrieval Challengeで上手くいっていたGeMやMACなどのプーリング手法を用いました．
3. loss(criterion)に`FocalLoss`を用いた
4. 最終的に得られた特徴量をconcatした後，`ZCAWhitening`による次元削減を行った

#### 有効でなかったもの
- 一方で，上手くいかなかった内容としては以下になります．
    - ResNeXt-50 32x4d
    - swin_small_patch4_window7_224 with ArcFace
    - CosFace, AdaCos
    - PCA Whitening (worse than ZCAWhitening)

### Text Model
- paraphrase-xlm-r-multilingual-v1
    - loss: ArcFace
    - scheduler: linear schedule with warmup
    - loss(criterion): CrossEntropyLoss
    - optimizer: AdamW
- TF-IDF

textの方のモデルは特に改良する時間が取れなかったので，ほとんど手を付けれてなかったです．
- transformerとTF-IDFで得られた特徴量それぞれに対して，Cosine Similarityを計算し，テキストのpredictionを作成しました．

最終的な予測値は画像特徴量とテキスト特徴量に加えて，画像のphash値を追加して，ユニークを取った値としています．

### 反省
- post-processingが全然できていない
    - 他の解法を見るに，post-processingでスコアが伸びているので，この部分は結構大事だったんだなと感じています．
- ImageとTextをMulti-modal的にモデルに組み込んで学習することができていなかった
- グラフ理論全然わかってないです笑
- 細かい部分
    - optimizerの変更（SAMとか）
    - DBA / QE
    - 閾値の調整
    - テキストモデルの追加


---
ここからは上位の解法を書きたいと思います．数もそれなりにあるので，載せるのは上位5つにします．最後に上位5つ以外にもDiscussionsに投稿されている解法のリンクを載せておきます．

## 1st Place Solution
解法はこちらになります: [1st Place Solution - From Embeddings to Matches](https://www.kaggle.com/c/shopee-product-matching/discussion/238136)

### Model
- Image: 2つのモデル
    - `eca_nfnet_l1s` * 2
- Text: 5つのモデル
    - `xlm-roberta-large`
    - `xlm-roberta-base`
    - `cahya/bert-base-indonesian-1.5G`
    - `indobenchmark/indobert-large-p1`
    - `bert-base-multilingual-uncased`
- loss: ArcFace
- poolingした後にbatch normalizationとfeature-wise normalizationを行った
- ArcFaceのチューニングを行った
    - 学習段階で徐々にmarginを大きくした（埋め込み表現のqualityに影響する）
        - 画像に対するmargin: 0.8~1.0
        - テキストに対するmargin: 0.6~0.8
    - marginを大きくすると，モデルの収束に問題が出たので，以下を行った
        - warmup stepsを大きくする
        - cosineheadに対するlearning rateをより大きくする
        - gradient clippingを行う
    - class-size-adaptive marginも試したが，不均衡性が小さかったため，改善は少しだけだった
        - image: `class_size^-0.1`
        - text: `class_size^-0.2`
- global average poolingの後にFC層を追加するとモデルが悪くなるが，feature-wise normalizationの前にbatch normalizationを追加するとスコアが改善した

![Model](../../img/kaggle-shopee-1st_solution_model.png "1st solution model")

### Features
- Combining Image & Text Matches
    - マッチングさせる方法をいくつかトライした
    1. 画像のembeddingによるマッチングとテキストのembeddingによるマッチングを結合する
    2. 画像のembeddingとテキストのembeddingをconcatしてからマッチングする
    3. 1と2を組み合わせてマッチングする
    - これは，画像のembeddingが強く示唆するアイテム・テキストのembeddingが強く示唆するアイテム・画像とテキストのembeddingがどちらも適度に示唆するアイテムを取り入れることができる

![Features](../../img/kaggle-shopee-comb_match.png "comb match")

- Iterative Neighborhood Blending (INB)
    - QE/DBAとは少し異なる，embeddingからマッチするプロダクトを検索するパイプラインを作った
    - 近隣探索ラリブラリ: [faiss](https://github.com/facebookresearch/faiss)
    - コサイン類似度をコサイン距離(=1-コサイン類似度)に変換し，distance<thresholdを満たすペアを取得した


## 2nd Place Solution
解法はこちらになります: [2nd place solution (matching prediction by GAT & LGB)](https://www.kaggle.com/c/shopee-product-matching/discussion/238022), [2nd Place Solution Code](https://www.kaggle.com/c/shopee-product-matching/discussion/238362)

### Summary
- 1st stage: 画像・テキスト・画像+テキストデータに対するコサイン類似度を得るためにMetric Learningによるモデルを学習する
- 2nd stage: 同じlabel groupに属するアイテムのペアかどうかを識別するために"メタ"モデルを学習する

### Model
- image: 2つのモデル
    - backbone
        - `nfnet-F0`
        - `ViT`
    - loss: CurricularFace
    - optimizer: SAM
    - embeddingの結合: `F.normalize(torch.cat([F.normalize(emb1), F.normalize(emb2)], axis=1))`
- text: 3つのモデル + TF-IDF
    - `Indonesian-BERT`
    - `Multilingual-BERT`
    - `Paraphrase-XLM`
- image+text: `NFNet-F0`と`Indonesian BERT`のFC層のembeddingを結合したもの
- Training Tips:
    - label groupのサイズに基づいたSample Weighting
        - 1 / (label group size) ** 0.4

### Features
#### Graph features
- それぞれのアイテムのtop-Kコサイン類似度の平均と分散
    - K=5, 10, 15, 30, etc
- 標準化（mean=0, std=1）
- Pagerank

#### その他
- テキストの長さ
- #の記号
- Levenshtein距離
- 画像ファイルのサイズ
- 画像の高さと幅
- Query Extension

### Ensemble
#### Methodology
- ローカルデータでそれぞれのモデルのベストな閾値を計算する
- 上記閾値でそれぞれのモデルの予測値を差し引く
- 差し引かれた予測値を合計する
- 合計値>0の場合，アイテムペアが同じグループに属するとする
- お互いにエッジを持たないペアを削除する
    - A-Bはあるが，B-Aがない場合は両方を削除する

### Post-processing
- 中間中心性が最も高いエッジを再帰的に削除する（Graph-Based）

### その他
- Performance and Memory Tunings
    - CuDF, cupy, cugraph: GPUを有効に使うためには大事
    - ForestInference: 40分かかるCPUでの推論が2分になる
- Not worked
    - Graph-basedの特徴量
    - 固有ベクトルの中心性とjaccard
    - End-to-end model
    - Local feature matching

## 4th Place Solution
解法はこちらになります: [4th Place Solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238295)

### Model
- image: backboneに`nfnet`と`efficientnet`
    - pooling: GeM＆Avg pooling
    - embeddingが大きいほど，LBのスコアがよくなった
    - loss: ArcFace
    - marginの調整が大事だった
- text: BERTベース + TF-IDF
    - TF-IDF: かなり大きいembeddingを生成し，notebook上ではメモリの制約を受けるため，Random Sparse Projectionで次元削減を行った
    - loss: ArcFace
    - marginの調整が大事だった

![Model](../../img/kaggle-shopee-4th_solution_model.png "4th solution model")

### Ensemble
- 画像のembedding・テキスト（BERTベース）のembedding・TF-IDFのembeddingを結合して，正規化した
- これらのベクトル表現のそれぞれに対して，pairwiseコサイン類似度を計算し，3つの行列を作成した
    - 最初に二乗し，その後加重平均を取って行列を結合した

### Post-processing
- 閾値の調整
- Rank2 matching
    - もし，AがRank2にBを持っており，BはRank2にAを持っている場合，お互いに追加する
- Rank2とRank3の違いが大きい場合，Rank2のIDを追加する
- 少なくとも1つの他とのマッチング（Rank2のコサイン類似度が極端に小さくなければ）
- Query Expansion
- マッチしていない行との再マッチング

## 5th Place Solution
解法はこちらになります: [5th Place Solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238078)



## その他上位解法のリンク
- [6th place solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238010)
- [7th place solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238174)
- [8th Place Solution Overview](https://www.kaggle.com/c/shopee-product-matching/discussion/238125)
- [Public 16th / Private 10th Solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238039)
- [11th Place Solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238181)
- [14th Place Gold - Image Text Decision Boundary](https://www.kaggle.com/c/shopee-product-matching/discussion/238033)
- [15th place solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238029)
- [Public 13th / Private 16th solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238128)
- [18th place solution - You really don't need big models](https://www.kaggle.com/c/shopee-product-matching/discussion/237972)
- [19 place. Voting and similarity chain](https://www.kaggle.com/c/shopee-product-matching/discussion/238146)
- [26th Place Solution : Effective Cluster Separation and Neighbour Search](https://www.kaggle.com/c/shopee-product-matching/discussion/238126)
- [[31st Place] Object Detection approach](https://www.kaggle.com/c/shopee-product-matching/discussion/238322)
- [Public 39th / Private 37th Solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238110)
- [48th Place Silver - Simple Baseline](https://www.kaggle.com/c/shopee-product-matching/discussion/238263)
- [Public 56th / Private 57th Solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238079)
- [62th Place Solution (Stacking Logistic Regression)](https://www.kaggle.com/c/shopee-product-matching/discussion/238149)
- [72nd place solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238167)