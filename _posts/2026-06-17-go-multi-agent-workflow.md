---
layout: post
title: Go의 고루틴과 채널로 구현하는 다중 에이전트(Multi-Agent) 비동기 워크플로우
categories:
- Tech
tags:
- Go
- AI
- Concurrency
lang: ko
---

2026년 AI 흐름을 보면, 이제 중요한 질문은 “모델이 얼마나 똑똑한가”보다 **“여러 작업을 끝까지 연결해 실제 결과를 만들어낼 수 있는가”**에 가깝습니다.

특히 백엔드 관점에서는 하나의 거대한 모델보다, 역할이 분리된 여러 에이전트를 묶어 **검색 → 분석 → 정리 → 보고**를 자동으로 이어주는 구조가 실무에 더 잘 맞는 경우가 많습니다. Go는 이 패턴을 구현하기에 아주 좋은 언어입니다. 고루틴과 채널 덕분에 작업 분리와 결과 수집이 명확하고, 성능도 가볍게 유지할 수 있기 때문입니다.

이 글에서는 다음 내용을 순서대로 정리합니다.

1. 다중 에이전트 워크플로우가 왜 필요한지
2. Go에서 고루틴과 채널로 어떻게 오케스트레이션하는지
3. 예제 코드에서 주의해야 할 포인트
4. 실무 적용 시 타임아웃, 취소, 장애 처리까지 어떻게 확장할지
5. 직접 캡처하면 좋은 화면은 무엇인지

<br />

# 1. 다중 에이전트 워크플로우가 필요한 이유

단일 AI 호출로 끝나는 업무는 많지 않습니다. 실제로는 아래처럼 단계가 나뉘는 경우가 많습니다.

- 원천 자료를 수집해야 한다
- 수집한 내용을 요약해야 한다
- 다른 관점에서 재검토해야 한다
- 최종 결과를 사람이 읽기 좋은 형태로 정리해야 한다

이 흐름을 하나의 긴 함수로 작성할 수도 있지만, 그러면 다음 문제가 생깁니다.

- 역할이 섞여서 유지보수가 어려워짐
- 일부 단계가 느려져도 전체 흐름을 제어하기 어려움
- 병렬로 처리할 수 있는 작업까지 순차 실행하게 됨

Go의 고루틴과 채널은 이런 문제를 자연스럽게 풀어 줍니다. 각 에이전트를 독립된 작업 단위로 분리하고, 결과만 안전하게 모아 최종 응답을 만들 수 있기 때문입니다.

<br />

# 2. 전체 구조 요약

이번 예제는 다음 3개의 역할로 구성합니다.

- **Orchestrator**: 전체 흐름을 관리하고 결과를 취합
- **Search Agent**: 참고 자료나 원천 데이터를 수집
- **Analysis Agent**: 수집된 정보를 해석하고 핵심 인사이트 생성

흐름은 이렇게 진행됩니다.

1. 요청이 들어오면 오케스트레이터가 시작된다.
2. 검색 에이전트와 분석 에이전트를 고루틴으로 동시에 실행한다.
3. 각 에이전트는 결과를 채널에 보낸다.
4. 오케스트레이터는 채널에서 결과를 모두 받아 최종 리포트를 만든다.

핵심은 **작업 실행은 병렬**, **결과 취합은 단일 지점**으로 가져가는 것입니다.

<br />

# 3. 예제 코드

아래 코드는 학습용 최소 예제입니다. 실무에서는 여기에 외부 API 호출, 캐시, 타임아웃, 재시도, 로깅을 추가하면 됩니다.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

type AgentResult struct {
	AgentName string
	Content   string
	Success   bool
	Err       error
}

