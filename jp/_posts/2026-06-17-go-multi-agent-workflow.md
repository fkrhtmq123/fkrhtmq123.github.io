---
layout: post
title: Goのゴルーチンとチャネルで実装するマルチエージェント（Multi-Agent）非同期ワークフロー
categories:
- Tech
tags:
- Go
- AI
- Concurrency
lang: jp
---

2026年のAIトレンドを見ると、もはや「モデルがどれほど賢いか」よりも、**「複数のタスクを最後まで連携させて実際の成果物を作り出せるか」**が重要な問いになっています。

特にバックエンドの観点では、一つの巨大なモデルを呼び出すよりも、役割が明確に分かれた複数のエージェントを束ねて、**検索 → 分析 → 整理 → 報告**を自動的に繋げる構造が、実務においてより適しているケースが多いです。Goは、このパターンを実装する上で非常に優れた言語です。ゴルーチンとチャネルのおかげで、タスクの分離と結果の収集が明快になり、パフォーマンスも軽量に維持できるためです。

この記事では、以下の内容を順に整理します。

1. なぜマルチエージェントワークフローが必要なのか
2. Goのゴルーチンとチャネルでどのようにオーケストレーションするのか
3. サンプルコードで注意すべきポイント
4. 実務への適用時、タイムアウト・キャンセル・障害処理までどのように拡張するか
5. 実際の実行結果の要約

<br />

# 1. マルチエージェントワークフローが必要な理由

単一のAI呼び出しだけで完結する業務は多くありません。実際には、以下のように複数の段階に分かれることが一般的です。

- ソース資料を収集する
- 収集した内容を要約する
- 別の観点から再検討（検証）する
- 最終結果を人間が読みやすい形式に整理する

このフローを一つの長い関数として書くこともできますが、その場合は以下のような問題が生じます。

- 役割が混ざり合い、メンテナンスが困難になる
- 一部のフェーズが遅延した際、フロー全体の制御が難しくなる
- 並行（並列）で処理できるタスクまで順番に実行することになる

Goの ゴルーチン（Goroutine）とチャネル（Channel）は、これらの問題を自然に解決します。各エージェントを独立した作業単位として分離し、結果だけを安全に集約して最終的な応答を作成できるためです。

<br />

# 2. 全体構造の要約

今回のサンプルは、以下の3つの役割で構成します。

- **Orchestrator**: 全体のフローを管理し、結果を取りまとめる
- **Search Agent**: 参考資料やソースデータを収集する
- **Analysis Agent**: 収集された情報を解析し、核心的なインサイトを生成する

フローは以下のように進行します。

1. リクエストが入るとオーケストレーターが起動する。
2. 検索エージェントと分析エージェントをゴルーチンで同時に実行する。
3. 各エージェントは結果をチャネルに送信する。
4. オーケストレーターはチャネルからすべての結果を受け取り、最終レポートを作成する。

核心は、**「タスクの実行は並行、結果の収集は単一の地点」**で行うことです。

<br />

# 3. サンプルコード

以下のコードは、学習用の最小限のサンプルです。実務では、ここに外部API呼び出し、キャッシュ、タイムアウト、再試行、ロギングなどを追加していきます。

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

	fmt.Printf("[Search Agent] %s 関連資料の収集を開始\n", topic)

	select {
	case <-time.After(2 * time.Second):
		ch <- AgentResult{
			AgentName: "SearchAgent",
			Content:   fmt.Sprintf("%s 関連の一次資料の収集が完了", topic),
			Success:   true,
		}
	case <-ctx.Done():
		ch <- AgentResult{
			AgentName: "SearchAgent",
			Content:   "検索処理がタイムアウトにより中断されました",
			Success:   false,
			Err:       ctx.Err(),
		}
	}
}

func StartAnalysisAgent(ctx context.Context, topic string, ch chan<- AgentResult, wg *sync.WaitGroup) {
	defer wg.Done()

	fmt.Printf("[Analysis Agent] %s の分析を開始\n", topic)

	select {
	case <-time.After(3 * time.Second):
		ch <- AgentResult{
			AgentName: "AnalysisAgent",
			Content:   fmt.Sprintf("%s に関する重要なインサイト3件の抽出が完了", topic),
			Success:   true,
		}
	case <-ctx.Done():
		ch <- AgentResult{
			AgentName: "AnalysisAgent",
			Content:   "分析処理がタイムアウトにより中断されました",
			Success:   false,
			Err:       ctx.Err(),
		}
	}
}

func RunOrchestration(topic string) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	fmt.Printf("=== [Orchestrator] '%s' のマルチエージェント分析を開始 ===\n", topic)
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
		fmt.Printf("[%s] 結果到着: %s\n", result.AgentName, result.Content)
		report = append(report, result)
	}

	elapsed := time.Since(start)
	fmt.Println("\n=== [Orchestrator] 最終レポート ===")
	fmt.Printf("総処理時間: %v\n", elapsed)

	for i, r := range report {
		fmt.Printf("%d. [%s] %s (success=%t)\n", i+1, r.AgentName, r.Content, r.Success)
	}
}

