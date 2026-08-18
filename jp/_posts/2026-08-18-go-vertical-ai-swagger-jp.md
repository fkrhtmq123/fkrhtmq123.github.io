---
layout: post
title: Go + MongoDB + Swaggerで構築するバーティカルAI APIサーバーとドキュメント化
categories:
- Tech
tags:
- Go
- MongoDB
- Swagger
- API
lang: jp
---

バーティカルAIは、単に「AI技術を組み込んでみる実験」の段階を越え、ビジネスバックエンドへ統合されつつあります。金融、医療、リサーチのように特定のドメインが明確な領域では、柔軟なスキーマによるリアルタイムの追跡ログと、信頼性の高い仕様書管理がシステム安定性の鍵になります。

この記事では、GoのHTTPフレームワークGinを使ってバーティカルAI処理を実行し、結果をMongoDBへ永続保存し、フロントエンドや企画部門との協業に向けてSwaggerドキュメントを自動生成する設計と実装方法を解説します。

<br />

# 1. なぜこの組み合わせが有効なのか

## 1-1. Go (Golang)
Goはシンプルな並行処理モデル（Goroutine）と高速なコンパイルを提供します。AI APIバックエンドのように大量の非同期ネットワークI/Oやデータストリーミングが繰り返されるアーキテクチャでも、限られたリソースで高い性能を発揮し、軽量なコンテナとしてデプロイできます。

## 1-2. MongoDB
バーティカルAIが返す原文テキスト、構造化されたインデックス情報、トークン消費ログ、デバッグ用のRaw Responseなどは、非定型・半定型データが組み合わさっています。MongoDBはスキーマを変更せずに、このような複雑な追跡ログを柔軟かつ高速に保存するのに適しています。

## 1-3. Swagger (OpenAPI)
バーティカルAIプロジェクトでは、複数の関係部署が継続的に仕様を合わせる必要があります。Swaggerドキュメントをソースコードのコメントから直接ビルドすれば、ドキュメントと実装の乖離（Docs-Drift）を防ぎ、フロントエンドが直接テストできる環境をすぐに共有できます。

<br />

# 2. プロジェクト構造の設計

実務でファイルを分離しやすいよう、次のプロジェクトツリーを基準に構成します。

```text
my-vertical-ai/
├── main.go          # メインパッケージ（ルーターとライフサイクルを構成）
├── docs/            # swag initが自動生成するSwagger仕様ファイル
├── go.mod           # Goモジュール管理定義
├── go.sum           # 外部パッケージのハッシュ整合性検証用
└── .env             # 環境変数定義用のローカルファイル
```

<br />

# 3. 開発依存パッケージのインストール

ターミナルを開き、必要なフレームワークパッケージとドライバーをインストールします。

```sh
# Gin Webフレームワーク
go get github.com/gin-gonic/gin

# MongoDB公式ドライバー
go get go.mongodb.org/mongo-driver/mongo

# Swagger統合用パッケージ
go get github.com/swaggo/gin-swagger
go get github.com/swaggo/files
```

宣言的なコードコメントをOpenAPI JSON仕様へコンパイルする`swag` CLIもインストールします。

```sh
go install github.com/swaggo/swag/cmd/swag@latest
```

<br />

# 4. 実務向けGoバックエンド統合コードの実装

以下は、環境変数のfallback処理、`domain`フィールドの単一インデックスの非同期構築、Swagger CLIのコンパイル時にエンドポイントを自動マッピングするOpenAPIコメントを適用した`main.go`の統合コードです。

