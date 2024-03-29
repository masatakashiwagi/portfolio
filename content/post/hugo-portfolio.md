+++
author = "Masataka Kashiwagi"
title = "Hugo を使ったポートフォリオ作成"
date = "2020-07-13"
description = "Hugo で簡単にポートフォリオを作成したので，その備忘録を書きました"
draft = false
share = true
showLicense = false
support = true
tags = ["DEV"]
+++

## ポートフォリオ作成

[こちら](https://qiita.com/ysdyt/items/a581277dd1312a0e83c3)の Qiita の記事で Hugo を使って簡単にポートフォリオを作成できるというのを見かけたので，以前まで使っていた personal page を移植しました．
移植した際に少し詰まった部分があるので，Tips としてこの記事で紹介します．

<u>**この記事は以前に使用していた Hugo Theme の内容になります**</u>

最初は [Hugo Theme Cactus Plus](https://github.com/nodejh/hugo-theme-cactus-plus) というテーマで作成していたのですが，再度作り直してます．
（再作成した理由は，少しだけ凝ったテーマを使って見たくなったためです笑）
作り直したテーマは [Hugo Future Imperfect Slim](https://github.com/pacollins/hugo-future-imperfect-slim) になります．

Hugo はシンプルなデザインが多いので非常にオススメです．

基本的な構築方法は上記 Qiita の記事に沿って行っています．
別途追加した要素としては，最初に作成した Hugo Theme Cactus Plus と作り直した Hugo Future Imperfect Slim それぞれありますので，この際どちらも紹介します．

* **Hugo Theme Cactus Plus**
  1. メニューの追加設定
  2. Custom-CSSの設定（custom-cssの設定は簡単に設定できますので，今回は割愛します）

* **Hugo Future Imperfect Slim**
  1. faviconの設定
  2. github.ioでサイトをhostした場合のpath設定

個人的には，1回目に作成したテーマより2回目の方が簡単でした．

### Hugo Theme Cactus Plus

#### メニューの追加設定

Hugo Theme Cactus Plus のテーマでは，デフォルトで About/Archive/Tags の3つがメニューとして存在しています．今回はそこに Projects を新しく追加しましたので，その方法を記載します．
実施することとしては，以下の4ステップになります．

1. `content` 配下に projects ディレクトリを作成し，`_index.md` ファイルを配置する．
記事などのページ情報は `content` で管理します．

    ```bash
    ├── content
    │   ├── about
    │   ├── posts
    │   └── projects
    │       └── _index.md
    ```

2. `themes/layouts/partials` 配下にある `nav.html` に Projects のリンクを追記する．
これは Tags などのリンクをコピーして，name の部分は projects に修正すれば大丈夫です．
メニューバーに Projects を表示させるために，この部分を修正する必要があります．

3. `themes/layouts/section` 配下に `about.html` をコピーして，`projects.html` に rename する．ここに追加することで，セクションのトップページとして扱われることになります．

4. 最後に，コマンドラインで `hugo` を実行する．
`hugo` コマンドを実行することで，必要なものが自動生成・反映されます．

以上でメニューを追加することができます．
（他のテーマでは，もう少し簡単にメニュー追加が可能なものもあります）

### Hugo Future Imperfect Slim

#### favicon の設定

favicon を設定する方法は，下記の3ステップになります．

1. まず，下記のデフォルトの `config.toml` の内容のうち，`favicon` と `faviconVersion` を変更します．

    ```yaml
    [params.meta]
        description         = "A theme by HTML5 UP, ported by Julio Pescador. Slimmed and enhanced by Patrick Collins. Multilingual by StatnMap. Powered by Hugo."
        author              = "HTML5UP and Hugo"
        favicon             = false <-- trueに変更する
        svg                 = true
        faviconVersion      = "1" <-- 1を削除する
        msColor             = "#ffffff"
        iOSColor            = "#ffffff"
    ```

2. `config.toml` を修正したら，static 配下に `favicon` ディレクトリを作成する．

3. `static/favicon` 配下に `favicon.ico` と `favicon-32x32.png` を配置する．
    なぜ favicon-32x32 かというと？
	- `layouts/partials/meta.html` の `rel=icon` に以下が記載されている
	  - favicon-32x32
	  - favicon-16x16
	  - site.webmanifest

    なので，これに合わせて名前を変更するか webmanifest を新しく作成し，その中に諸々の内容を記載する必要がある．

#### github.io でサイトを host した場合の path 設定

今回作成したサイトを github.io で host した場合に発生した内容です．
各メニューの URL として，`https://<アカウント名>.github.io/portfolio/home/` などとなって欲しいのですが，`https://<アカウント名>.github.io/portfolio/portfolio/home/` と `portfolio` が重なってしまうエラーが発生しました．

上記エラーを回避する方法の紹介になります．
`config.toml` ファイルに各メニューの設定をする箇所があります．

```yaml
[menu]

  [[menu.main]]
    name              = "Home"
    identifier        = "home"
    url               = "/" <-- /を削除する
    pre               = "<i class='fa fa-home'></i>"
    weight            = 1

  [[menu.main]]
    name              = "About"
    identifier        = "about"
    url               = "/about/" <-- 先頭の/を削除する
    pre               = "<i class='far fa-id-card'></i>"
    weight            = 2

  [[menu.main]]
    name              = "Blog"
    identifier        = "blog"
    url               = "/blog/" <-- 先頭の/を削除する
    pre               = "<i class='far fa-newspaper'></i>"
    weight            = 3
```

`url` の部分で先頭の`/`を削除することで，上記問題を回避することできます．

## おわりに

参考となる記事などがあまりなかったので，試行錯誤しながら行いました．そのため，もっと簡単にする方法が他にもあるかもしれないですので，もし他にあれば，Twitter などでコメント頂けると大変助かります．