func main() {
	RunOrchestration("2026年ITトレンド分析")
}
```

<br />

![全体のコード構造](/assets/img/posts/2026-06-17-go-multi-agent-workflow/go-multi-01-jp.png)

<br />

<br />

# 4. コードにおける重要ポイント

上記のコードを実際に実行してみると、各エージェントが同時に起動し、最終結果をオーケストレーターが安全に収集する一連のフローを確認できます。

<br />

![正常実行結果](/assets/img/posts/2026-06-17-go-multi-agent-workflow/go-multi-02-jp.png)

<br />

## 4-1. ゴルーチンによる役割の分離
`go StartSearchAgent(...)` や `go StartAnalysisAgent(...)` のように、各役割を独立して実行することで、タスクが並行処理されます。

この構造には明確なメリットがあります。

- 各機能の責任（責務）が分離される
- 処理の遅いフェーズがあっても、全体の処理時間を短縮できる
- 将来的に新しいエージェントを追加しやすい

<br />

![ゴルーチン並行処理の比較](/assets/img/posts/2026-06-17-go-multi-agent-workflow/go-multi-03-jp.png)

<br />

## 4-2. チャネルは結果収集専用にする
`make(chan AgentResult, 2)` のようにバッファを持たせたチャネルを使用することで、処理結果を一時的に安全に保持できます。

バッファを設ける理由はシンプルです。

- エージェントが同時に終了しても、ブロッキングが発生しにくい
- オーケストレーターの処理が一瞬遅れても、結果を失わずに保持できる

## 4-3. WaitGroupによる終了タイミングの管理
チャネルのみを使用すると、終了するタイミングの管理が曖昧になりがちです。`sync.WaitGroup` を用いて、すべてのゴルーチンが完了した後にチャネルをクローズすることで、結果を読み取るループが正常に終了します。

このパターンを怠ると、以下の問題が発生する可能性があります。

- チャネルがクローズされず、無限に待機状態になる（デッドロック）
- すべての結果を受信する前にループが終了してしまう
- ゴルーチンリークの発生

## 4-4. contextによるタイムアウト制御
実務において最も重要な部分です。特定のエージェントの処理が遅延すると、パイプライン全体が停止してしまうため、`context.WithTimeout` を用いて最大待機時間を明示する必要があります。

今回のサンプルでは、5秒を超えると処理が中断されるように設定しています。

<br />

![コンテキストタイムアウト例外処理](/assets/img/posts/2026-06-17-go-multi-agent-workflow/go-multi-04-jp.png)

<br />

# 5. 実務における拡張ポイント

今回のサンプルはあくまで出発点であり、実際のプロダクション環境では以下のような機能を追加することをお勧めします。

- **リトライロジック**: 外部API呼び出しが失敗した際、1〜2回自動リトライする
- **優先度付きキュー**: 重要なエージェントを優先的に実行する
- **結果の順序制御**: エージェントごとの応答順序を固定し、レポート出力を安定化する
- **キャッシュ**: 同一トピックへのリクエストは結果をキャッシュして再利用する
- **モニタリング**: 実行時間やエラー率をメトリクスとして記録する

特にAI関連のパイプラインにおいては、「いかに早く回答を返すか」だけでなく、「失敗（タイムアウトやエラー）をいかに予測可能かつ安全に処理できるか」が極めて重要です。

<br />

# 6. 実行結果およびキャプチャの要約

この記事で使用されている実行結果のスクリーンショットは、以下のキービジュアルをキャプチャして適切なフローに配置しています。

1. **全体のコード構造** (`go-multi-01-jp.png`) - オーケストレーターと各エージェントの結合・連携関係
2. **正常実行結果** (`go-multi-02-jp.png`) - 2つのエージェントの完了タイミングとオーケストレーターの出力ログ
3. **ゴルーチン並行処理の比較** (`go-multi-03-jp.png`) - 順次実行と比較した際の、わずか3秒で完了するパフォーマンス面の優位性
4. **コンテキストタイムアウト例外処理** (`go-multi-04-jp.png`) - デッドラインを超過した際に、各エージェントが安全に終了される例外ハンドリングの挙動

<br />

# 7. よくある質問

## Q. この構成はシンプルすぎませんか？
シンプルですが、並行オーケストレーションの核心パターンは十分に網羅しています。実際の開発では、このパターンをベースにAPIサーバーやワーカーキューへとスケールアップしていくことになります。

## Q. なぜGoがこの構造に適しているのですか？
Goは並行処理モデル（CSP）がシンプルであり、ゴルーチンの起動コストが非常に低く、チャネルを用いたデータのやり取りが直感的に行えるためです。

## Q. 実際にはどのようなサービスに応用できますか？
- ドキュメント要約パイプライン
- 検索＋分析による自動レポート生成
- システムログのリアルタイム分類・解析
- 自律型AIエージェントのオーケストレーション

<br />

# 8. おわりに

マルチエージェント構造の核心は、「一つの巨大なAIを作る」ことではなく、**「仕事を小さな役割に分割し、それらの役割を安定的につなぎ合わせる」**ことにあります。

Goは、このパターンを最もクリーンに実装できる言語の一つです。ゴルーチンとチャネル、そして context を適切に組み合わせることで、実務に耐えうる非同期エージェントワークフローの強固な骨組みを構築することができます。

次のステップとして、この結果を MongoDB に保存し、Swagger（OpenAPI）を介して外部に公開するAPIへと拡張していくことで、実用的なバックエンドサービスが完成します。
