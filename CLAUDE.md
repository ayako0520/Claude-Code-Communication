# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Agent Communication System

複数のClaude Codeエージェントがtmuxペイン間でメッセージを送り合い、会社組織のように協力して開発を行うデモシステム。実装はすべてBashスクリプトで、ビルド・テストフレームワークは存在しない。

## エージェント構成
- **PRESIDENT** (別セッション): 統括責任者
- **boss1** (multiagent:0.0): チームリーダー
- **worker1,2,3** (multiagent:0.1-3): 実行担当

## あなたの役割
- **PRESIDENT**: @instructions/president.md
- **boss1**: @instructions/boss.md
- **worker1,2,3**: @instructions/worker.md

各エージェントは「あなたは[役割名]です」というメッセージを受け取ったら、対応する指示書に従って行動する。

## メッセージ送信
```bash
./agent-send.sh [相手] "[メッセージ]"

# 例
./agent-send.sh boss1 "あなたはboss1です。〜"
./agent-send.sh --list   # エージェント一覧表示
```

## 基本フロー
PRESIDENT → boss1 → workers → boss1 → PRESIDENT

## 主要コマンド

```bash
./setup.sh           # tmux環境構築（既存セッションを削除して再作成、./tmp/をクリア）
./launch-agents.sh   # 全ペインで claude --dangerously-skip-permissions を一括起動
./agent-send.sh      # エージェント間メッセージ送信
./project-status.sh  # worker完了状況の表示
```

環境リセット: `tmux kill-server && rm -rf ./tmp/* && ./setup.sh`

## アーキテクチャ

### 通信の仕組み
`agent-send.sh` がエージェント名をtmuxターゲットに解決し（`get_agent_target()` 内のcase文）、`tmux send-keys` で相手ペインのClaude Codeプロンプトに直接メッセージを打ち込む。送信履歴は `logs/send_log.txt` に記録される。

| エージェント | tmuxターゲット |
|---|---|
| president | president |
| boss1 | multiagent:0.0 |
| worker1-3 | multiagent:0.1-3 |

### 進捗管理の仕組み
ファイルベースで状態を共有する:
- `./tmp/worker{N}_done.txt` — workerがタスク完了時に `touch` で作成。boss1やスクリプトがこの有無で完了判定する
- `logs/send_log.txt` — 全メッセージの送信ログ
- `./tmp/` と `logs/` はgitignore対象（`*.txt` `*.log` も除外されるので注意）

### 成果物の作業ディレクトリ
エージェントが開発する成果物はこのリポジトリではなく `/workspace/[プロジェクト名]` に置く（boss1が指示時に明示する規約）。

## カスタマイズ手順

新しいworkerを追加する場合、3ファイルの変更が必要:
1. `instructions/worker{N}.md` を作成
2. `setup.sh` にペイン作成処理を追加
3. `agent-send.sh` の `get_agent_target()` にマッピングを追加

## 規約
- ドキュメント・スクリプト内メッセージ・コミットメッセージは日本語
- スクリプトは `set -e` と色付きログ関数（`log_info` / `log_success`）のパターンに従う
- `.claude/settings.local.json` にエージェント運用に必要なBash許可リストが定義済み（バージョン管理対象）
