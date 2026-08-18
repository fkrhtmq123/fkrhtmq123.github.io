---
layout: post
title: Go + MongoDB + Swagger를 활용한 버티컬 AI API 서버 구축 및 문서화
categories:
- Tech
tags:
- Go
- MongoDB
- Swagger
- API
lang: ko
---

버티컬 AI는 단순한 "AI 기술을 이식하는 실험" 단계를 넘어 비즈니스 백엔드에 통합되고 있습니다. 금융, 의료, 리서치처럼 특정 도메인이 뚜렷한 영역일수록 유연한 스키마 기반의 실시간 추적 이력 로깅과 신뢰성 높은 명세서 관리가 시스템 안정성의 핵심이 됩니다.

이번 글에서는 Go 언어의 HTTP 프레임워크인 Gin을 활용하여 버티컬 AI 처리 과정을 수행하고, 인프라의 확장성 확보를 위해 결과를 MongoDB에 영구 기록하며, 프론트엔드 및 기획 부서와의 협업을 위한 Swagger 문서 자동화까지 이루어지는 설계 및 구현 방법을 다룹니다.

<br />

# 1. 왜 이 조합이 유용한가

## 1-1. Go (Golang)
Go는 단순한 동시성 모델(Goroutine)과 빠른 컴파일 속도를 제공합니다. AI API 백엔드와 같이 다량의 비동기 네트워크 I/O 및 데이터 스트리밍 연산이 반복적으로 발생하는 아키텍처에서 고정된 자원만으로 고성능을 제공하며 가벼운 일체형 컨테이너 배포가 가능해집니다.

## 1-2. MongoDB
버티컬 AI가 반환하는 원문 텍스트, 구조화된 인덱스 정보, 토큰 소비 로그, 디버그용 원시 응답(Raw Response) 등은 비정형과 반정형 데이터가 결합되어 있습니다. MongoDB는 스키마의 변경 없이도 이러한 복잡한 추적 로그 문서를 다차원 데이터 형태로 유연하고 빠르게 영구 수집하기에 적합한 데이터 저장소입니다.

## 1-3. Swagger (OpenAPI)
버티컬 AI 프로젝트는 여러 유관 부서가 지속적으로 스펙을 맞춰야 하는 영역입니다. Swagger 문서를 소스코드 주석 형태로 직접 빌드하면 사후 문서 불일치(Docs-Drift) 현상을 방지하고, 프론트엔드 파트가 직접 테스트할 수 있는 실습 보드를 즉각 공유할 수 있습니다.

<br />

# 2. 프로젝트 구조 설계

실무적인 파일 분리가 용이하도록 다음과 같은 프로젝트 폴더 트리를 기준으로 구성을 설정합니다.

```text
my-vertical-ai/
├── main.go          # 메인 패키지 (라우터 및 라이프사이클 구성)
├── docs/            # swag init에 의해 자동 생성되는 Swagger 빌드 명세 파일들
├── go.mod           # Go 모듈 관리 정의 파일
├── go.sum           # 외부 패키지 해시 무결성 검증용
└── .env             # 환경 변수 정의용 로컬 파일
```

<br />

# 3. 개발 의존성 패키지 설치

터미널을 열고 필요한 프레임워크 패키지와 드라이버를 설치합니다.

```sh
# 1. Gin 웹 프레임워크 설치
go get github.com/gin-gonic/gin

# 2. MongoDB 공식 드라이버 설치
go get go.mongodb.org/mongo-driver/mongo

# 3. Swagger 통합을 위한 swaggo 패키지 라이브러리 설치
go get github.com/swaggo/gin-swagger
go get github.com/swaggo/files
```

선언형 코드 주석을 OpenAPI JSON 명세서로 컴파일해주는 `swag` CLI 도구를 설치합니다. CLI 도구는 `go install` 명령어를 사용하여 전역에 설정합니다.

```sh
go install github.com/swaggo/swag/cmd/swag@latest
```

<br />

# 4. 실무형 Go 백엔드 통합 코드 구현

아래는 환경 변수 fallback 처리, 도메인 필드 단일 인덱스 비동기 튜닝, 그리고 Swagger CLI 컴파일 시 자동으로 앤드포인트를 매핑해주는 OpenAPI 2.0 선언형 데코레이터 주석이 적용된 `main.go` 통합 소스코드입니다.

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

	_ "my-vertical-ai/docs" // swag init으로 빌드 완료된 docs 패키지를 강제 동반 임포트
)

// @title           Vertical AI API Server
// @version         1.0
// @description     특정 도메인에 특화된 버티컬 AI 처리를 수행하고 기록하는 API 서버입니다.
// @host            localhost:8080
// @BasePath        /api/v1