func StartSearchAgent(ctx context.Context, topic string, ch chan<- AgentResult, wg *sync.WaitGroup) {
	defer wg.Done()

	fmt.Printf("[Search Agent] %s 관련 자료 수집 시작\n", topic)

	select {
	case <-time.After(2 * time.Second):
		ch <- AgentResult{
			AgentName: "SearchAgent",
			Content:   fmt.Sprintf("%s 관련 원천 자료 수집 완료", topic),
			Success:   true,
		}
	case <-ctx.Done():
		ch <- AgentResult{
			AgentName: "SearchAgent",
			Content:   "검색 작업이 타임아웃으로 중단됨",
			Success:   false,
			Err:       ctx.Err(),
		}
	}
}

func StartAnalysisAgent(ctx context.Context, topic string, ch chan<- AgentResult, wg *sync.WaitGroup) {
	defer wg.Done()

	fmt.Printf("[Analysis Agent] %s 분석 시작\n", topic)

	select {
	case <-time.After(3 * time.Second):
		ch <- AgentResult{
			AgentName: "AnalysisAgent",
			Content:   fmt.Sprintf("%s에 대한 핵심 인사이트 3개 도출 완료", topic),
			Success:   true,
		}
	case <-ctx.Done():
		ch <- AgentResult{
			AgentName: "AnalysisAgent",
			Content:   "분석 작업이 타임아웃으로 중단됨",
			Success:   false,
			Err:       ctx.Err(),
		}
	}
}

func RunOrchestration(topic string) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	fmt.Printf("=== [Orchestrator] '%s' 다중 에이전트 분석 시작 ===\n", topic)
	start := time.Now()

	resultChan := make(chan AgentResult, 2)
	var wg sync.WaitGroup

	wg.Add(2)
	go StartSearchAgent(ctx, topic, resultChan, &wg)
	go StartAnalysisAgent(ctx, topic, resultChan, &wg)

	go func() {
		wg.Wait()
		close(resultChan)
	}()

	var report []AgentResult
	for result := range resultChan {
		fmt.Printf("[%s] 결과 도착: %s\n", result.AgentName, result.Content)
		report = append(report, result)
	}

	elapsed := time.Since(start)
	fmt.Println("\n=== [Orchestrator] 최종 리포트 ===")
	fmt.Printf("총 처리 시간: %v\n", elapsed)

	for i, r := range report {
		fmt.Printf("%d. [%s] %s (success=%t)\n", i+1, r.AgentName, r.Content, r.Success)
	}
}

