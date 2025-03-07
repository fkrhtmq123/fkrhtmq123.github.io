---
layout: post
title: go version アップデート(Mac)
categories: 
- Tech
tags:
- Linux
- Go
- Mac
- brew
lang: jp
---

Macでのgo versionをアップデートをやり方をここに作成します。

まず、簡単にgoをアップデートします。
下記のコマンドでgoをアップデートします。

```sh
brew upgrade go
```

コマンドでアップデートをしたら最新のgo versionを取得し、インストールします。

# 1. アップデート成功時
アップデートに成功するとこのようなイメージが出ます。

<img src="/assets/img/go/go-update-01.png">

イメージはすでにインストール済みなので```already installed and up-to-date```とでますが実際がインストールがされる過程が出るので大丈夫です。

```sh
go version

go version go1.23.6 darwin/arm64
```
<br/>

# 2. アップデート失敗
アップデートに失敗すると
```sh
go version

go version go1.22.1 darwin/arm64
```
上記のようにversionがアップデートされず、昔のversionを見せます。

<br />

## 2.1. 強制的に特定のversionをインストール
```sh
brew install go@1.23
```
上記のコマンドで特定のversionをインストールをします。
version確認で更新されれば大丈夫ですが、更新されず昔のversionが出れば次の方法に移しましょう。

<br />

## 2.2. PATHの確認と修正
```sh
vi ~/.zshrc
```
上記のファイルを確認して下記のexportを最後の行に追加して更新

```sh
# ~/.zshrc
export PATH="/opt/homebrew/opt/go@1.23/bin:$PATH"

# ~/.zshrcの更新
source ~/.zshrc
```
version確認で更新されれば大丈夫ですが、更新されず昔のversionが出れば次の方法に移しましょう。

<br />

## 2.3. go env確認と修正
```sh
....
GOROOT='/opt/homebrew/opt/go@1.23/libexec'
GOSUMDB='sum.golang.org'
GOTMPDIR=''
GOTOOLCHAIN='go1.23.6'
GOTOOLDIR='/opt/homebrew/opt/go@1.23/libexec/pkg/tool/darwin_arm64'
GOVCS=''
GOVERSION='go1.23.6'
GODEBUG=''
GOTELEMETRY='local'
....
```
上記の```GOROOT```,```GOTOOLCHAIN```,```GOTOOLDIR```が違う場合、下記のコマンドで上記の３つのルートを修正します。
```sh
go env -w GOROOT="/opt/homebrew/opt/go@1.23/libexec"
go env -w GOTOOLDIR="/opt/homebrew/opt/go@1.23/libexec/pkg/tool/darwin_arm64"
go env -w GOTOOLCHAIN="go1.23.6"
```
2.3の方法が自分が試した最後の方法でversionが修正されました。