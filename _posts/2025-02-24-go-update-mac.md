---
layout: post
title: go version 업데이트(Mac)
categories: 
- Tech
tags:
- Linux
- Go
- Mac
- brew
lang: ko
---

Mac에서의 go version을 업데이트를 하는 방법을 여기에 작성 하겠습니다.

먼저, 간단하게 go를 업데이트를 합니다.
밑의 커맨드로 go를 업데이트를 합니다.

```sh
brew upgrade go
```

커맨드로 업데이트를 하면 최신 go version을 가져오고 설치를 합니다.

# 1. 업데이트에 성공하면
업데이트에 성공하면 밑에와 같은 이미지로 나옵니다.

<img src="/assets/img/go/go-update-01.png">

이미지는 이미 설치가 되었기 때문에 ```already installed and up-to-date``` 라고 나오지만 실제로는 설치가 되는 과정이 나오기 때문에 괜찮습니다.

```sh
go version

go version go1.23.6 darwin/arm64
```
<br/>

# 2. 업데이트에 실패하면
업데이트에 실패하면
```sh
go version

go version go1.22.1 darwin/arm64
```
위와 같이 version가 업데이트가 안되고, 예전 version이 보일 것입니다.

<br />

## 2.1. 강제로 특정 version을 설치
```sh
brew install go@1.23
```
위 커맨드로 특정 version을 설치합니다.
version 확인 시 갱신이 되면 괜찮지만, 갱신이 안되고 예전 version이 나오면 다음 방법으로 시도해 보세요.

<br />

## 2.2. PATH 확인 및 수정
```sh
vi ~/.zshrc
```
위 커맨드를 확인하고 밑의 export를 마지막 행이 추가를 하고 갱신

```sh
# ~/.zshrc
export PATH="/opt/homebrew/opt/go@1.23/bin:$PATH"

# ~/.zshrc 갱신
source ~/.zshrc
```
version 확인 시 갱신이 되면 괜찮지만, 갱신이 안되고 예전 version이 나오면 다음 방법으로 시도해 보세요.

<br />

## 2.3. go env 확인 및 수정
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
위의 ```GOROOT```,```GOTOOLCHAIN```,```GOTOOLDIR``` 가 다를 경우, 밑의 커맨드로 상기 3개의 루트를 수정합니다.
```sh
go env -w GOROOT="/opt/homebrew/opt/go@1.23/libexec"
go env -w GOTOOLDIR="/opt/homebrew/opt/go@1.23/libexec/pkg/tool/darwin_arm64"
go env -w GOTOOLCHAIN="go1.23.6"
```
2.3의 방법이 제가 시도한 마지막 방법으로 version이 수정되었습니다.