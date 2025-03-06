---
layout: post
title: Go CORS설정
categories: 
- Tech
tags:
- Go
lang: ko
---

서버를 만들고 코드를 서버에 올리면 CORS에러가 나는데 에러를 없애기 위해 CORS설정을 하는 방법을 작성하겠습니다.

# 1. CORS란?
CORS(Cross-origin resource sharing) <br />
교차 출처 자원 공유는 웹 페이지 상의 제한된 리소스를 최초 자원이 서비스된 도메인 밖의 다른 도메인으로부터 요청할 수 있게 허용하는 구조입니다. <br />
<출처 : [위키백과](https://ko.wikipedia.org/wiki/%EA%B5%90%EC%B0%A8_%EC%B6%9C%EC%B2%98_%EB%A6%AC%EC%86%8C%EC%8A%A4_%EA%B3%B5%EC%9C%A0#cite_note-mozhacks_cors-1)> <br /><br />
간단하게 말하면 <br />
API주소 : https://example.com <br />
클라이언트(WEB) : http://localhost:8080 <br /><br />
클라이언트 주소에서 API주소를 사용할 경우 도메인 다르기 때문에 여기서 CORS 에러가 납니다.

<br />

# 2. CORS설정 방법
```sh
go get github.com/gin-contrib/cors
```
CORS라이브러리를 설치합니다. <br />

그리고 밑에와 같이 코드를 입력하면 CORS설정 완료.

```sh
package main

import (
	"log"
	"net/http"

	"github.com/gin-contrib/cors"
)

func main() {
	// Gin 라우터 초기화
	r := gin.Default()

	// CORS 설정
	r.Use(cors.New(cors.Config{
        // 모든 도메인에서의 요청을 허용
		AllowOrigins:     []string{"*"},
        // 보안 필요 시 특정 도메인만 허용
		// AllowOrigins:  []string{"http://example.com", "http://localhost:3000"},
        // 허용할 HTTP 메서드
		AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "PATCH"}, 
        // 허용할 헤더
		AllowHeaders:     []string{"Origin", "Content-Type", "Accept"},
        // 자격 증명(쿠키 등)을 허용할지 여부
		AllowCredentials: true,
	}))

	// ✅ Gin 라우터 실행 (http.ListenAndServe 대신 사용)
	if err := r.Run(":" + *port); err != nil {
		log.Fatal("Failed to start server:", err)
	}
}
```