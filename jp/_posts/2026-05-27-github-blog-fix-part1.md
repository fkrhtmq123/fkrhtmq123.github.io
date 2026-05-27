---
layout: post
title: "GitHubブログ修正記 1部: GitHub Pagesのデプロイエラーとbaseurl問題の解決"
categories:
- Tech
tags:
- Github
- Jekyll
- Ruby
- GitHub Pages
lang: jp
---

GitHubブログを運用していると、記事の内容より先に**デプロイやパスの問題**に悩まされることがあります。今回の1部では、実際に遭遇した GitHub Pages のデプロイエラーと、誤って設定された baseurl の問題を整理します。

<br /><br />

![GitHub Pagesデプロイとbaseurl修正の要約](/assets/img/posts/2026-05-27-github-blog-fix/2026-05-27-part1-deploy-path-fix-v2.svg)

<br /><br />

# 問題の始まり
最初は単純にデプロイが失敗しているだけだと思っていました。ところが調べてみると、問題はもっと複雑でした。

- GitHub Pages のデプロイが正常に終わらない
- CSS / JS のパスが不自然に生成される
- `user pages` なのに `project pages` のような URL になる

たとえば、以下のような誤ったパスが作られていました。

```text
https://fkrhtmq123.github.io/pages/fkrhtmq123/assets/css/style.css
```

本来は次のようになるべきでした。

```text
https://fkrhtmq123.github.io/assets/css/style.css
```

つまり、サイトが**ユーザーページではなくプロジェクトページのように解釈されている状態**でした。

<br /><br />

# `_config.yml` の修正
まず修正したのは `_config.yml` です。

```yml
url: "https://fkrhtmq123.github.io"
baseurl: ""
repository: fkrhtmq123/fkrhtmq123.github.io
assets: /assets
```

ポイントは2つです。

1. `url` を実際の GitHub Pages のURLに明示する
2. `baseurl` を空文字にして余計なパスが付かないようにする

この修正で、ブログ内の静的リソースとリンクがユーザーページ基準で正しく生成されるようになりました。

<br /><br />

# Workflow も整理
デプロイ workflow 側でも、GitHub Pages のビルド環境を明確に指定しました。

```yaml
env:
  JEKYLL_ENV: production
  GITHUB_PAGES: true
  PAGES_REPO_NWO: fkrhtmq123/fkrhtmq123.github.io
```

これらの値は Jekyll が GitHub Pages 環境を正しく認識するために重要です。特に `PAGES_REPO_NWO` は repo ベースのパス判定に大きく関わりました。

また、デプロイが本当に成功したかどうかを感覚ではなく、以下の項目で確認しました。

- workflow run の状態
- artifact アップロード成功の有無
- Pages 設定値
- rerun 後の結果

<br /><br />

# 確認結果
修正後は次のように改善されました。

- CSS / JS のパスが正しく生成される
- 記事リンクが user pages 基準で出力される
- GitHub Actions で artifact のアップロードが成功する
- deploy workflow が正常に動作する

つまり今回の問題は、コードそのものよりも**Pages の設定と Jekyll のパス解釈**に近いものでした。

<br /><br />

# 1部を終えて
今回の1部は、記事を書く前に片付ける必要があった土台の作業です。

- GitHub Pages のデプロイ問題の確認
- user pages / project pages の違いの把握
- `_config.yml` の `url` と `baseurl` の修正
- workflow の環境値の整理

2部では、このデプロイ問題を解決したあとに残った**多言語ページネーションと言語切り替えの問題**を整理します。
