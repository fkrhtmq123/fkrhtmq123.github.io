---
layout: post
title: AWS EC2 linux環境に Nakama設置(Docker利用)
categories: 
- Tech
tags:
- AWS
- Linux
- Go
- Docker
lang: jp
---

# Nakamaとは
ソーシャルまたはリアルタイムゲーム、アプリに使用可能な拡張型サーバーです。
<br /><br />
Nakamaサーバーを使用するとユーザー認証、ソーシャルネットワーキング、ストレージ、オンラインデータ交換などに使用が可能です。
<br /><br />

# 事前準備
筆者は下記の表のスペックで設定、実行しました。

| ライブラリー名 | バージョン |
| --- | --- |
| go | 1.22.5 |
| Nakama |3.21.1 |
| Nakama-common | 1.31.0 |
| google.golang.org/protobuf | 1.31.0 |
| docker | 25.0.5 |

<br /><br />

# AWS EC2に dockerを設定
※EC2のインスタンスが Amazon Linux 2023の場合の説明

<br />
1. Dockerパッケージ設置

```sh
sudo yum install -y docker
```

<br />
2. Dockerサービス開始

```sh
sudo service docker start
```

<br />
3. サーバー再起動時サービス開始

```sh
sudo systemctl enable docker
```

<br />
4. 使用者グループを追加(選択事項)<br />

(``sudo``を毎回入力しなくてコマンドを入力可能にするため)

```sh
sudo usermod -aG docker user

// 変更事項適用
newgrp docker
```
<br /><br />

# Goに Nakama設定及びdocker設定
1.Goのプロジェクトを作るためにフォルダ作成

```sh
mkidr go-nakam-test
cd go-nakama-test
```

2.```go.mod``` 初期化

```sh
go mod init project-name
```

3.nakama-commonを設定
goは 1.22.5ですが、筆者が1.21に設定して開始

```sh
// goが 1.21なので nakama-common는 1.31.0
go get github.com/heroiclabs/nakama-common@1.31.0
```

4.vendorフォルダ生成

```sh
go mod vendor
```

上記までがコマンドで設置
下記からはファイルを直接作成します。

## ファイル作成

```javascript
// main.go
package main

import (
	"context"
	"database/sql"
	"time"

	"github.com/heroiclabs/nakama-common/runtime"
)

func InitModule(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule, initialzer runtime.Initializer) error {
	initStart := time.Now()

	err := initialzer.RegisterRpc("healthcheck", RpcHealthcheck)
	if err != nil {
		return err
	}

	logger.Info("Module loaded in %dmx", time.Since(initStart).Milliseconds())

	return nil
}

// healthcheck.go
package main

import (
	"context"
	"database/sql"
	"encoding/json"

	"github.com/heroiclabs/nakama-common/runtime"
)

type HealthcheckResponse struct {
	Success bool `json: "success"`
}

func RpcHealthcheck(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule, payload string) (string, error) {
	logger.Debug("Healthcheck RPC  called")
	response := &HealthcheckResponse{Success: true}

	out, err := json.Marshal(response)
	if err != nil {
		logger.Error("Error marshalling response type to JSON: %v", err)
		return "", runtime.NewError("Cannon marshal type", 13)
	}

	return string(out), nil
}

// local.yml
logger:
  level: "DEBUG"

console:
  port: 7351
  username: "admin"
  password: "password"

// Dockerfile
FROM heroiclabs/nakama-pluginbuilder:3.21.1 AS builder

ENV GO111MODULE on
ENV CGO_ENABLED 1

WORKDIR /backend
COPY . .

RUN go build --trimpath --mod=vendor --buildmode=plugin -o ./backend.so

FROM heroiclabs/nakama:3.21.1

COPY --from=builder /backend/backend.so /nakama/data/modules
COPY --from=builder /backend/local.yml /nakama/data/
COPY --from=builder /backend/*.json /nakama/data/modules

// docker-compose.yml
services:
  postgres:
    command: postgres -c shared_preload_libraries=pg_stat_statements -c pg_stat_statements.track=all
    environment:
      - POSTGRES_DB=nakama
      - POSTGRES_PASSWORD=localdb
    expose:
      - "8080"
      - "5432"
    image: postgres:12.2-alpine
    ports:
      - "5432:5432"
      - "8080:8080"
    volumes:
      - data:/var/lib/postgresql/data

  nakama:
    build: .
    depends_on:
      - postgres
    entrypoint:
      - "/bin/sh"
      - "-ecx"
      - >
        /nakama/nakama migrate up --database.address postgres:localdb@postgres:5432/nakama &&
        exec /nakama/nakama --config /nakama/data/local.yml --database.address postgres:localdb@postgres:5432/nakama        
    expose:
      - "7349"
      - "7350"
      - "7351"
    healthcheck:
      test: ["sh", "/nakama/nakama", "healthcheck"]
      interval: 10s
      timeout: 5s
      retries: 5
    links:
      - "postgres:db"
    ports:
      - "7349:7349"
      - "7350:7350"
      - "7351:7351"
    restart: unless-stopped

volumes:
  data:
```
<br />
## docker-compose設置

dockerだけ設置してる状況だと ```docker-compose.yml```ファイルを使用せず基本設定で実行されるので docker-compose コマンドを使って ```docker-compose.yml```の設定で実行します。

<br />
1.docker-composeコマンド設置

```sh
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

<br />
2.ダウンロード実行権限付与

```sh
sudo chmod +x /usr/local/bin/docker-compose
```
<br />
## docker-compose実行

```sh
docker-compose up -d
```
<br />
## docker-composeログ確認

```sh
docker-compose logs

// linuxの tail -f と同じ機能
docker-compose logs -f
```
<br />
## docker-compose停止コマンド

```sh
docker-compose down
```
