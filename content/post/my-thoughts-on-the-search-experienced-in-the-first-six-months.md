+++
author = "Masataka Kashiwagi"
title = "検索に対する感想 - 検索基盤から検索エンジンの改善を始めて半年経て"
description = "初めて検索という領域に触れたので，その感想を書きました"
date = 2022-12-20T00:01:02+09:00
draft = false
share = true
showLicense = false
support = true
tags = ["POEM"]
+++

## はじめに

最近，スタバで期間限定で販売しているの冬の新作「[バターキャラメルミルフィーユ ラテ](https://menu.starbucks.co.jp/4524785522695)」のミルクをアーモンドミルクに変更して飲んだら激ウマでハマってしまいました笑．皆さんも是非飲んでみて下さい！（ちなみに寒いので，ホットで注文しました😋）

今回のポストは <span class="marker_yellow">**[情報検索・検索技術 Advent Calendar 2022](https://adventar.org/calendars/7389)**</span> の20日目の記事になります．

どんな内容の記事を書こうか悩みましたが，今回は技術的なものではなく，僕が今年の5月半ばぐらいから関わり出した「**検索**」に関しての取り組みやその感想を書こうと思います．具体的には，検索基盤の整備〜検索エンジンのチューニング〜輪読会などを通して感じたことを書きたいと思います．

モチベーション的には，後から見返した時に当時の検索に対する初々しい気持ちはどんなものだったのかを残しておきたい気持ちです笑

## 検索する上で欠かせない検索基盤

何と言っても検索エンジンにデータを連携し供給しないと検索は何もできないので，まずは検索基盤を整備するのが必要です．検索という行為は普段から Google 先生に聞いたり，各種Webサイト内で検索をしたりと何気なしに使っていますが，新しいデータを追加して検索できるようにするためには，検索エンジンのインデックスにデータを追加する必要があるというのを正直この時初めて知りました笑（最初はインデックスという単語をよく理解していなかったです笑）．

※ 検索エンジンを使わずに，データベースに検索クエリを投げているケースもあると思いますが，ここでは，検索エンジンを使っているとします．

検索エンジンにデータを定常的に連携するために検索基盤を用意するわけですが，検索基盤の具体的な構成とかの話は，僕が以前登壇したこちらの資料「**[JAWS DAYS 2022 - AWS のマネージドサービスで実現するニアリアルタイムな検索基盤](https://speakerdeck.com/masatakashiwagi/jaws-days-2022-awsnomanezidosabisudeshi-xian-suruniariarutaimunajian-suo-ji-pan)**」を見て頂けるとありがたいです．

構築段階よりはどう日々の運用と上手く付き合っていくか，「日々運用をしていく上で，何に気をつけるべきなのか？どういったことを意識すると良いのか？」を考えると，そういった話は調べてもなかなか出てこないし，環境によって違うしという感じで悩ましいなという気持ちです．が，僕は以下の2つを意識しながら，設計して作るようにしました．

- **ユーザーへの影響を考える**
  - ユーザー視点で見ると，検索機能が使えなくなることは，そのサービスへの期待値を下げてしまうことになりますし，欲しい・知りたい情報を見つけられなくなるため，影響が大きいです．そのため，アラートの粒度やエラー時のリカバリーをどうするか，今回の検索基盤だと2つのインデックスを運用していたので，切り替え時にエラーが出てもそのまま動き続けれるように，削除タイミングやどのタイミングで切り替えるのが良いのか，などは考慮しながら作るようにしていました．
- **システムを運用していくチームメンバーのことも考える**
  - 基盤を作った僕以外の人が触る時に，簡単に処理を修正できるか？エラーが起きた場合のリカバリーが簡単にできるか？といった部分は，構築後の運用において属人化を避けるためにも意識したいところでした．
  - ドキュメントをきちんと用意することもそうですし，処理をモジュール単位で作る（Step Functions だとこの辺りは整理しやすいです）とかは意識していました．

より大きなシステムを運用するとなると，また違った要素も必要なのかも知れないです．

**システムは作った後の日々の運用をどうしやすくするかが大事**だと思うので，この点は今後も意識したいと思っています．（なんか，検索システムに限った話ではない感じになってしまいましたね😅）

検索基盤の構築で個人的にこれは良かったなと思った取り組みは，インデックスの切り替えで，Blue/Green deployment をパイプライン（AWS Step Functions）の中で行うようにすることで，安全に切り替わる方式を作れたのは，ユーザーへの影響を考えると良い取り組みだったかなと思います．他の会社さんはどういった感じで運用されているのかとても気になっているので，是非教えて下さい笑

なので，検索基盤が安定&信頼できるものでないと，ユーザー影響もそうですし，検索エンジンの精度改善だったりもおぼつかなくなるので，検索基盤大事ですね！

## みんなで検索の書籍を読んでワイワイ話す

チーム内の検索システムに対する共通認識や目線を合わせるために，「[検索システム - 実務者のための開発改善ガイドブック（通称ペンギン本）](https://www.lambdanote.com/products/ir-system)」の輪読会を僕が主催したりもしました．（チーム内でも個々人で検索システムに対しての理解のバラツキがあったので，理解を深めて議論が活発になると良いなと思い始めました．あとは，単純にこの書籍がとても面白そうだったので，みんなで読みたいと思い始めた感じです笑）

この書籍は名の通りで，実務向けに書かれていてとても参考になることが多かったです．特に後半の第II部にその要素が詰め込まれていて，

- 検索システムの運用の話
  - 上で話した Blue/Green deployment の話やオフライン評価，障害検知など実務的に考えていく必要があることが載っていて，どういったことを運用時には考えて用意しておく必要があるのかなど参考になります
- 分析の観点の話
  - 検索クエリだけでなく，その結果がどうだったかなどを考えるべきといったことが書かれています
- 改善箇所の発見プロセス
  - Why-What-Who を定めないと意味のない改善になってしまういった心に刺さることが書かれています
- 機械学習を使ったランキングの話
  - あまり体系的に整理されたものはないと思うので，非常にありがたいことが書かれています
- etc...

これ以外にも色々とありすぎて全部はここに書けないですが，検索をする人は一度は目を通しておくべき内容だなと思いました！（定期的に読み直したいです）

輪読会を通して，検索システムに対する理解やそれが及ぶ範囲などを知ることができたのと，それをチームメンバーでうちのサービスだとどうか？など自分達の状況に置き換えながら議論して，内容を共有しながら進めることができたので，とても学びがある会になったと思います．

ただ，準備に時間がかかったり，読み込みの時間が十分でなく議論がそこまで活発化しなかった会もあったので，そういった場合は会の最初に黙々と読む時間を用意しても良かったのかもと思ったりもしました．

検索システムは奥が深くて知れば知るほど関わる人も多く一筋縄ではいかないなと思う一方で，取り組み甲斐のある面白い領域だなと感じてます😊

## 検索エンジンのチューニングって難しいな

メインは検索基盤周りを扱っていて，今の所チューニング部分は少しだけ関わっていますが，

- Analyzer の定義
- Query DSL の定義
- 辞書整備
- etc...

上に挙げたあたりで難しいなと日々感じてます...

最初の方はゼロ件ヒットの結果や検索ログに基づいたユーザー辞書・同義語辞書の整備は色々とやっていて，特に同義語辞書は何を同義語と設定するか，双方向なのか一方向の辞書展開なのかは答えがないので，言葉の意味や定義を調べたり，ユーザーの一人である妻に聞いたりしながら設定していました．（同義語の収集自動化したい...）

また，検索ログを眺めていると，ユーザーが何を調べているか，ワードもそうですし全体でのそのワード回数などを感覚的に掴めるので，一度やったのは正解でした！

他には，こんな感じで検索ワード入れると検索にヒットするのになーと思ったりして，ユーザー全員が検索の仕方が上手いわけではないので，サジェストで「これのことですか？」とか，意味は同じで「こちらの方がヒット数が多いですよ」とか，もう少しガイドがあったりすると改善できるのかな？と感じたり．

あとは内部的に表記揺れを吸収して直したりとかはありそうだなと感じたりはしています．すぐに技術的に解決が難しくても，UI でもそういったガイドはできると思っていて，この辺りはエンジニアリングとデザインが上手く混ざり合うと良い体験をユーザーに提供できるのかなと感じています．

機械学習の導入に関しては，検索結果のパーソナライズ化できる利点もありますが，それを動かす環境だったりレイテンシーを抑えて計算して結果を表示する工夫が必要だと思うので，そんなに簡単には導入でき無さそうかなと思ったりしてます🤔

## おわりに

半年程度検索という領域に携わって，手探りながら進めて感じたのは，

- 検索ログを泥臭く見て改善ポイントを見つけたり，ユーザーの行動に思いを馳せる
- ゼロ件ヒットワードや検索利用率などの各種メトリクスを取って可視化できるようにする
- ロジック変更したら，定性チェックして検索結果に違和感がないか見る
- エラー検知はもちろんだけど，エラー時の Runbook を用意する
- リカバリーする手順はなるべく自動化して手動によるオペミスを減らす（インデックスの再作成）

などが大事だなと思ったり（他にもいっぱいあると思います笑），あとは検索で良い体験を提供するためには，単純にアルゴリズムの話だけでなく，

- UI/UX
- 信頼できる検索基盤
- 検索エンジンのチューニング
- 辞書の整備
- 検索のパフォーマンス
- etc...

本当に総合格闘技で取り組まないと成功しないなというの実感しているのと，そもそもユーザーにどういった検索体験を提供したいかを考えないと目指す方向性がブレてしまうので，言語化が必要だなというお気持ちです😅
これからもいっぱい悩んでいこうと思います！