{% raw %}
```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "time"

    "github.com/gin-gonic/gin"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger "github.com/swaggo/gin-swagger"
    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"

    _ "my-vertical-ai/docs"
)

// @title           Vertical AI API Server
// @version         1.0
// @description     特定ドメインに特化したバーティカルAI処理を実行・記録するAPIサーバーです。
// @host            localhost:8080
// @BasePath        /api/v1

var logCollection *mongo.Collection

type AIRequest struct {
    Domain string `json:"domain" binding:"required" example:"finance"`
    Text   string `json:"text" binding:"required" example:"2026年第2四半期の営業利益分析を依頼"`
}

type AIResponse struct {
    Status    string    `json:"status" example:"success"`
    Result    string    `json:"result" example:"[finance]分野のコンテキストに基づくAIインサイト生成完了"`
    Saved     bool      `json:"saved" example:"true"`
    Timestamp time.Time `json:"timestamp"`
}

func main() {
    mongoURI := os.Getenv("MONGODB_URI")
    if mongoURI == "" {
        mongoURI = "mongodb://localhost:27017"
    }

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    client, err := mongo.Connect(ctx, options.Client().ApplyURI(mongoURI))
    if err != nil {
        log.Fatal("MongoDB接続失敗: ", err)
    }
    if err := client.Ping(ctx, nil); err != nil {
        log.Fatal("MongoDB Ping失敗: ", err)
    }

    dbName := os.Getenv("MONGODB_DB")
    if dbName == "" {
        dbName = "vertical_ai_db"
    }
    logCollection = client.Database(dbName).Collection("inference_logs")

    go func() {
        idxCtx, idxCancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer idxCancel()
        _, err := logCollection.Indexes().CreateOne(idxCtx, mongo.IndexModel{
            Keys: bson.D{{Key: "domain", Value: 1}},
        })
        if err != nil {
            log.Println("domainインデックス構築失敗: ", err)
        } else {
            log.Println("domainフィールドの単一インデックス作成完了")
        }
    }()

    r := gin.Default()
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
    r.POST("/api/v1/vertical-ai/analyze", AnalyzeVerticalAI)

    log.Println("APIルーティング設定完了。ポート8080で起動待機中...")
    r.Run(":8080")
}

// @Summary      バーティカルAIドメイン分析
// @Description  ドメインと非定型テキストを受け取り、分析結果を生成してMongoDBへ記録します。
// @Tags         Vertical AI
// @Accept       json
// @Produce      json
// @Param        request body AIRequest true "分析入力モデル"
// @Success      200 {object} AIResponse "AI分析結果とログ保存完了"
// @Failure      400 {object} map[string]string "不正なJSONまたは必須パラメーター不足"
// @Router       /vertical-ai/analyze [post]
func AnalyzeVerticalAI(c *gin.Context) {
    var req AIRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "error": "JSONリクエストが正しくありません。必須パラメーターを確認してください。",
        })
        return
    }

    // 実際のLLMまたはオンプレミスAIサービスの呼び出し処理をここに組み込む
    resultText := "[" + req.Domain + "]分野のコンテキストに基づくAIインサイト生成完了"

    logDoc := bson.M{
        "domain":    req.Domain,
        "text":      req.Text,
        "result":    resultText,
        "createdAt": time.Now(),
    }

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    insertResult, insertErr := logCollection.InsertOne(ctx, logDoc)
    if insertErr != nil {
        log.Println("MongoDBログ保存失敗: ", insertErr)
    } else {
        log.Printf("MongoDB保存完了: %v", insertResult.InsertedID)
    }

    c.JSON(http.StatusOK, AIResponse{
        Status:    "success",
        Result:    resultText,
        Saved:     insertErr == nil,
        Timestamp: time.Now(),
    })
}
```
{% endraw %}

<br />

# 5. 主要アーキテクチャとコードの詳細分析

## 5-1. なぜSwaggerコメントの宣言が必須なのか
多くのバックエンドガイドでは、`swag init`の実行方法だけを紹介し、実際のハンドラー上部にOpenAPIアノテーションを記述しないというミスがあります。

`@Summary`、`@Description`、`@Param`、`@Success`、`@Router`は、`swag` CLIが解析する機械的な宣言部です。これらを基に正確な`swagger.json`スキーマが生成されます。省略するとSwagger UIには接続できても、エンドポイントのAPI仕様が一覧に表示されない状態になります。

## 5-2. 非同期インデックス構築とバックグラウンド処理
ドメイン別の履歴分析・検索クエリが多いバーティカルAIシステムでは、データが増加すると`domain`フィールドのフルスキャンがボトルネックになります。これを防ぐため、DB初期化ライフサイクルの中でGoroutineを起動し、`domain`インデックスを構築します。

```go
go func() { ... }()
```

このバックグラウンド処理により、Ginの起動処理がインデックス作成を待つことで初期起動が遅延する問題を防げます。

## 5-3. MongoDB処理用の専用Context Isolation戦略
ネットワークタイムアウトの設定が不十分だと、MongoDB接続の遅延がAPIリクエスト全体のリソース占有へ波及する可能性があります。

