---
layout: post
title: Github blog local実行（Mac）
categories: 
- Tech
tags:
- Github
- Ruby
- Mac
lang: jp
---

前の投稿ではブログの作成までだったのですが実際にGitに反映する前にちゃんと動いているかlocalで実行して確認をする必要があります。

まず、Github blogはRubyで作られているのでRubyをインストールします。

# MacにRubyインストール
1.Homebrewインストール
```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
インストール時にはパスワード入力が必要なので入力窓が出れば入力

2.brewパスをパスリストに追加
```sh
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> /Users/awesome-d/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"  
```

brewがちゃんとインストールされてるか確認
```sh
brew --version
```

<img src="/assets/img/brew1.png">

3.Rubyインストール

```rbenv```とは?

いろんなバージョンのRubyを管理するためのプログラムです。<br />

```ruby-build```とは?

UNIX系のシステム上で異なるバージョンのRubyをコンパイルし、インストールするための ```rbenv install```コマンドを提供する ```rbenv```のプラグインです。

[https://ruby.studio-kingdom.com/rbenv/ruby_build/](https://ruby.studio-kingdom.com/rbenv/ruby_build/)

```sh
brew install rbenv ruby-build
```

インストールができるRubyのバージョンのを確認します。
```sh
rbenv install -l
```

<img src="/assets/img/brew2.png">

最新バージョンをインストールしてもいいのですが最新バージョンでブログテーマがちゃんと動かない場合もなるので２つくらい前のバージョンをインストールします。

```sh
rbenv install {インストールするバージョン}
```

筆者は ```3.1.6```をインストール

インストール直後 ```rbenv versions```をすると先ほどインストールしたバージョンと systemというバージョン、２つがあり現在使われてるバージョンの左に (*)が表示されるが最初はsystemになっています。

<img src="/assets/img/brew3.png">

```sh
rbenv global 3.1.6
rbenv rehash
```

上記のコマンドでglobalバージョンを3.1.6に変更し、Rubyのコマンドを使用するために ```rbenv rehash```を実行します。<br />
（もし、バージョンを変える時そのたび実行する必要があります）

<img src="/assets/img/brew4.png">

4.rbenvでインストールしたRuby適用

rbenvでインストールしたRubyはまだmacMacにRubyとして適用されてない状態なのでrbenvをパスに適用します。

パスのファイルを開きます。
```sh
open ~/.zshrc
```

下記の２列をパスファイルに追加
```sh
export PATH={$Home}/.rbenv/bin:$PATH && \
eval "$(rbenv init -)"
```

コマンドを使ってパスファイルの修正内容を適用
```sh
source ~/.zshrc
```

# bundler 及び jekyll インストール
```sh
gem install bundler jekyll
```

ちゃんとインストールされているか確認
```sh
jekyll -v
```

# テーマで使用しているパッケージをインストール
```sh
bundler install
```
上記のコマンドを使用して```Gemfile```の中にあるパッケージをインストールします。

# ブログのlocal実行
```sh
bundle exec jekyll serve
```
上記のコマンドを使用してlocal実行されます。

<br /><br />

※localで実行する時の筆者の```Gemfile``` 内容

```sh
source 'https://rubygems.org'
gem 'github-pages', group: :jekyll_plugins
#gem 'jekyll-admin', group: :jekyll_plugins

gem "webrick", "~> 1.8.2"
gem 'jekyll', '3.10.0'
```