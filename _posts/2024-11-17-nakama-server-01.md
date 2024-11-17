---
layout: post
title: Nakama Server 제작기 - 1
categories: 
- Tech
tags:
- AWS
- Linux
- Go
- Docker
- Nakama
lang: ko
---

[Nakama설치](/_posts/2024-10-05-aws-docker-nakama.md)에서 만든 기본 서버 코드에 매치를 하는 기능을 추가

먼저, 루트 폴더에 ```lobbyMatch.go``` 파일 생성

```sh
package main

import (
	"context"
	"database/sql"
	"encoding/json"

	"github.com/heroiclabs/nakama-common/runtime"
)

type LobbyMatch struct {
	state *LobbyMatchState
}

type LobbyMatchState struct {
	Players   map[string]runtime.Presence `json:"players"`
	MatchData map[string]interface{}      `json:"match_data"`
	TickRate  int                         `json:"tick_rate"`
}
```

LobbyMatch 인터페이스는 status에 LobbyMatchState를 설정<br />
match status에는 플레이어 데이터, 매치 데이터, 틱 에 대한 설정을 설정합니다.

여기서 TickRate란, 서버가 초당 상태를 얼마나 자주 계산 할지를 결정하는 값을 말합니다.<br />
```TickRate = 30``` 라고 하면 서버는 초당 30번 상태를 계산합니다.

# TickRate가 필요한 이유
TickRate는 게임 서버에서 실시간 동기화 성능에 직접적인 영향을 주기 때문입니다.

TickRate의 영향은 밑에와 같습니다.<br />
## 1. 게임의 응답성
・높은 TickRate는 빠른 반응이 필요한 게임(예: FPS, 레이싱 게임 등)에 유리합니다.<br />
・낮은 TickRate는 턴 기반 게임이나 단순한 멀티 플레이어 게임에 적합합니다.

## 2. 네트워크 대역폭
・높은 TickRate는 더 많은 데이터를 클라이언트로 전송하므로 네트워트 대역폭 사용량이 증가 합니다.<br />
・낮은 TickRate는 네트워브 부하를 줄일 수 있지만, 실시간 동기화 정확도가 떨어질 수 있습니다.

## 3. 서버 성능
・TickRate가 높을 수록 서버에서 더 자주 계산을 수행해야 하므로 CPU와 메모리 사용량이 증가 합니다.<br />
・낮은 TickRate는 서버 부하를 줄이는데 유리하지만, 클라이언트의 게임 플레이가 부드럽지 않을 수 있습니다.

```sh
func CreateMatch(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule) (runtime.Match, error) {
	return &LobbyMatch{
		state: &LobbyMatchState{
			Players:   make(map[string]runtime.Presence),
			MatchData: make(map[string]interface{}),
			TickRate:  10, // 초당 틱 수
		},
	}, nil
}
```
위 코드는 초당 10번의 상태 계산하도록 설정한 함수이며 return 값이 runtime.Match이기 때문에 Match에 대한 함수를 설정해줘야합니다.

```sh
// runtime안의 Match에 있는 함수를 lobbyMatch.go에 작성합니다.
type Match interface {
	MatchInit(ctx context.Context, logger Logger, db *sql.DB, nk NakamaModule, params map[string]interface{}) (interface{}, int, string)
	MatchJoinAttempt(ctx context.Context, logger Logger, db *sql.DB, nk NakamaModule, dispatcher MatchDispatcher, tick int64, state interface{}, presence Presence, metadata map[string]string) (interface{}, bool, string)
	MatchJoin(ctx context.Context, logger Logger, db *sql.DB, nk NakamaModule, dispatcher MatchDispatcher, tick int64, state interface{}, presences []Presence) interface{}
	MatchLeave(ctx context.Context, logger Logger, db *sql.DB, nk NakamaModule, dispatcher MatchDispatcher, tick int64, state interface{}, presences []Presence) interface{}
	MatchLoop(ctx context.Context, logger Logger, db *sql.DB, nk NakamaModule, dispatcher MatchDispatcher, tick int64, state interface{}, messages []MatchData) interface{}
	MatchTerminate(ctx context.Context, logger Logger, db *sql.DB, nk NakamaModule, dispatcher MatchDispatcher, tick int64, state interface{}, graceSeconds int) interface{}
	MatchSignal(ctx context.Context, logger Logger, db *sql.DB, nk NakamaModule, dispatcher MatchDispatcher, tick int64, state interface{}, data string) (interface{}, string)
}
```

위 코드 중에 MatchJoin, MatchLeave 등 에 플레이어가 매치 가입을 하거나 떠날때는 메세지를 보내도록 합니다.

```sh
func encodeMessage(message map[string]interface{}) []byte {
	encoded, _ := json.Marshal(message)
	return encoded
}

// 플레이어 데이터를 반복문을 돌려 메세지는 보냅니다.
// 예: 플레이어가 떠날때
func (m *LobbyMatch) MatchLeave(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule, dispatcher runtime.MatchDispatcher, tick int64, state interface{}, presences []runtime.Presence) interface{} {
	matchState := state.(*LobbyMatchState)

    // 반복문 실행으로 메세지 송신
	for _, presence := range presences {
		delete(matchState.Players, presence.GetUserId())
		message := map[string]interface{}{
			"action":  "player_leave",
			"user_id": presence.GetUserId(),
		}
		dispatcher.BroadcastMessage(1, encodeMessage(message), nil, nil, true)
	}

	return matchState
}
```

```lobbyMatch.go``` 작성 완료 후, ```main.go``` 의 ```InitModule```에 매치 기능을 모듈로서 설정

```sh
func InitModule(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule, initialzer runtime.Initializer) error {
	initStart := time.Now()

	// Lobby Match추가
	err = initialzer.RegisterMatch("lobby", CreateMatch)
	if err != nil {
		logger.Error("Failed to register match: %v", err)
		return err
	}

	logger.Info("Module loaded in %dmx", time.Since(initStart).Milliseconds())

	return nil
}
```