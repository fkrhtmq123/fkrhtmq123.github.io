---
layout: post
title: Go Mongo Api & Swagger設定
categories: 
- Tech
tags:
- Linux
- Go
- Mac
- brew
lang: jp
---

Goのプロジェクトを作成する時、Mongoの設定に役に立てればと思い作成します。<br />
ついでにSwaggerの設定方法を書きますのでよろしくお願いします。

goをインストールした後、goを初期化して初期設定をします。<br />
プロジェクトのフォルダの構成は下記のようになります。

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
    |   |-/config           # 設定ファイル
    |   |-/handlers         # ginのrouterで使用するためのAPI handler
    |   |-/models           # データモデル定義
    |   |-/mongodb          # mongodbの接続設定ファイル
    |   |-/repositories     # データベース連携
    |   |-/services         # ビジネスロジック
    |   |-/utils            # ユーティリティメソッド
    |-.env                  # 環境変数
    |-go.mod                # go mod initで初期化したら作成されます
    |-go.sum
```

# Mongodbの接続設定
まず、下記のコマンドでMongoDBのライブラリーをインストールします
```sh
go get go.mongodb.org/mongo-driver/v2/mongo
```
そして下記のフォルダの位置にファイルを作成して内容を入力します。
```sh
# internal/mongodb/connection.go
# DB接続でクライアント作成
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

	// MongoDB クライアントに接続
	client, err := mongo.Connect(options.Client().ApplyURI(uri))
	if err != nil {
		return nil, err
	}
	return client, nil
}
```
```sh
# internal/repositories/match_repository.go
# DBと連携します。
func GeYourCollection() ([]*models.YourModel, error) {
	collection := mongodb.MongoClient.Database("your-daba-base").Collection("your-collection")

	// 全体Collectionデータ参照
	cursor, err := collection.Find(context.TODO(), bson.D{})
	if err != nil {
		return nil, fmt.Errorf("failed to find matchs repository: %v", err)
	}
	defer cursor.Close(context.TODO())

	// 結果を入れるスライス宣言
    # internal/models/match.go
	var matchs []*models.YourModel

	// 結果をスライスに追加
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

	// MongoDB 接続成功後、追加ロジック（ログ）
	log.Println("Successfully connected to MongoDB!!")
}
```
```sh
go run cmd/app/main.go
```

上記のコマンドでgoを実行するとMongoDBに接続します。

# Ginを利用したAPI実装
```sh
# Ginのインストール
go get github.com/gin-gonic/gin
```
cmd/app/main.goの修正
```sh
package main

import (
	"flag"
	"log"

	"github.com/fkrhtmq123/test-mongo-api/internal/handlers/match_handler" // ←追記
	"github.com/fkrhtmq123/test-mongo-api/internal/mongodb"
	"github.com/gin-gonic/gin" // ←追記
)

func main() {
	// MongoDB 初期化
	InitMongo()

	// Gin ラウター初期化
	r := gin.Default()

	r.GET("/match", match_handler.GetMatchId)

	// ポート設定 (default値: 8080)
	port := flag.String("port", "8080", "Port to run the server on")
	flag.Parse()

	log.Printf("Starting server on port %s...\n", *port)

	// ✅ Gin ラウター実行
	if err := r.Run(":" + *port); err != nil {
		log.Fatal("Failed to start server:", err)
	}
}
```

# Swagger設定
```sh
go get -u github.com/swaggo/swag/cmd/swag
go get -u github.com/swaggo/gin-swagger
go get -u github.com/swaggo/files
```
上記のコマンドでSwaggerをインストールします。<br />

```sh
# Swaggerの初期化
swag init -g cmd/app/main.go -o cmd/app/docs
```

cmd/app/main.goの修正
```sh
package main

import (
	"flag"
	"log"

	_ "github.com/fkrhtmq123/test-mongo-api/cmd/app/docs"　// ←追記

	swaggerFiles "github.com/swaggo/files"　// ←追記
	ginSwagger "github.com/swaggo/gin-swagger" // ←追記

	"github.com/fkrhtmq123/test-mongo-api/internal/handlers/match_handler"
	"github.com/fkrhtmq123/test-mongo-api/internal/mongodb"
	"github.com/gin-gonic/gin"
)

func main() {
	// MongoDB 初期化
	InitMongo()

	// Gin ラウター初期化
	r := gin.Default()

    // Swagger APIマッピング
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	// Gin ラウター初期化
	r := gin.Default()

	r.GET("/match", match_handler.GetMatchId)

	// ポート設定 (default値: 8080)
	port := flag.String("port", "8080", "Port to run the server on")
	flag.Parse()

	log.Printf("Starting server on port %s...\n", *port)

	// ✅ Gin ラウター実行
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

# Swaggerの設定の説明です
// 該当APIの説明です
@Description Create a new match with the provided Match_id
// Groupにあたいします
@Tags match
// Content-Typeです
@Accept  application/x-www-form-urlencoded
// 応答(Response)データのデータ形式です
@Produce  json
// サーバーに送る内容です
@Param match_id formData string true "Match ID"
// 成功した時の内容例です
@Success 200 {object} match.Match
// 失敗した時の内容例です（ポート別に設定できます）
@Failure 400 {object} map[string]string "error: Match_id is required"
// エンドポイントです。（[]中はget,postなと設定します）
@Router /match [post]
```

Swaggerの設定が終わった場合、下記のでgoを実行
```sh
go run cmd/app/main.go -port=8080
```
実行して下記のURLに接続するとSwagger画面に接続します。

```sh
http://localhost:8080/swagger/index.html#/
```

<img src="/assets/img/go/go-mongo-api-swagger-01.png">