var logCollection *mongo.Collection

// AIRequest 도메인 분석 요청 모델 명세
type AIRequest struct {
	Domain string `json:"domain" binding:"required" example:"finance" doc:"도메인 분류 (예: finance, medical)"`
	Text   string `json:"text" binding:"required" example:"2026년 2분기 영업이익 분석 요청" doc:"분석 대상 원문 텍스트"`
}

// AIResponse 도메인 분석 완료 응답 모델 명세
type AIResponse struct {
	Status    string    `json:"status" example:"success" doc:"처리 상태"`
	Result    string    `json:"result" example:"[finance] 분야 컨텍스트 기반 AI 인사이트 생성 완료" doc:"AI 분석 결과"`
	Saved     bool      `json:"saved" example:"true" doc:"MongoDB 로그 저장 여부"`
	Timestamp time.Time `json:"timestamp" doc:"응답 생성 시각"`
}

func main() {
	// 1. 환경 변수를 기반으로 접속 정보 수립 (Docker/Cloud 배포 용이성 극대화)
	mongoURI := os.Getenv("MONGODB_URI")
	if mongoURI == "" {
		mongoURI = "mongodb://localhost:27017"
	}

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// MongoDB 드라이버 연결 수립
	client, err := mongo.Connect(ctx, options.Client().ApplyURI(mongoURI))
	if err != nil {
		log.Fatal("🚨 MongoDB 연결 실패: ", err)
	}

	// 핑 테스트를 통한 실시간 커넥션 상태 확인
	if err := client.Ping(ctx, nil); err != nil {
		log.Fatal("🚨 MongoDB Ping 테스트 실패: ", err)
	}

	dbName := os.Getenv("MONGODB_DB")
	if dbName == "" {
		dbName = "vertical_ai_db"
	}

	logCollection = client.Database(dbName).Collection("inference_logs")

	// 2. 도메인(domain) 필드 단일 인덱스 비동기 빌드 (백엔드 기동 스레드 지연 방지)
	go func() {
		idxCtx, idxCancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer idxCancel()
		_, err := logCollection.Indexes().CreateOne(idxCtx, mongo.IndexModel{
			Keys: bson.D{{Key: "domain", Value: 1}},
		})
		if err != nil {
			log.Println("⚠️ domain 인덱스 빌딩 예외 발생: ", err)
		} else {
			log.Println("✅ domain 필드 단일 인덱스 생성 완료")
		}
	}()

	r := gin.Default()

	// 3. Swagger 문서 연동 API 라우터 매핑
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	// 4. 비즈니스 핵심 분석 라우터 등록
	r.POST("/api/v1/vertical-ai/analyze", AnalyzeVerticalAI)

	log.Println("🚀 API 라우팅 할당 완료. 8080 포트에서 기동 대기 중...")
	r.Run(":8080")
}

// AnalyzeVerticalAI Go Gin Handler
// @Summary      버티컬 AI 도메인 분석 수행
// @Description  클라이언트로부터 유효 도메인과 비정형 텍스트를 전달받아 분야 특화 분석 결과를 생성하고, 해당 이력을 MongoDB에 영구 로깅합니다.
// @Tags         Vertical AI
// @Accept       json
// @Produce      json
// @Param        request body AIRequest true "분석을 위한 입력값 정의 모델"
// @Success      200 {object} AIResponse "AI 분석 결과 도출 및 로그 DB 기록 완수"
// @Failure      400 {object} map[string]string "요청 본문이 올바르지 않은 잘못된 JSON 형식 구조"
// @Router       /vertical-ai/analyze [post]
func AnalyzeVerticalAI(c *gin.Context) {
	var req AIRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "올바른 JSON 요청이 아닙니다. 필수 파라미터가 누락되었거나 타입이 다릅니다."})
		return
	}

	// [실무 참고]: 실제 LLM 혹은 파인튜닝된 온프레미스 AI 서비스의 API 호출 로직이 이곳에 결합됩니다.
	resultText := "[" + req.Domain + "] 분야 컨텍스트 기반 AI 인사이트 생성 완료"

	// BSON Document 구조 수립
	logDoc := bson.M{
		"domain":    req.Domain,
		"text":      req.Text,
		"result":    resultText,
		"createdAt": time.Now(),
	}

	// 5초 타임아웃 컨텍스트 분배를 통해 DB 오버헤드 전파 예방
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// MongoDB 로그 기록 처리
	insertResult, insertErr := logCollection.InsertOne(ctx, logDoc)
	if insertErr != nil {
		log.Println("🚨 MongoDB 이력 기록 도중 런타임 예외 발생: ", insertErr)
	} else {
		log.Printf("💾 MongoDB 저장 완수 (InsertedID: %v)", insertResult.InsertedID)
	}

	// 비즈니스 로직과 데이터 적재 단계를 결합하되, 데이터 적재의 일시적 오류가 비즈니스 자체의 중단으로 이어지지 않도록 방어 설계
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

