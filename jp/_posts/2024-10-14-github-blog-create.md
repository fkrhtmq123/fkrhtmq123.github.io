---
layout: post
title: Github blog作成
categories: 
- Tech
tags:
- Github
- Ruby
lang: jp
---

筆者は開発者として働きながら前の職場を含めて３年になるが技術ブログがないとダメな気がしていろんなブログを確認したのですがGithubでブログが作れると知りGithubでどうやってブログを作れるか書こうと思います。

<br /><br />

# Git プロジェクト作成
Gitブログを作るためにまずGitプロジェクト作成を作成します。

<img src="/assets/img/git_blog_create_project.png">

[Repository name] には
```cmd
[USERNAME].github.io
```

<br /><br />

# Git Clone깃 클론Git プロジェクト
上記で作成したGitプロジェクトをCloneします。

<img src="/assets/img/git_blog_clone.png">

作成したプロジェクトページに緑色のボタン ```Code```をクリックし ```HTTPS```でGit URLをコピー
<br />
コピーしたURLをCloneしたいフォルダに移動し、Clone

```cmd
cd developer
git clone https://github.com/fkrhtmq123/fkrhtmq123.github.io.git
```

<br /><br />

# ブログのテーマ決め
[http://jekyllthemes.org/](http://jekyllthemes.org/)
<br />
上記のURLで使いたいテーマを決めてクリック

<img src="/assets/img/jekyll_theme1.png">

ライセンスを確認してダウンロードをクリックしてテーマをダウンロード<br />
ライセンスが無料（MIT）じゃないものがあるので要確認

<img src="/assets/img/jekyll_theme2.png">

<br /><br />

# テーマ適用
ダウンロードしたテーマのZIPを解凍して、内容を作成してCloneしたプロジェクトにコピペ

<img src="/assets/img/jekyll_theme3.png">

<img src="/assets/img/jekyll_theme4.png">

<br /><br />

# Git commit & push
Gitにcommit & pushするとテーマが適用されます。<br />
適用に早いものも遅いものをあるので約1分を待ちます。

これでGit Blogが完成。

次にGit Blogをlocalで実行する方法を説明します。