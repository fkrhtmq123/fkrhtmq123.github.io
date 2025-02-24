---
layout: post
title: Go Mongo Api & Swagger설정
categories: 
- Tech
tags:
- Linux
- Go
- Mac
- brew
lang: ko
---

Go프로젝트를 작성 시, Mongo설정에 도움이 되길 바라며 작성합니다. <br />
그리고 참고로 Swagger의 설정방법도 적으니 잘 부탁드립니다. <br />

go를 설치한 후, go를 초기화하여 초기 설정을 합니다. <br />
프로젝트의 폴더 구성은 밑에와 같습니다. <br />

```sh
go mod init `modules名`

# 例
go mod init github.com/fkrhtmq123/test-mongo-api
```

```sh
/TEST-MONGO-API
    |
    |-/cmd
    |   |-app
    |       |-main.go
    |-/intenal
    |   |-/config           # 설정 파일
    |   |-/handlers         # gin의 router에 설정하기 위한 API handler
    |   |-/models           # 데이터 모델 정의
    |   |-/mongodb          # mongodb의 접속 설정 파일
    |   |-/repositories     # 데이터 베이스 연동
    |   |-/services         # 비지니스 로직
    |   |-/utils            # 유틸리티 함수
    |-.env                  # 환경변수
    |-go.mod                # go mod init의 초기화로 작성이 됩니다.
    |-go.sum
```

# Mongodb의 접속 설정
먼저, 밑의 커맨드로 MongoDB의 라이브러리를 설치합니다.
```sh
go get go.mongodb.org/mongo-driver/v2/mongo
```
그리고 밑의 폴더의 위치에 파일을 작성하고 내용을 입력합니다.
```sh
# internal/mongodb/connection.go
# DB접속으로 클라이언트 작성
package mongodb

import (
	"fmt"

	"go.mongodb.org/mongo-driver/v2/mongo"
	"go.mongodb.org/mongo-driver/v2/mongo/options"
)

var (
	MongoClient      *mongo.Client
	MongoClinetError error
)

func ConnectToMongoDB() (*mongo.Client, error) {
    // mongodbURI
	uri := "mongodb+srv://testuser:TestTest123@test-mongo.test.mongodb.net/test-go"
	docs := "www.mongodb.com/docs/drivers/go/current/"
	if uri == "" {
		return nil, fmt.Errorf("set your 'MONGODB_URI' environment variable. See: " + docs + "usage-examples/#environment-variable")
	}

	// MongoDB 클라이언드 접속
	client, err := mongo.Connect(options.Client().ApplyURI(uri))
	if err != nil {
		return nil, err
	}
	return client, nil
}
```
```sh
# internal/repositories/match_repository.go
# DB연동
func GeYourCollection() ([]*models.YourModel, error) {
	collection := mongodb.MongoClient.Database("your-daba-base").Collection("your-collection")

	// 전체Collection데이터 참조
	cursor, err := collection.Find(context.TODO(), bson.D{})
	if err != nil {
		return nil, fmt.Errorf("failed to find matchs repository: %v", err)
	}
	defer cursor.Close(context.TODO())

	// 결과를 넣기 위한 슬라이스 선언
    # internal/models/match.go
	var matchs []*models.YourModel

	// 결과를 슬라이스에 추가
	for cursor.Next(context.TODO()) {
		var match match.Match
		if err := cursor.Decode(&match); err != nil {
			return nil, fmt.Errorf("failed to decode match: %v", err)
		}
		matchs = append(matchs, &match)
	}

	if err := cursor.Err(); err != nil {
		return nil, fmt.Errorf("cursor error: %v", err)
	}

	return matchs, nil
}
```
```sh
# cmd/app/main.go
package main

import (
	"flag"
	"log"

	"github.com/fkrhtmq123/test-mongo-api/internal/mongodb"
	"github.com/gin-gonic/gin"
)

func main() {
	// MongoDB 초기화
	InitMongo()
}

func InitMongo() {
	mongodb.MongoClient, mongodb.MongoClinetError = mongodb.ConnectToMongoDB()
	if mongodb.MongoClinetError != nil {
		log.Fatalf("Failed to connect to MongoDB: %v", mongodb.MongoClinetError)
	}

	// MongoDB 접속 성공 후, 추가 로직(로그)
	log.Println("Successfully connected to MongoDB!!")
}
```
```sh
go run cmd/app/main.go
```

