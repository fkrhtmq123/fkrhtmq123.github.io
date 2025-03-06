---
layout: post
title: Go CORS設定
categories: 
- Tech
tags:
- Go
lang: jp
---

サーバーを作りコードをサーバーを上げるとCORSエラー発生しますが、エラーを無くすためCORS設定をする方法を作成します。

# 1. CORSとは?
CORS(Cross-origin resource sharing) <br />
クロスオリジンリソース共有はWEBページ上の制限されたリソースを最初リソースがサービスされたドメインの外の他のドメインからの要請をできるようにする仕組みです。<br />
<出典 : [위키백과](https://ko.wikipedia.org/wiki/%EA%B5%90%EC%B0%A8_%EC%B6%9C%EC%B2%98_%EB%A6%AC%EC%86%8C%EC%8A%A4_%EA%B3%B5%EC%9C%A0#cite_note-mozhacks_cors-1)> <br />
上記の出典を翻訳したものです。
<br /><br />
簡単に言うと <br />
APIドメイン : https://example.com <br />
クライアント(WEB) : http://localhost:8080 <br /><br />
クライアントドメインからAPIドメインを呼び出した場合、ここでドメインが違うためCORSエラーが出ます。

<br />

# 2. CORS設定方法
```sh
go get github.com/gin-contrib/cors
```
CORSライブラリをインストールします。<br />

そして、下記のようにコードを書くとCORS設定完了。

```sh
package main

import (
	"log"
	"net/http"

	"github.com/gin-contrib/cors"
)

func main() {
	// Gin ラウター初期化
	r := gin.Default()

	// CORS 設定
	r.Use(cors.New(cors.Config{
        // 全てのドメインからの要請を許可
		AllowOrigins:     []string{"*"},
        // セキュリティが必要な場合、下記のように * からと特定のドメインのみ許可
		// AllowOrigins:  []string{"http://example.com", "http://localhost:3000"},
        // 許可するHTTPメソッド
		AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "PATCH"}, 
        // 許可するヘッダー
		AllowHeaders:     []string{"Origin", "Content-Type", "Accept"},
        // 資格証明（クッキーなの）を許可するか
		AllowCredentials: true,
	}))

	// ✅ Gin ラウター実行 (http.ListenAndServe の代わりに使用)
	if err := r.Run(":" + *port); err != nil {
		log.Fatal("Failed to start server:", err)
	}
}
```