# 5. 핵심 아키텍처 및 코드 심층 분석

## 5-1. 왜 Swagger 주석 데코레이터 선언이 필수적인가?
기존에 많은 백엔드 가이드에서 단순히 `swag init` 명령어 기동 프로세스만을 소개하고 실제 메서드 핸들러 상단에 OpenAPI 어노테이션 주석을 생략하는 실수를 저지릅니다. 

주석에 명시된 `@Summary`, `@Description`, `@Param`, `@Success`, `@Router` 지시자는 `swag` CLI 도구가 파싱하는 기계적 선언부입니다. 이 주석을 기반으로 프로젝트 내에 정밀한 `swagger.json` 스키마가 그려지며, 이를 생략할 경우 Swagger UI 접속은 되나 리스트에 어떠한 엔드포인트 API 스펙도 로드되지 않는 명세 누락이 발생합니다.

## 5-2. 비동기 인덱스 빌드 및 백그라운드 처리 기법
도메인별로 이력을 분석하고 조회하는 쿼리가 잦은 버티컬 AI 시스템 특성상, 데이터가 급증하면 `domain` 필드의 풀스캔 병목이 심각해집니다. 이를 미리 방어하고자 DB 적재 초기화 라이프사이클 안에서 고루틴을 선언하여 `domain` 인덱스를 빌드했습니다. 

```go
go func() { ... }()
```
이러한 백그라운드 로직을 활용해 메인 메커니즘인 Gin 기동 스레드가 인덱스 생성을 위해 대기(Blocking)하느라 서버 초기 활성화 시간이 지연되는 현상을 안전하게 방지할 수 있습니다.

## 5-3. MongoDB 작업용 전용 Context Isolation 전략
네트워크 타임아웃 처리가 느슨할 경우 MongoDB 커넥션 지연이 그대로 전체 API 요청 쓰레드 점유로 전파되어 시스템 전체 데드락을 유발할 수 있습니다. 

이를 예방하기 위해 다음과 같이 비즈니스 핵심 흐름과 DB 적재를 위해 각각 격리된 독립 타임아웃을 수립하는 설계가 필요합니다.
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```
이 타임아웃 처리를 통해 데이터 아카이브 기록 지연 시 5초 이내에 실패를 신속하게 처리하고, 사용자(클라이언트)에게는 실제 처리 결과 응답을 수급하는 Fail-safe 구조를 안전하게 설계했습니다.

![MongoDB Compass에 저장된 버티컬 AI 분석 로그](/assets/img/posts/2026-08-18-go-vertical-ai-swagger/vertical-swagger-02-kr.png)

*MongoDB Compass의 `vertical_ai_db.inference_logs` 컬렉션에서 도메인, 입력 텍스트, AI 결과, 생성 시각이 저장된 문서를 확인할 수 있습니다.*

<br />

# 6. Swagger 문서 빌드 및 런타임 테스트

## 6-1. OpenAPI 명세서 빌드
서버를 작동시키기 전, 코드 상단에 주석으로 기록된 명세 명칭들을 기반으로 Swagger JSON 및 Go API 핸들러들을 일체 빌드합니다. 프로젝트 루트 디렉터리 내에서 다음과 같이 입력합니다.

```sh
swag init
```
이 작업을 거치면 프로젝트 내에 자동으로 `docs/` 서브 디렉터리가 활성화되며 `docs.go`, `swagger.json`, `swagger.yaml` 리소스가 정상 컴파일 및 도킹 배치됩니다.



## 6-2. 서버 구동
```sh
go run main.go
```
서버가 정상적으로 동작하면 콘솔 창에 MongoDB 연결 로그, 인덱싱 완료 트리거 및 Gin의 엔드포인트 할당 이정표가 출력됩니다.

![Go API 서버 실행 및 MongoDB 저장 로그](/assets/img/posts/2026-08-18-go-vertical-ai-swagger/vertical-swagger-03-kr.png?v=2cf81f8)

*서버 실행 후 Swagger 라우팅, `domain` 인덱스 생성, MongoDB 저장 및 API `200` 응답이 정상적으로 처리된 로그입니다.*

<br />

# 7. Talend API Tester와 curl을 통한 시각화 검증

브라우저 혹은 CLI를 사용하여 수립한 API 가 동작하는지 실시간 유효성 테스트를 전개합니다.

## 7-1. 브라우저 Swagger UI 검증
웹 브라우저를 열고 다음 주소에 진입합니다.

```text
http://localhost:8080/swagger/index.html
```

수립한 API 엔드포인트 명세 목록이 표출됩니다. `POST /api/v1/vertical-ai/analyze`의 `Try it out` 버튼을 선택하고 JSON Body를 입력해 성공적으로 통신되는지 직접 시각적으로 관측할 수 있습니다.

![Swagger UI Try it out을 이용한 API 성공 응답](/assets/img/posts/2026-08-18-go-vertical-ai-swagger/vertical-swagger-01-kr.png)

*Swagger UI에서 `finance` 도메인 분석 요청을 실행한 결과, HTTP 200과 `saved: true` 응답을 확인했습니다.*

## 7-2. 크롬 Talend API Tester 및 curl CLI 검증
별도의 API 툴 설치 없이도 크롬 브라우저에서 편리하게 구동할 수 있는 확장 프로그램인 **Talend API Tester** 또는 표준 CLI 명령어 도구인 `curl`을 통해 실시간 통신을 트리거합니다.

```sh
# 1. API 성공 요청 테스트 (정상적인 JSON 전송)
curl -X POST http://localhost:8080/api/v1/vertical-ai/analyze \
     -H "Content-Type: application/json" \
     -d '{"domain": "finance", "text": "2026년 2분기 영업이익 분석 요청"}'