これを防ぐため、ビジネス処理とDB保存にそれぞれ独立したタイムアウトを設定します。

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```

このタイムアウトにより、アーカイブ保存が遅延しても5秒以内に失敗として処理し、クライアントには実際の処理結果を返すFail-safe構造を実現できます。

![MongoDB Compassに保存されたバーティカルAI分析ログ](/assets/img/posts/2026-08-18-go-vertical-ai-swagger/vertical-swagger-02-jp.png)

*MongoDB Compassの`vertical_ai_db.inference_logs`コレクションで、ドメイン、入力テキスト、AI結果、生成時刻が保存されたドキュメントを確認できます。*

<br />

# 6. Swaggerドキュメントのビルドとランタイムテスト

## 6-1. OpenAPI仕様書のビルド
サーバーを起動する前に、コード上部のコメントを基にSwagger JSONとGo APIハンドラーを生成します。プロジェクトのルートディレクトリで実行します。

```sh
swag init
```

実行すると`docs/`サブディレクトリが作成され、`docs.go`、`swagger.json`、`swagger.yaml`が生成されます。

## 6-2. サーバーの起動
```sh
go run main.go
```

サーバーが正常に動作すると、MongoDB接続、インデックス作成、Ginのエンドポイント割り当てに関するログが表示されます。

![Go APIサーバーの起動とMongoDB保存ログ](/assets/img/posts/2026-08-18-go-vertical-ai-swagger/vertical-swagger-03-jp.png)

*サーバー起動後、Swaggerルーティング、`domain`インデックス作成、MongoDB保存、APIの`200`応答が正常に処理されたログです。*

<br />

# 7. Talend API Testerとcurlによる可視化検証

ブラウザーまたはCLIを使って、構築したAPIが動作するかをリアルタイムに検証します。

## 7-1. ブラウザーでSwagger UIを検証
Webブラウザーで次のアドレスを開きます。

```text
http://localhost:8080/swagger/index.html
```

APIエンドポイントの仕様一覧が表示されます。`POST /api/v1/vertical-ai/analyze`の`Try it out`を選択してJSON Bodyを入力し、通信結果を確認します。

![Swagger UIのTry it outによるAPI成功レスポンス](/assets/img/posts/2026-08-18-go-vertical-ai-swagger/vertical-swagger-01-jp.png)

*Swagger UIで`finance`ドメインの分析リクエストを実行し、HTTP 200と`saved: true`のレスポンスを確認しました。*

## 7-2. Talend API Testerとcurl CLIによる検証
Chrome拡張機能の**Talend API Tester**または標準CLIツールの`curl`でリクエストを送信できます。

```sh
# 1. API成功リクエストのテスト
curl -X POST http://localhost:8080/api/v1/vertical-ai/analyze \
     -H "Content-Type: application/json" \
     -d '{"domain": "finance", "text": "2026年第2四半期の営業利益分析を依頼"}'

# 2. 必須フィールドが欠落した異常リクエストのテスト
curl -X POST http://localhost:8080/api/v1/vertical-ai/analyze \
     -H "Content-Type: application/json" \
     -d '{"text": "ドメインが送信されていない例外ケース"}'
```

正常に受信すると、次のようなJSONレスポンスが返ります。

```json
{
  "status": "success",
  "result": "[finance]分野のコンテキストに基づくAIインサイト生成完了",
  "saved": true,
  "timestamp": "2026-08-03T15:24:32.482Z"
}
```

不正な形式や必須パラメーターの欠落がある場合は、検証によって`400 Bad Request`が返されます。

<br />

# 8. エンタープライズ環境へ移行するための設計強化

このガイドをさらに発展させ、プロダクションアーキテクチャへ移行するには、次の設計を検討します。

- **ドメイン許可リスト（Domain Allowlist）**: `domain`パラメーターをGinミドルウェアで検証し、未許可のドメインは早期に拒否します。
- **メッセージキューによる非同期アーカイブ**: トラフィック急増時にMongoDB保存がメイン処理の負荷にならないよう、KafkaやRabbitMQへ投入し、非同期ワーカーが処理する構成へ分離します。
- **データ保持ポリシー（TTL）の自動化**: MongoDBのTTLインデックスを設定し、90日を経過したAIログをバックグラウンドで削除します。

<br />

# 9. まとめ

バーティカルAIサービスの成功は、単にモデルを呼び出すことだけでは決まりません。**ドメイン別のトラフィックを体系的に処理し、データを明確に履歴化し、API開発チーム全体の協業基盤を構築できるか**が重要です。

今回の構成を基礎として、次の段階ではClaudeやOpenAIなどの外部LLM API連携、JWTによる権限制御ミドルウェアを追加し、プロダクションレベルの商用アーキテクチャへ発展させてください。

