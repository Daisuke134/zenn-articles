---
title: "How to cronジョブのエラーをデバッグする：配信失敗 vs 実行失敗を見分ける"
emoji: "🔍"
type: "tech"
topics: ["devops", "cron", "monitoring", "debugging"]
published: true
---

## TL;DR
cronジョブで「Message failed」と表示されても、実際のタスク実行は成功している可能性があります。この記事では、報告レイヤー（Slack/メール配信）と実行レイヤー（実際の処理）を分離してデバッグする方法を解説します。日曜日に68%のcronが「Message failed」を出したが、実行ログを確認すると一部は正常動作していたという実例をもとに、効率的な切り分け手法を紹介します。

## 前提条件
- cronジョブが実行結果をSlack/メール等に配信している環境
- ログファイルまたは実行結果ファイルへのアクセス権
- bash/zsh等のシェル実行環境

## 問題：「Message failed」の氾濫

日曜日の朝、監視ダッシュボードを見ると以下の状況でした：

| 時間帯 | 成功 | 失敗 | 成功率 |
|--------|------|------|--------|
| 03:00-15:30 | 0/14 | 14/14 | 0% |
| 18:00-23:30 | 7/7 | 0/7 | 100% |
| **全体** | **7/22** | **15/22** | **30%** |

エラーメッセージはすべて「Message failed」。これだけでは、実際に何が失敗したのか分かりません。

### 2つのレイヤー

cronジョブのエラーは2層で考える必要があります：

```
[実行レイヤー] タスク本体（データ取得、ファイル生成、API呼び出し等）
       ↓
[報告レイヤー] 結果の配信（Slack投稿、メール送信等）
```

「Message failed」は報告レイヤーのエラー。実行レイヤーは成功している可能性があります。

## Step 1: 実行レイヤーの確認

### 1-1. ログファイルを確認

```bash
# cronジョブのログディレクトリを確認
LOG_DIR="/var/log/cron-jobs"  # 環境に応じて変更
TASK_NAME="larry-trend-hunter-ja"
TODAY=$(date +%Y-%m-%d)

# 実行ログの最終行を取得
tail -20 "${LOG_DIR}/${TASK_NAME}/${TODAY}.log"
```

**確認ポイント:**
- タスク開始時刻のログがあるか？
- 処理完了のログがあるか？
- エラースタックトレースがあるか？

### 1-2. 生成ファイルを確認

多くのcronジョブは結果をファイルに保存します：

```bash
# 結果ファイルの存在確認
OUTPUT_DIR="/Users/anicca/.openclaw/workspace/hooks"
ls -lh "${OUTPUT_DIR}/${TODAY}-09-00-slot.json"

# ファイルが存在する場合、内容を確認
cat "${OUTPUT_DIR}/${TODAY}-09-00-slot.json" | jq '.status'
```

**判定基準:**
- ファイルが存在 = 実行レイヤーは動作
- ファイルが空 or 存在しない = 実行レイヤー失敗

### 1-3. 外部APIの実行履歴を確認

TikTok投稿やX投稿など、外部APIを叩くcronの場合：

```bash
# Postiz APIの投稿履歴を確認
curl -H "Authorization: ${POSTIZ_API_KEY}" \
  "https://api.postiz.com/v1/posts?date=${TODAY}" | jq '.[].createdAt'

# TikTok動画ディレクトリを確認
ls -lh /Users/anicca/.openclaw/workspace/reelclaw/output/${TODAY}/
```

**実例（日曜日）:**
```bash
$ ls -lh ~/.openclaw/workspace/larry/slideshow/output/ | grep 2026-03-22
drwxr-xr-x  slideshow-ja-3-2026-03-22-18-00
drwxr-xr-x  slideshow-en-3-2026-03-22-18-30
drwxr-xr-x  reelclaw-ja-2-2026-03-22-21-00
drwxr-xr-x  reelclaw-en-2-2026-03-22-21-30
```

→ 夕方4本は実行成功、朝6本のディレクトリは存在せず = 実行レイヤーも失敗。

## Step 2: 報告レイヤーの確認

### 2-1. Slack配信エラーの原因を特定

```bash
# Slack API tokenの有効性確認
curl -X POST https://slack.com/api/auth.test \
  -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" | jq '.ok'
```

**よくある原因:**

| 症状 | 原因 | 対処 |
|------|------|------|
| `invalid_auth` | トークン期限切れ | Slack Appページで再発行 |
| `channel_not_found` | チャンネルID誤り | 正しいID（例: C091G3PKHL2）を確認 |
| `not_in_channel` | Botが未参加 | `/invite @bot_name` で招待 |
| `rate_limited` | レート制限 | リトライ間隔を広げる（Exponential backoff） |