# 2. 필수 필드가 누락된 잘못된 형식의 오류 차단 테스트
curl -X POST http://localhost:8080/api/v1/vertical-ai/analyze \
     -H "Content-Type: application/json" \
     -d '{"text": "도메인이 전송되지 않은 예외 케이스"}'
```



정상 수신 시 서버는 아래와 같이 JSON 처리 성공 결과와 MongoDB 아카이브 성공 마킹 여부를 탑재한 응답 메시지를 반환합니다.

```json
{
  "status": "success",
  "result": "[finance] 분야 컨텍스트 기반 AI 인사이트 생성 완료",
  "saved": true,
  "timestamp": "2026-08-03T15:24:32.482Z"
}
```

잘못된 형식이나 필수 파라미터가 유실되었을 경우에는 미들웨어 및 바인딩 엔진의 자체 검증을 통해 `400 Bad Request`와 함께 적합한 오류 피드백 메시지가 전달됩니다.

<br />

<br />

# 8. 엔터프라이즈 환경 전환을 위한 설계 보강 제안

본 가이드를 한 단계 더 확장하여 고도의 마이크로서비스 프로덕션 아키텍처로 승격하려면 다음 세 가지 설계를 추가로 검토해야 합니다.

- **도메인 허용 목록 필터(Domain Allowlist)**: 들어오는 `domain` 파라미터의 남용을 차단하기 위해, 화이트리스트 검사 슬라이스 기능을 Gin 미들웨어로 수립하여 비인가 도메인 접근 시 요청의 조기 반환(Early Return) 처리 구현.
- **메시지 큐(Message Queue) 연계 비동기 아카이빙**: 트래픽의 스파이크 상태 발생 시 MongoDB 기록 작업 자체가 주 스레드의 오버헤드가 되지 않도록, 요청은 Kafka 혹은 RabbitMQ에 넣은 뒤 비동기 워커가 구독 처리하는 완전 분리형 아키텍처로 이관.
- **데이터 보존 정책 (TTL) 자동화 설정**: 무한대로 불어날 수 있는 AI 원문 로깅 적재 비용 낭비를 절감하기 위해, MongoDB의 TTL(Time-To-Live) 인덱스 기능을 지정하여 90일 경과 로그는 백그라운드에서 주기적으로 정리 소멸되도록 설계.

<br />

# 9. 마치며

버티컬 AI 서비스 아키텍처의 설계 성공 여부는 단순히 모델을 호출하는 수준을 넘어, **들어오는 도메인 트래픽을 체계적으로 핸들링하고, 데이터를 명료하게 이력화하며, 전체 API 개발 협업의 연결망을 구축하는가**에 달려 있습니다.

오늘 수립한 구조를 기초로 하여, 다음 아키텍처 구성 단계에서는 실제 외부 LLM API 서비스(Claude, OpenAI 등) 연동 인터페이스 및 권한 제어 JWT 검증 미들웨어를 붙여 가며 프로덕션 레벨의 상용 아키텍처를 전개해 보시길 바랍니다.