func main() {
	RunOrchestration("2026 IT 트렌드 분석")
}
```

<br />

![전체 코드 구조](/assets/img/posts/2026-06-17-go-multi-agent-workflow/go-multi-01-kr.png)

<br />

<br />

# 4. 코드에서 꼭 봐야 할 포인트

위 코드를 직접 실행해 보면 각 에이전트가 동시에 시작되고, 최종 결과를 오케스트레이터가 안전하게 수집하는 전체 과정을 확인할 수 있습니다.

<br />

![정상 실행 결과](/assets/img/posts/2026-06-17-go-multi-agent-workflow/go-multi-02-kr.png)

<br />

## 4-1. 고루틴으로 역할을 분리한다
`go StartSearchAgent(...)`, `go StartAnalysisAgent(...)`처럼 각 역할을 독립 실행하면 작업이 병렬로 돌아갑니다.

이 구조의 장점은 명확합니다.

- 각 기능의 책임이 분리됨
- 느린 단계가 있어도 전체 처리 시간을 줄일 수 있음
- 나중에 에이전트를 추가하기 쉬움

<br />

![고루틴 병렬 처리 비교](/assets/img/posts/2026-06-17-go-multi-agent-workflow/go-multi-03-kr.png)

<br />

## 4-2. 채널은 결과 수집 전용으로 둔다
`make(chan AgentResult, 2)`처럼 버퍼를 둔 채널을 사용하면, 작업 결과를 임시로 안전하게 보관할 수 있습니다.

버퍼를 두는 이유는 간단합니다.

- 에이전트가 동시에 끝나도 블로킹이 줄어듦
- 오케스트레이터가 잠깐 늦어져도 결과를 보관할 수 있음

## 4-3. WaitGroup으로 종료 시점을 관리한다
채널만 쓰면 종료 타이밍이 애매해질 수 있습니다. `sync.WaitGroup`으로 모든 고루틴이 끝난 뒤 채널을 닫아야 루프가 정상 종료됩니다.

이 패턴을 놓치면 다음 문제가 생깁니다.

- 채널이 닫히지 않아 무한 대기
- 결과를 다 받기 전에 루프가 끝남
- 고루틴 누수 발생

## 4-4. context로 타임아웃을 건다
실무에서는 가장 중요한 부분입니다. 한 에이전트가 늦어지면 전체 파이프라인이 멈출 수 있으므로, `context.WithTimeout`으로 최대 대기 시간을 명시해야 합니다.

이 예제에서는 5초를 넘기면 작업이 중단되도록 했습니다.

<br />

![컨텍스트 타임아웃 예외 처리](/assets/img/posts/2026-06-17-go-multi-agent-workflow/go-multi-04-kr.png)

<br />

# 5. 실무에서 더 확장할 수 있는 부분

이 예제는 시작점이고, 실제 서비스에서는 아래 기능이 추가되면 좋습니다.

- **재시도 로직**: 외부 API 실패 시 1~2회 재시도
- **우선순위 큐**: 중요한 에이전트부터 먼저 실행
- **결과 정렬**: 에이전트별 응답 순서를 정해 리포트 안정화
- **캐시**: 같은 topic 요청은 재사용
- **모니터링**: 실행 시간과 실패율 기록

특히 AI 관련 파이프라인은 “정답을 빨리 반환하는 것”보다 “실패를 예측 가능하게 다루는 것”이 중요합니다.

<br />

# 6. 실행 결과 및 캡처 요약

이 글에 활용된 모든 실행 결과 스크린샷은 다음 핵심 화면들을 캡처하여 적절한 흐름에 배치했습니다.

1. **전체 코드 구조** (`go-multi-01-kr.png`) - 오케스트레이터와 에이전트들의 결합 관계
2. **정상 실행 결과** (`go-multi-02-kr.png`) - 두 에이전트의 완료 시점과 오케스트레이터의 출력 로그
3. **고루틴 병렬 처리 비교** (`go-multi-03-kr.png`) - 순차 처리 대비 단 3초만에 처리가 끝나는 성능 이점
4. **컨텍스트 타임아웃 예외 처리** (`go-multi-04-kr.png`) - 데드라인을 초과했을 때 각 에이전트가 안전하게 종료되는 예외 상황

<br />

# 7. 자주 묻는 질문

## Q. 이 구조는 너무 작은 예제 아닌가요?
작지만 핵심 패턴은 충분히 담고 있습니다. 실제 글에서는 이 예제를 기반으로 API 서버나 워커 큐까지 확장하는 과정을 붙이면 됩니다.

## Q. 왜 Go가 이런 구조에 잘 맞나요?
Go는 동시성 모델이 단순하고, 고루틴 비용이 낮고, 채널로 결과를 관리하기 쉽기 때문입니다.

## Q. 실제로는 어떤 서비스에 응용할 수 있나요?
- 문서 요약 파이프라인
- 검색 + 분석 자동 리포트
- 운영 로그 분류
- AI 에이전트 오케스트레이션

<br />

# 8. 마무리

다중 에이전트 구조의 핵심은 “AI를 하나 크게 만드는 것”이 아니라 **일을 작은 역할로 나누고, 그 역할을 안정적으로 연결하는 것**입니다.

Go는 이 패턴을 가장 깔끔하게 구현할 수 있는 언어 중 하나입니다. 고루틴과 채널, 그리고 context를 잘 조합하면 실무용 비동기 에이전트 워크플로우의 기본 뼈대를 충분히 만들 수 있습니다.

다음 단계로는 이 결과를 MongoDB에 저장하고, Swagger로 외부에 노출하는 형태로 이어가면 하나의 완성된 백엔드 흐름이 됩니다.