### 2-2. cron実行時刻とSlack配信の相関を分析

日曜日のパターンから、時間帯依存の問題を発見：

```bash
# 時間帯別の成功率を集計
cat cron-results.log | awk '{
  hour = substr($2, 1, 2)
  if (hour < 18) morning++
  else evening++
  if ($3 == "success") {
    if (hour < 18) morning_ok++
    else evening_ok++
  }
}
END {
  print "Morning (03-17): " morning_ok "/" morning
  print "Evening (18-23): " evening_ok "/" evening
}'
```

**実例結果:**
```
Morning (03-17): 0/14 (0%)
Evening (18-23): 7/7 (100%)
```

→ Slack APIのレート制限が朝に厳しい可能性、または自社インフラの夜間高負荷。

## Step 3: 時間帯最適化でリスクヘッジ

### 3-1. クリティカルなcronを成功率の高い時間帯に移動

```bash
# crontab編集
crontab -e

# Before: 05:00に実行（朝、失敗率高）
0 5 * * * /usr/local/bin/app-metrics-morning

# After: 18:00に移動（夕方、成功率100%）
0 18 * * * /usr/local/bin/app-metrics-evening
```

### 3-2. 配信リトライロジックの実装

```bash
# Slack配信を3回リトライ（Exponential backoff）
post_to_slack() {
  local message="$1"
  local channel="$2"
  local max_retries=3
  local retry=0
  
  while [ $retry -lt $max_retries ]; do
    response=$(curl -s -X POST https://slack.com/api/chat.postMessage \
      -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
      -H "Content-Type: application/json" \
      -d "{\"channel\":\"${channel}\",\"text\":\"${message}\"}")
    
    if echo "$response" | jq -e '.ok == true' > /dev/null; then
      echo "Slack投稿成功"
      return 0
    fi
    
    retry=$((retry + 1))
    sleep $((2 ** retry))  # 2秒, 4秒, 8秒
  done
  
  echo "Slack投稿失敗（3回試行）" >&2
  return 1
}
```

### 3-3. ローカルログを常に残す

Slack配信が失敗しても、ローカルログで実行状況を確認可能にする：

```bash
#!/bin/bash
LOG_FILE="/var/log/cron-jobs/task-$(date +%Y-%m-%d).log"

# タスク実行
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

echo "[$(date)] Task started"
./actual-task.sh
echo "[$(date)] Task finished (exit code: $?)"

# Slack配信（失敗してもログは残る）
post_to_slack "Task completed" "C091G3PKHL2" || echo "[WARN] Slack配信失敗"
```

## Step 4: 監視ダッシュボードの改善

### 4-1. 実行レイヤーと報告レイヤーを分離した表示

```bash
# cron結果を2層で集計するスクリプト
#!/bin/bash
echo "| Job | 実行 | 配信 |"
echo "|-----|------|------|"
for job in larry-trend-hunter app-metrics build-in-public; do
  # 実行レイヤー: 出力ファイルの存在確認
  if [ -f "/path/to/${job}/output.json" ]; then
    exec_status="✅"
  else
    exec_status="❌"
  fi
  
  # 報告レイヤー: Slackログ確認
  if grep -q "Slack投稿成功" "/var/log/${job}.log"; then
    report_status="✅"
  else
    report_status="❌"
  fi
  
  echo "| $job | $exec_status | $report_status |"
done
```

**出力例:**
```
| Job                  | 実行 | 配信 |
|----------------------|------|------|
| larry-trend-hunter   | ✅   | ❌   |
| app-metrics          | ✅   | ❌   |
| build-in-public      | ✅   | ✅   |
```

→ larry-trend-hunterは実行成功・配信失敗 = 実行レイヤー健全、Slack側の調査が必要。

## まとめ

| 教訓 | 詳細 |
|------|------|
| **エラーを2層で診断する** | 「Message failed」はどちらのレイヤーか？実行ログ・出力ファイル・API履歴で実行レイヤーを確認 |
| **時間帯とエラーの相関を取る** | 朝0%・夕方100%のパターンがあればレート制限や負荷が原因。クリティカルなcronを夕方に移動 |
| **配信失敗でもログを残す** | Slack/メール配信が失敗してもローカルログで実行状況を追跡可能にする |
| **リトライロジックを実装** | Exponential backoffで一時的な障害を吸収 |
| **監視ダッシュボードを2層表示** | 実行✅配信❌なら優先度低、実行❌配信❌なら緊急対応 |

Source: [SRE Workbook - Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) — "Alert on symptoms, not causes. Distinguish between availability and delivery."

次回cronジョブで「Message failed」が出たら、まず実行レイヤーのログ・ファイル・API履歴を確認してください。配信失敗だけなら、タスク本体は動いています。
