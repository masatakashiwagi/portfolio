+++
author = "Masataka Kashiwagi"
title = "VSCode と git を連携して push できるようにするまで"
description = "VSCode と git の連携"
date = 2021-03-26T22:18:34+09:00
draft = false
share = true
showLicense = false
support = true
tags = ["Dev"]
+++

## はじめに

今まで仕事では，開発環境として IntelliJ を使っていたのですが，最近は VSCode の人気が高く Extensions も便利なものが多くあるということで，個人的な作業をする時は VSCode を使ってみようと思って使っています．
そんな中で，タイトルにもあるように VSCode から git push しようとしたら，`<アカウント名>@github.com: Permission denied (publickey).` とエラーが出たので，それを解消して VSCode で git push できるようにした備忘録になります．

## エラー原因（SSH 接続エラー）

「**Permission denied (publickey)**」とあり，GitHub に SSH 接続するために，公開鍵を登録しておかないといけないのですが，それをしていなかったので，エラーが発生したということになります．

以下のコマンドを打つことで接続を確認することができます．

```bash
ssh -T git@github.com
> git@github.com: Permission denied (publickey).
```

ではどうすればいいかというと，鍵を生成して GitHub に登録すればいいということになります．

## 公開鍵と秘密鍵の作成

詳細な作成方法は[こちらの記事](https://qiita.com/shizuma/items/2b2f873a0034839e47ce)が参考になります．

簡単に手順を載せておきます．

```bash
cd ~/.ssh
ssh-keygen -t rsa -b 4096 -C "<メールアドレス>" -f github_rsa  # オプションをいくつか設定して，鍵を生成

# 以下実行結果（一部マスクしてます）
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in github_rsa.
Your public key has been saved in github_rsa.pub.
The key fingerprint is:
SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx <メールアドレス>
The key's randomart image is:
+---[RSA 4096]----+
|===              |
|.B      o     .  |
|o..  . *     . o |
|. . . B +. oo .o*|
| . o * OSo.oooo*+|
|  . = + = o ..*..|
|   E . . o . . ..|
|        . .      |
|                 |
+----[SHA256]-----+
```

鍵の種類を RSA にし，鍵の長さを4096にしています．ファイル名は github_rsa と設定しました．

## GitHub に生成した公開鍵を登録

```bash
cd ~/.ssh
ssh-add -K github_rsa  # 秘密鍵をssh-agentデーモンに登録
pbcopy < github_rsa.pub  # pbcopyコマンドで公開鍵の中身をクリップボードにコピー
```

この後は，コピーした公開鍵の中身を GitHub に登録します．

GitHub のアカウントから Settings に進み，SSH and GPG keys を選択し，New SSH key を押して，先程コピーした中身をペーストし，名前を決めて保存します．

## SSH 接続確認

保存が完了したら，SSH 接続できるか確認するために，以下のコマンドを打って確認します．

```bash
ssh -T git@github.com
>Hi <ユーザー名>! You've successfully authenticated, but GitHub does not provide shell access.
```

## Remote 設定の上書き

ここまで来たら後一息で，最後に remote の設定を上書きします．
以下のような感じでリポジトリ名を書いて，実行すればOK．

```bash
git remote set-url origin git@github.com:<ユーザー名>/<リポジトリ名>.git
```

## VSCode からgit push

![VSCodeからpush](../../img/vscode_git_connect_img1.png "git push from VSCode")

今までの設定が完了していれば，上記画像の手順で VSCode の画面から簡単に git に commit や push などの操作を行うことができます．

## おわりに

まだまだ VSCode 初心者なので，使いやすい Extensions を取り入れて開発環境をカスタマイズしていきたいと思います！

## 参考

- [初めてのgitは5ステップで完了](https://qiita.com/takuyanin/items/c6a097028a837052c90c)
- [GitHubでssh接続する手順~公開鍵・秘密鍵の生成から~](https://qiita.com/shizuma/items/2b2f873a0034839e47ce)
