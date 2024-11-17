---
layout: post
title: Nakama Server 作成期 - 1
categories: 
- Tech
tags:
- AWS
- Linux
- Go
- Docker
- Nakama
lang: jp
---

[Nakama설치](https://fkrhtmq123.github.io/jp/tech/2024/10/05/aws-docker-nakama/)で作った基本サーバーコードにマッチをするための機能を追加

まず, ルートフォルダに ```lobbyMatch.go``` ファイルを作成

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

LobbyMatch インターフェースは statusに LobbyMatchStateを設定<br />
match statusにはプレイヤーデータ、マッチデータ、チックに関する設定を設定します。

ここで TickRateとは、サーバーが１秒に状態を何回計算するかを決定する値です。<br />
```TickRate = 30``` にするばサーバーは１秒あたり１０回状態を計算します。

# TickRateが必要な理由
TickRateはゲームサーバーのリアルタイム同期化性能に直接影響を与えるためです。

TickRateの影響は下記になります。<br />
## 1. ゲームの応答性
・高いTickRateは早い反応が必要なゲーム(例: FPS, レーシングゲームなど)に適しています。<br />
・低いTickRateはターン制のゲームやマルチプレイヤーゲームに適しています。

## 2. ネットワーク帯域幅
・高いTickRateはもっと多いデータをクライアントに転送するためネットワーク帯域幅の使用量が増加します。<br />
・低いTickRateはネットワーク負荷を減らせますが、リアウタイム同期化の正確度が落ちる可能性があります。

## 3. サーバー性能
・TickRateが高いほどサーバー通信が頻繁に行い計算するためCPUとメモリーの使用量が増加します。<br />
・低いTickRateはサーバーの負荷を減らせますが、クライアントのゲームのプレイがスムーズに行かない可能性があります。

```sh
func CreateMatch(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule) (runtime.Match, error) {
	return &LobbyMatch{
		state: &LobbyMatchState{
			Players:   make(map[string]runtime.Presence),
			MatchData: make(map[string]interface{}),
			TickRate:  10, // １秒あたりのチック数
		},
	}, nil
}
```

上のコードは１秒あたりの１０回状態を計算するためのメソットで return の値が runtime.MatchのためMatchに関するメソットの設定が必要です。

```sh
// runtimeのMatchにあるメソットをlobbyMatch.goに作成します。
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

上のコードの中でMatchJoin, MatchLeaveなどプレイヤーがマッチに入る、出る時メッセージを送るようにします。

```sh
func encodeMessage(message map[string]interface{}) []byte {
	encoded, _ := json.Marshal(message)
	return encoded
}

// プレイヤーデータをfor文を回しメッセージを送ります。
// 例: プレイヤーが出る時
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

```lobbyMatch.go``` 作成完了後、 ```main.go``` の ```InitModule```にマッチ機能をモジュールとして設定

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