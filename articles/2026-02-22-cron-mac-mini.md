---
title: "Mac Miniで発生するcronジョブ重複実行を特定・修正する方法"
emoji: "🔄"
type: "tech"
topics: ["cron", "macos", "devops", "debugging"]
published: true
---

## TL;DR
Mac Mini上で自動化システムを運用していると、cronジョブが予期せず1日3回実行される重複実行問題が発生することがある。本記事では、この問題の特定方法と対処法、そして週末運用でも安定稼働させる設計パターンを解説する。

## 前提条件
- macOS環境でのcron運用経験
- LaunchAgentとcrontabの違いの理解
- システムログの基本的な読み方

## 背景：なぜcron重複実行が起きるのか

自動化エージェントシステム（Anicca）をMac Miniで運用していると、以下の症状が発生：

```
予期される動作: 1日1回（深夜実行）
実際の動作: 1日3回実行（原因不明）
影響: APIクレジット消費3倍、ログ肥大化
```

## Step 1: 実行履歴の可視化

まずは本当に重複実行されているかを確認：

```bash
# 本日の実行履歴をチェック
TODAY=$(date +%Y-%m-%d)
grep "daily-memory" /var/log/system.log | grep "$TODAY"

# または、アプリケーション固有のログがあれば
tail -n 100 ~/.openclaw/logs/gateway.log | grep "daily-memory"
```

## Step 2: crontab vs LaunchAgent の競合チェック

Mac環境では、cronとLaunchAgentが同時に走って重複する可能性：

```bash
# 1. crontabをチェック
crontab -l | grep -v '^#'

# 2. LaunchAgentをチェック
ls ~/Library/LaunchAgents/ | grep -i openclaw
ls /Library/LaunchAgents/ | grep -i openclaw

# 3. システムレベルのLaunchDaemon
sudo ls /Library/LaunchDaemons/ | grep -i openclaw
```

## Step 3: OpenClawプロセスの重複起動チェック

```bash
# OpenClaw Gatewayプロセスを確認
ps aux | grep openclaw | grep -v grep

# ポートバインディングの重複確認
lsof -i :3000  # OpenClawのデフォルトポート
netstat -an | grep LISTEN | grep 3000
```

## Step 4: 実行時刻のパターン分析

ログから実行時刻のパターンを抽出：

```bash
# 過去7日間の実行時刻を抽出
for i in {0..6}; do
  DATE=$(date -v-${i}d +%Y-%m-%d)
  echo "=== $DATE ==="
  grep "daily-memory" /var/log/system.log 2>/dev/null | grep "$DATE" | awk '{print $1,$2,$3}'
done
```

## Step 5: タイムゾーン問題の確認

```bash
# システムのタイムゾーン
date
timedatectl show 2>/dev/null || echo "Not systemd"

# cronのタイムゾーン（通常はシステムと同じ）
env TZ=UTC date
env TZ=America/Los_Angeles date
env TZ=Asia/Tokyo date
```

## Step 6: 根本原因の特定

私のケースでは以下のパターンで分析：

| 時刻 | 実行トリガー | 推定原因 |
|------|------------|---------|
| 01:19 PST | cron or LaunchAgent | 正常（設定された時刻） |
| 09:XX PST | 不明 | LaunchAgentの`StartInterval`？ |
| 17:XX PST | 不明 | 別プロセス？ |

## Step 7: 修正方法

### A) LaunchAgentを使う場合（推奨）

```xml
<!-- ~/Library/LaunchAgents/com.anicca.daily-memory.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.anicca.daily-memory</string>
    <key>Program</key>
    <string>/path/to/your/script</string>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>1</integer>
        <key>Minute</key>
        <integer>19</integer>
    </dict>
    <key>RunAtLoad</key>
    <false/>
</dict>
</plist>
```

```bash
# LaunchAgentを登録
launchctl load ~/Library/LaunchAgents/com.anicca.daily-memory.plist

# 既存のcrontabをクリア（重複防止）
crontab -r
```

### B) crontabのみ使う場合

```bash
# 既存をクリア
crontab -r

# 新規設定（1回のみ）
cat > /tmp/new_crontab << 'EOF'
19 1 * * * /path/to/your/script
EOF

crontab /tmp/new_crontab
```

## Step 8: 修正後の検証

```bash
# 1週間後に検証
for i in {0..6}; do
  DATE=$(date -v-${i}d +%Y-%m-%d)
  COUNT=$(grep "daily-memory" /var/log/system.log 2>/dev/null | grep "$DATE" | wc -l)
  echo "$DATE: $COUNT 回実行"
done
```

## まとめ

| 教訓 | 詳細 |
|------|------|
| **単一スケジューラーの原則** | crontabとLaunchAgentを併用しない。どちらか一方に統一する |
| **プロセス重複の監視** | 自動化システムでは同一プロセスの重複起動を定期チェックする |
| **実行ログの可視化** | 問題発生前に正常パターンを記録しておく |
| **週末運用の考慮** | 日付境界やタイムゾーン変更時も安定動作するよう設計する |
| **段階的修正** | 複数原因の可能性がある場合、1つずつ確認・修正する |

この手順で、Mac Miniベースの自動化インフラでも安定したcron運用が実現できる。重要なのは、「なぜ3回実行されるのか」を推測ではなく、ログとプロセス状況から特定することである。