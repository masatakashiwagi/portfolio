+++
author = "Masataka Kashiwagi"
title = "Kaggle-Shopee コンペの振り返りとソリューション"
description = "コンペの振り返りとソリューション"
date = 2021-05-14T15:44:00+09:00
draft = false
share = true
showLicense = false
support = true
tags = ["KAGGLE"]
+++

## Kaggle-Shopee コンペの振り返り

2021/03/09~2021/05/11まで開催していた[Shopee コンペ](https://www.kaggle.com/c/shopee-product-matching)の振り返りになります．

2週間程度しか手を動かせなかったですが，久しぶりに参加したので備忘録として記録を残しておきます．最終的な結果は179th/2464で銅メダルで，特に凝ったことは何もしていなかったので，妥当かなと思います．このコンペは上位10チーム中7チームが日本人チームで，日本人のレベルの高さを改めて実感できるコンペでした！

## 概要

コンペの内容は簡単に言うと，画像とテキスト情報を用いて、2つの画像の類似性を比較し，どのアイテムが同じ商品であるかを予測するコンペになります．

- 開催期間: 2021/03/09 ~ 2021/05/11
- 参加チーム数: 2464
- 予測対象: `posting_id` 列にマッチする全ての posting_id を予測する．ただし，posting_id は必ず self-match し，グループの上限は50個となっている．
- データ: 投稿ID，画像，商品のタイトル，画像の知覚ハッシュ (perceptual hash)，ラベルグループID
- 評価指標: F1-Score
- その他: コードコンペ

## My Solution

![My Solution](../../img/kaggle-shopee-my_solution.png "my solution")

- 画像特徴量・テキスト特徴量・画像の phash 値を concat して結果をユニーク化したものを最終的な予測値としました．
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

画像特徴量を抽出する部分で，いくつか工夫した点をあげます（[CNN Image Retrieval](http://cmp.felk.cvut.cz/cnnimageretrieval/) を参考にしました）．

1. loss に `ArcFace` と `CurricularFace` を用いた
    - ArcFace を使っている人が多かったが，CurricularFace も使いました．スコア的にはCurricularFace の方がよかったです．
2. いくつかのモデルで pooling 層に `GeM` と `MAC` を用いた
    - Google Landmark Retrieval Challenge で上手くいっていた GeM や MAC などのプーリング手法を用いました．
3. loss (criterion) に `FocalLoss` を用いた
4. 最終的に得られた特徴量を concat した後，`ZCAWhitening` による次元削減を行った

#### 有効でなかったもの

- 一方で，上手くいかなかった内容としては以下になります．
  - resnext50_32x4d
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

text の方のモデルは特に改良する時間が取れなかったので，ほとんど手を付けれてなかったです．
transformer と TF-IDF で得られた特徴量それぞれに対して，Cosine Similarity を計算し，text の prediction を作成しました．

最終的な予測値は画像特徴量とテキスト特徴量に加えて，画像の phash 値を追加して，ユニークを取った値としています．

### 反省

- post-processing が全然できていなかった
  - 他の解法を見るに，post-processing でスコアが伸びているので，この部分は結構大事だったんだなと感じています．
  - 6位の解法にもありましたが，今回のコンペでは，label_group の長さが2以上であることから，「予測した結果の posting_id が1つしかない場合，強制的に似たものを持ってきて，2つにする」というアイデアで LB がかなり上がるみたい
- Image と Text を Multi-modal 的にモデルに組み込んで学習することができていなかった
- グラフ理論全然わかってないです笑
- 細かい部分
  - 他の optimizer を試す
    - SAM とか
  - Database-side feature augmentation (DBA) / Query Extension (QE)
    - 全然知らなかったので，[End-to-end Learning of Deep Visual Representations for Image Retrieval](https://arxiv.org/pdf/1610.07940.pdf) を読んで勉強したい
  - 閾値の調整
  - テキストモデルの追加

---
ここからは上位の解法を書きたいと思います．数もそれなりにあるので，載せるのは上位5つにします．最後に上位5つ以外にも Discussions に投稿されている解法のリンクを載せておきます．

## 1st Place Solution

解法はこちらになります: [1st Place Solution - From Embeddings to Matches](https://www.kaggle.com/c/shopee-product-matching/discussion/238136)

### Model

- Image: 2つのモデル
  - `eca_nfnet_l1` * 2
- Text: 5つのモデル
  - `xlm-roberta-large`
  - `xlm-roberta-base`
  - `cahya/bert-base-indonesian-1.5G`
  - `indobenchmark/indobert-large-p1`
  - `bert-base-multilingual-uncased`
- loss: ArcFace
- pooling した後に batch normalization と feature-wise normalization を行った
- ArcFace のチューニングを行った
  - 学習段階で徐々に margin を大きくした（埋め込み表現の quality に影響する）
    - 画像に対する margin: 0.8~1.0
    - テキストに対する margin: 0.6~0.8
  - margin を大きくすると，モデルの収束に問題が出たので，以下を行った
    - warmup steps を大きくする
    - cosinehead に対する learning rate をより大きくする
    - gradient clipping を行う
  - class-size-adaptive margin も試したが，不均衡性が小さかったため，改善は少しだけだった
    - image: `class_size^-0.1`
    - text: `class_size^-0.2`
- global average pooling の後に FC 層を追加するとモデルが悪くなるが，feature-wise normalization の前に batch normalization を追加するとスコアが改善した

![Model](../../img/kaggle-shopee-1st_solution_model.png "1st solution model")

### Features

- Combining Image & Text Matches
マッチングさせる方法をいくつかトライした
  1. 画像の embedding によるマッチングとテキストの embedding によるマッチングを結合する
  2. 画像の embedding とテキストの embedding を concat してからマッチングする
  3. 1と2を組み合わせてマッチングする
  これは，画像の embedding が強く示唆するアイテム・テキストの embedding が強く示唆するアイテム・画像とテキストの embedding がどちらも適度に示唆するアイテムを取り入れることができる

![Features](../../img/kaggle-shopee-comb_match.png "comb match")

- Iterative Neighborhood Blending (INB)
  - QE/DBA とは少し異なる，embedding からマッチする商品を検索するパイプラインを作った
  - k-Nearest Neighbor Search:
    - 近隣探索ラリブラリ: [faiss](https://github.com/facebookresearch/faiss) (k=51: 50(自身以外)) + 1(自身))
  - Threshold:
    - コサイン類似度をコサイン距離(=1-コサイン類似度)に変換し，distance < threshold を満たす `(matches, distances)` のペアを取得した
    - 閾値処理を行う場合，1つのクエリに対して少なくとも2つ以上のマッチがあることがコンペで保証されているので，distance が min2-threshold を超えた場合にのみ，二番目に近いマッチを除外する
  - Neighborhood Blending:
    - kNN によるサーチと min2 による閾値処理をした後，各アイテムの `(matches, similarities)` ペアを取得し，グラフを作成する
      - 各ノードはアイテム，エッジの重みは2つのノード間の類似性を示す
      - 近傍のみが繋がっており，閾値条件と min2 条件を満たさないノードは切断される
    - 近傍のアイテムの情報を使いたいので，クエリアイテムの embedding を改良し，よりクラスタを明確にする
      - 重みとして類似性を持つ近傍の embedding を加重合計し，それをクエリ embedding に追加する
    ![Neighborhood Blending](../../img/kaggle-shopee-NB.png "neighborhood blending")
    - 上図を簡単に説明すると，最初 A は [B,C,D] と繋がっているが，加重合計した結果閾値=0.5の場合，C との接続が切れて，A は [B,D] のみと接続していることがわかる
    - この NB の処理を評価指標の改善が止まるまで繰り返し実行する
    - 最終的な NB のパイプラインは以下になる
    ![Neighborhood Blending pipeline](../../img/kaggle-shopee-NB_pipeline.png "neighborhood blending pipeline")
  - About Threhsold Tuning:
    - 調整する閾値は全部で10種類ある
      - stage1 の text, image, combination の閾値が3つ
      - stage2 の閾値が1つ
      - stage3 の閾値が1つ
      - 直接最終結合部分に繋がる stage1 の text, image の閾値が2つ
      - stage1~3 の min2 閾値が3つ
    - 最終的には stage2,3 の閾値を2つ調整するだけでよかった
- Visualizations of Embeddings before/after INB
  - INB の効果を可視化したノートブックが公開されている
    - [(SHOPEE) Embedding Visualizations before/after INB](https://www.kaggle.com/harangdev/shopee-embedding-visualizations-before-after-inb)
    - 見るからに各点が凝集されたクラスタを形成していることがわかる

### Others

- 画像に対する CutMix(p=0.1)
- 画像の augmentation
  - horizontal flip のみがよかった
- madgrad optimizer: https://github.com/facebookresearch/madgrad
- 学習データ全体を使った学習

## 2nd Place Solution

解法はこちらになります: [2nd place solution (matching prediction by GAT & LGB)](https://www.kaggle.com/c/shopee-product-matching/discussion/238022), [2nd Place Solution Code](https://www.kaggle.com/c/shopee-product-matching/discussion/238362)

### Summary

- 1st stage: 画像・テキスト・画像+テキストデータに対するコサイン類似度を得るために Metric Learning によるモデルを学習する
- 2nd stage: 同じ label group に属するアイテムのペアかどうかを識別するために"メタ"モデルを学習する

### Model

- image: 2つのモデル
  - backbone
    - `nfnet-F0`
    - `ViT`
  - loss: CurricularFace
  - optimizer: SAM
  - embedding の結合: `F.normalize(torch.cat([F.normalize(emb1), F.normalize(emb2)], axis=1))`
- text: 3つのモデル + TF-IDF
  - `indonesian-bert`
  - `multilingual-bert`
  - `paraphrase-xlm`
- image+text: `nfnet-F0` と `indonesian-bert` の FC 層の embedding を結合したもの
- Training Tips:
  - label group のサイズに基づいた Sample Weighting
    - 1 / (label group size) ** 0.4

### Features

#### Graph features

- それぞれのアイテムの top-K コサイン類似度の平均と分散
  - K=5, 10, 15, 30, etc
- 標準化（mean=0, std=1）
- Pagerank

#### Others

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
  - A-B はあるが，B-A がない場合は両方を削除する

### Post-Processing

- 中間中心性が最も高いエッジを再帰的に削除する（Graph-Based）

### Others

- Performance and Memory Tunings
  - CuDF, cupy, cugraph: GPU を有効に使うためには大事
  - ForestInference: 40分かかる CPU での推論が2分になる
- Not worked
  - Graph-based の特徴量
  - 固有ベクトルの中心性と jaccard
  - End-to-end model
  - Local feature matching

## 3rd Place Solution

解法はこちらになります: [3rd Place Solution (Triplet loss, Boosting, Clustering)](https://www.kaggle.com/c/shopee-product-matching/discussion/238515)

### Model

- image: 2つのモデル
  - backbone
    - `efficientnet_b2`
    - `ViT (DINO)`
- text: 2つのモデル (異なるtokenizersとCLIP)
  - `indonesian-bert`
  - `multilingual-bert`
- common
  - loss: TripletLoss (margin=0.1)
  - 各エポック，label_groups から重複したペアを作成し，各バッチで label_groups から1ペアが挿入される．バッチサイズは128を使っていたため，バッチ毎に64の重複を取得し，ランダムな5点でそれらの各々を比較する
  - 5fold CV で，validation スコアを計算する時は，validation-fold の中から候補を選ぶのではなく，学習サンプル全体から候補を探した
    - この方法は CV/LB の gap を抑えて，相関を取ることが出来る
    - oof を用意して，2nd-level の予測に使用する
    - CV から得られた5つのモデルで，テストデータに対して推論を行った
    - より速く推論するためには，validation なしの1つのモデルでテストデータにfitさせること
  - ArcFace は上手くいかなかった

### Features

- 複数モデルを用いて，異なる表現方法を作成するのが良い
- 全ての embeddings から候補となる近しいものを結合し，ペアが実際に重複しているかどうかにかかわらず，binary target を使用してペアオブジェクトのサンプルを作成する
  - 3M点のペアがあり，重複率は4%だった
- これらに対して，GBM(=CatBoost) を作成し，embedding 毎に計算する
  - pairwise-distances (コサイン類似度，ユークリッド距離など)
  - 両方の点周辺の密度
  - points ranks
- 最終的には500個特徴量を作成し，CV や LB を使いながら，重複確率による閾値の候補を出す → LB:0.76+

### Post-Processing

- クラスタリングを行ったが，embeddings を使うのではなく，pairwise-distance を使う
- 重複推定のために GBM(=CatBoost) の確率を使い，類似度を求める
- 凝集クラスタリングのアイデアを採用
  - 各単一点をクラスタとして開始し，平均的なクラスタサイズが閾値に等しくなるまでそれらをマージしていく
  - クラスタが単一の場合，最も近傍のクラスタにマージする → LB:0.79

### Others

- 最適化するために
  - `half precision(torch AMP)` を使って学習と推論を行った
  - 画像モデルの場合，画像の読み込みとリサイズに `NVIDIA DALI` を使った
  - GBM の推論には，Rapids ForestInference Library を使った
- 上記の方法を使わないと，2時間で全てのモデルを推論することは不可能だった

## 4th Place Solution

解法はこちらになります: [4th Place Solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238295)

### Model

- image: backbone に `nfnet` と `efficientnet`
  - pooling: GeM＆Avg pooling
  - embedding が大きいほど，LB のスコアがよくなった
  - loss: ArcFace
  - margin の調整が大事だった
- text: BERTベース + TF-IDF
  - TF-IDF: かなり大きい embedding を生成し，notebook 上ではメモリの制約を受けるため，Random Sparse Projection で次元削減を行った
  - loss: ArcFace
  - margin の調整が大事だった

![Model](../../img/kaggle-shopee-4th_solution_model.png "4th solution model")

### Ensemble

- 画像の embedding・テキスト（BERTベース）の embedding・TF-IDF の embedding を結合して，正規化した
- これらのベクトル表現のそれぞれに対して，pairwise コサイン類似度を計算し，3つの行列を作成した
  - 最初に二乗し，その後加重平均を取って行列を結合した

### Post-Processing

- 閾値の調整
- Rank2 matching
  - もし，A が Rank2 に B を持っており，B は Rank2 に A を持っている場合，お互いに追加する
- Rank2 と Rank3 の違いが大きい場合，Rank2 の ID を追加する
- 少なくとも1つの他とのマッチング（Rank2 のコサイン類似度が極端に小さくなければ）
- Query Expansion
- マッチしていない行との再マッチング

## その他上位解法のリンク

- [Kaggle ShopeeコンペPrivate LB待機枠＆プチ反省会](https://youtu.be/vtI8P-ttPrk)
- [5th Place Solution](https://www.kaggle.com/c/shopee-product-matching/discussion/238078)
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
