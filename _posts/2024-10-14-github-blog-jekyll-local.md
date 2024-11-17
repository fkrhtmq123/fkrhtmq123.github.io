---
layout: post
title: 깃허브 블로그 로컬 실행(Mac)
categories: 
- Tech
tags:
- Github
- Ruby
- Mac
lang: ko
---

전 포스팅에는 블로그 생성까지만 했고 실제로 깃에 올려 반영하기 전에 잘 움직이고 있는지 로컬에서 실행하여 확인해야합니다.

먼저, 깃허브 블로그는 루비로 만들어 져있기 때문에 루비를 설치를 해야합니다.

# Mac에 루비 설치
1.Homebrew설치
```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
설치 시에는 패스워드 입력이 필요하니 패스워드 입력창이 뜨면 입력

2.brew패스를 패스 리스트에 추가
```sh
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> /Users/awesome-d/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"  
```

brew가 제대로 설치 되었는지 확인
```sh
brew --version
```

<img src="/assets/img/brew1.png">

3.루비설치

```rbenv```란?

여러 루비 버전을 관리위한 프로그램입니다.<br />

```ruby-build```란?

UNIX계열의 시스템상에 서로 다른 버전의 루비를 컴파일하여 설치하지 위한 ```rbenv install```커맨드를 제공하는  ```rbenv```의 플러그인 입니다.

[https://ruby.studio-kingdom.com/rbenv/ruby_build/](https://ruby.studio-kingdom.com/rbenv/ruby_build/)

```sh
brew install rbenv ruby-build
```

설치를 할 수있는 루버 버전을 확인 
```sh
rbenv install -l
```

<img src="/assets/img/brew2.png">

최신 버전을 설치해도 되지만 최신 버전이 블로그 테마에서 제대로 움직이지 않을수도 있으니 두 개정도 버전 전을 설치합니다.

```sh
rbenv install {원하는 버전}
```

필자는 ```3.1.6```을 설치

설치한 직후 ```rbenv versions```를 하면 방금 설치한 버전과 system이라는 버전, 두 개가 있는데 현재 쓰고 있는 버전에는 버전 왼쪽에 (*)가 표시 되어있는데 처음에는 system이 되어 있다.

<img src="/assets/img/brew3.png">

```sh
rbenv global 3.1.6
rbenv rehash
```

위 커맨드로 global버전을 3.1.6로 변경하고 루비 커맨드를 사용하기 위해 ```rbenv rehash```를 실행합니다.<br />
(만약 버전을 바꿀때는 그때마다 실행할 필요가 있습니다)

<img src="/assets/img/brew4.png">

4.rbenv로 설치된 루비 적용

rbenv로 설치한 루비는 아직 Mac에 루비로서 적용이 안된 상태이기 때문에 rbenv를 패스에 적용.

패스 파일을 열어
```sh
open ~/.zshrc
```

밑의 두 줄을 패스 파일에 추가
```sh
export PATH={$Home}/.rbenv/bin:$PATH && \
eval "$(rbenv init -)"
```

커맨드를 써서 패스 파일 적용
```sh
source ~/.zshrc
```

# bundler 및 jekyll 설치
```sh
gem install bundler jekyll
```

제대로 설치가 됬는지 확인
```sh
jekyll -v
```

# 테마에서 사용하는 패키지 설치
```sh
bundler install
```
위 커맨드를 사용하면 ```Gemfile```안에 명시된 패키지를 설치를 합니다.

# 블로그 로컬 실행
```sh
bundle exec jekyll serve
```
위 커맨드를 사용하면 로컬로 실행이 됩니다.

<br /><br />

※로컬 실행 시 필자의 ```Gemfile``` 내용

```sh
source 'https://rubygems.org'
gem 'github-pages', group: :jekyll_plugins
#gem 'jekyll-admin', group: :jekyll_plugins

gem "webrick", "~> 1.8.2"
gem 'jekyll', '3.10.0'
```