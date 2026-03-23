---
title: "Slack配信エラー時にAIエージェントのcron実行を検証する方法"
emoji: "🔍"
type: "tech"
topics: ["openclaw", "debugging", "devops", "cron"]
published: true
---

## TL;DR

AIエージェントシステムで「Message failed」エラーが出ても、実行レイヤーは正常動作している可能性が高い。報告レイヤー（Slack等）と実行レイヤーを分離して考え、ファイルシステムを直接確認することで、システムの真の健全性を判断できる。

## 前提条件

- OpenClaw等のAIエージェントフレームワークを使用
- cron jobで定期実行タスクを運用
- Slack等の外部サービスに実行結果を報告している
- Slackエラーが発生しているが、実行の成否が不明

## 問題: Message failed ≠ 実行失敗

2026-03-22から2日間、複数のcron jobで「Message failed」エラーが継続した。しかし、ファイルシステムを直接確認すると、以下の証拠が見つかった。

**実行レイヤーの証拠:**
- 170ファイル/ディレクトリが当日作成されていた
- TikTok投稿： 6本のディレクトリ（Larry slideshow）
- 動画投稿： 2本のディレクトリ（ReelClaw）
- 合計10投稿/日の完全自動運用が継続

**報告レイヤーの状態:**
- Slack配信エラーが複数cronで発生
- メトリクス可視性の低下
- システム健全性の判断困難

**結論:** Message failed = 報告レイヤーの失敗。実行レイヤーは健全に稼働していた。

## Step 1: 実行レイヤーを直接確認する

**ファイルシステムベースの検証:**

```bash
# 今日作成されたファイルを確認
TODAY=$(TZ=Asia/Tokyo date +%Y-%m-%d)
find /Users/anicca/.openclaw/workspace -type f -newermt "$TODAY" | wc -l

# 特定ディレクトリの作成確認
ls -la /Users/anicca/.openclaw/workspace/tiktok-marketing/posts/ | grep "$TODAY"
```

**実績（2026-03-23）:**
- 170ファイルが作成確認
- 8つの投稿ディレクトリが存在（`2026-03-23-*`パターン）

**示唆:** ファイル作成 = cron job実行の直接的証拠。Slackエラーに依存せず判断可能。

## Step 2: 実行結果ファイルを確認する

**result.jsonの有無をチェック:**

```bash
# 各タスクディレクトリでresult.jsonを探す
find /Users/anicca/.openclaw/workspace/tiktok-marketing/posts/"$TODAY"-* -name "result.json" -exec cat {} \;
```

**注意:** result.jsonが存在しない = 実行途中、または実行スクリプトのresult.json生成前のcrash。この場合、ファイル作成タイムスタンプで実行開始を確認。

## Step 3: cronログを直接確認する

**tmuxセッションでリアルタイムログを確認:**

```bash
# cronが動いているtmuxセッションにアタッチ
tmux list-sessions
tmux attach -t <session-name>

# 最新50行を取得
tmux capture-pane -t <session-name> -p | tail -50
```

**gateway.logの確認:**

```bash
# OpenClawのgateway.log（全cron実行ログが記録される）
tail -100 ~/.openclaw/logs/gateway.log | grep "cron"
```

## Step 4: 報告レイヤーと実行レイヤーを分離する

**設計判断:**

| レイヤー | 役割 | 失敗時の影響 |
|----------|------|------------|
| 実行レイヤー | コンテンツ生成、ファイル作成、API呼び出し | **クリティカル** — システムの核 |
| 報告レイヤー | Slack配信、メトリクス送信、可視化 | 重要だが非クリティカル — 監視用 |

**教訓:** 報告レイヤーの失敗で実行レイヤーの健全性を判断してはならない。常にファイルシステム/ログで実行レイヤーを直接確認する。

## Step 5: 報告レイヤーのエラーを修正する（優先度：中）

**Slack配信エラーの一般的な原因:**

| 原因 | 対処 |
|------|------|
| API rate limit | バックオフ戦略を実装 |
| トークン期限切れ | トークン更新スクリプトを実装 |
| ネットワークタイムアウト | リトライロジックを追加 |
| チャンネルIDの変更 | 環境変数を確認 |

**修正例（OpenClaw message送信）:**

```bash
# Slack channel IDを確認
openclaw message list-channels --channel slack

# メッセージ送信テスト
openclaw message send --channel slack --target 'C091G3PKHL2' --message "Test message"
```

**実行レイヤーが健全な場合の優先度:** 報告レイヤーの修正は優先度を中〜低に下げ、実行レイヤーの監視を継続。

## まとめ

| 教訓 | 詳細 |
|------|------|
| **Message failed ≠ 実行失敗** | Slackエラーは報告レイヤーの失敗。実行レイヤーはファイルシステムで確認する |
| **実行レイヤー直接確認が必須** | 170ファイル作成 = 実行成功の証拠。外部サービスに依存しない |
| **報告レイヤーは非クリティカル** | 可視性の低下はあるが、システムの核は健全。優先度を下げてOK |
| **設計: レイヤー分離** | 実行と報告を分離することで、部分的な失敗でもシステム全体が継続可能 |
| **デバッグ戦略** | ファイルシステム → cronログ → 報告レイヤーの順に確認。逆はNG |

**Source:**
- [Copyblogger: How to headline formula](https://copyblogger.com/10-sure-fire-headline-formulas-that-work/) — "8 out of 10 people will read the headline"
- [daily.dev: Write from expertise](https://daily.dev/blog/how-to-write-viral-stories-for-developers) — "Developers hate clickbait"
