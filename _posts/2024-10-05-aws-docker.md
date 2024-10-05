---
title: AWS EC2에 linux에 Nakama설치
date: 2024-10-05 13:38:00
tags: 
- Tech
---
<h1>Nakama란</h1>
소셜 또는 리얼 타임 게임이나 어플에 사용이 가는 확장형 서버입니다.
</br></br>
Nakama서버를 사용하면 유저 인증, 소셜 네트워킹, 스토리지, 온라인 데이터 교환 등 사용이 가능합니다.
</br></br>

<h1>사전 준비</h1>
저자는 밑에 표의 스펙으로 설치, 실행했습니다.

|라이브터리명|버전|
|-|-|
|go|1.22.5|
|Nakama|3.21.1|
|Nakama-common|1.31.0|
|google.golang.org/protobuf|1.31.0|
|docker|25.0.5|
</br></br>

<h1>AWS EC2에 docker설치</h1>
※EC2가 Amazon Linux 2023일 경우로 설명

</br>
1. Docker패키지 설치

```cmd
sudo yum install -y docker
```

</br>
2. Docker서비스 시작

```cmd
sudo service docker start
```

</br>
3. 서버 부팅 시 서비스 시작

```cmd
sudo systemctl enable docker
```

</br>
4. 사용자 그룹 추가(선택 사항)</br>

(``sudo``를 사용하지 않고 명령어를 입력하기 위해서)

```cmd
sudo usermod -aG docker user

// 변경 사항 적용
newgrp docker
```
</br></br>

<h1>Go에 Nakama설치 및 docker설정</h1>
1.Go 프로젝트를 만들기 위해 폴더를 작성

```cmd
mkidr go-nakam-test
cd go-nakama-test
```

2.```go.mod``` 초기화

```cmd
go mod init project-name
```

3.nakama-common을 설치
go는 1.22.5가 설치되어있지만 저자는 1.21로 설정하여 설치

```cmd
// go가 1.21이기 때문에 nakama-common는 1.31.0
go get github.com/heroiclabs/nakama-common@1.31.0
```

4.vendor폴더 생성

```cmd
go mod vendor
```

위에 까지가 커맨드로 인한 설치
밑에 부터는 파일을 직접 작성합니다.

<h2>파일 생성</h2>

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
      test: ["CMD", "/nakama/nakama", "healthcheck"]
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
</br>
<h2>docker-compose설치</h2>

docker만 설치된 상황에서는 ```docker-compose.yml```를 사용을 안하고 기본 설정으로 실핼이 되므로 docker-compose 커맨드로 실행을 해야 ```docker-compose.yml```설정으로 실행이된다.

</br>
1.docker-compose커맨드 설치

```cmd
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

</br>
2.다운로드 실행 권한 부여

```cmd
sudo chmod +x /usr/local/bin/docker-compose
```
</br>
<h2>docker-compose실행</h2>

```cmd
docker-compose up -d
```
</br>
<h2>docker-compose 로그 확인</h2>

```cmd
docker-compose logs

// linux의 tail -f 과 같은 기능
docker-compose logs -f
```
</br>
<h2>docker-compose 정지</h2>

```cmd
docker-compose down
```