위 커맨드로 go를 실행하고 MongoDB를 접속 합니다.

# Gin을 이용한 API구현
```sh
# Gin설치
go get github.com/gin-gonic/gin
```
cmd/app/main.go 수정
```sh
package main

import (
	"flag"
	"log"

	"github.com/fkrhtmq123/test-mongo-api/internal/handlers/match_handler" // ←추가
	"github.com/fkrhtmq123/test-mongo-api/internal/mongodb"
	"github.com/gin-gonic/gin" // ←추가
)

func main() {
	// MongoDB 초기화
	InitMongo()

	// Gin 라우터 초기화
	r := gin.Default()

	r.GET("/match", match_handler.GetMatchId)

	// 포드 설정 (초기값: 8080)
	port := flag.String("port", "8080", "Port to run the server on")
	flag.Parse()

	log.Printf("Starting server on port %s...\n", *port)

	// ✅ Gin 라우터 실행
	if err := r.Run(":" + *port); err != nil {
		log.Fatal("Failed to start server:", err)
	}
}
```

# Swagger설정
```sh
go get -u github.com/swaggo/swag/cmd/swag
go get -u github.com/swaggo/gin-swagger
go get -u github.com/swaggo/files
```
위의 커맨드로 Swagger를 설치 합니다. <br />

```sh
# Swagger초기화
swag init -g cmd/app/main.go -o cmd/app/docs
```

cmd/app/main.go 수정
```sh
package main

import (
	"flag"
	"log"

	_ "github.com/fkrhtmq123/test-mongo-api/cmd/app/docs"　// ←추가

	swaggerFiles "github.com/swaggo/files"　// ←추가
	ginSwagger "github.com/swaggo/gin-swagger" // ←추가

	"github.com/fkrhtmq123/test-mongo-api/internal/handlers/match_handler"
	"github.com/fkrhtmq123/test-mongo-api/internal/mongodb"
	"github.com/gin-gonic/gin"
)

func main() {
	// MongoDB 초기화
	InitMongo()

	// Gin 라우터 초기화
	r := gin.Default()

    // Swagger API맵핑
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	// Gin 라우터 초기화
	r := gin.Default()

	r.GET("/match", match_handler.GetMatchId)

	// 포트 설정 (초기값: 8080)
	port := flag.String("port", "8080", "Port to run the server on")
	flag.Parse()

	log.Printf("Starting server on port %s...\n", *port)

	// ✅ Gin 라우터 실행
	if err := r.Run(":" + *port); err != nil {
		log.Fatal("Failed to start server:", err)
	}
}
```
```sh
# internal/handlers/match_handler.go
// @Tags match
// @Description Create a new match with the provided Match_id
// @Accept  application/x-www-form-urlencoded
// @Produce  json
// @Param match_id formData string true "Match ID"
// @Success 200 {object} match.Match
// @Failure 400 {object} map[string]string "error: Match_id is required"
// @Router /match [post]
func PostMatchId(id string, c *gin.Context) {
	err := match_service.PostMatchId(id)
	if err != nil {
		c.JSON(http.StatusNotFound, gin.H{"error": "Match not create"})
		return
	}
	c.JSON(http.StatusOK, gin.H{
		"message": "Success Create Matchs",
		"result":  "",
	})
}

# Swagger설정을 설명합니다.
// 해당 API의 설명입니다
@Description Create a new match with the provided Match_id
// Group과 비슷한 기능입니다
@Tags match
// Content-Type입니다.
@Accept  application/x-www-form-urlencoded
// 응답(Response)데이터의 데이터 형식입니다
@Produce  json
// 서버에 보내는 내용입니다
@Param match_id formData string true "Match ID"
// 성공 할 시의 내용 예시입니다
@Success 200 {object} match.Match
// 실패 할 시의 내용 예시입니다（포트 별로 설정이 가능합니다）
@Failure 400 {object} map[string]string "error: Match_id is required"
// 엔드 포인트입니다（[]안은 get,post등 설정 합니다）
@Router /match [post]
```

Swagger 설정이 끝난 경우, 밑의 커맨드로 go를 실행합니다.
```sh
go run cmd/app/main.go -port=8080
```
실행 하고 밑의 URL로 접속하면 Swagger화면으로 접속합니다.

```sh
http://localhost:8080/swagger/index.html#/
```

<img src="/assets/img/go/go-mongo-api-swagger-01.png">