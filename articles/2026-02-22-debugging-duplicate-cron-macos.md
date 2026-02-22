---
title: "cronジョブが1日3回重複実行される問題の調査と対策"
emoji: "⏰"
type: "tech"
topics: ["cron", "linux", "macos", "devops", "自動化"]
published: true
---

# TL;DR

Mac miniで運用中のcronジョブが1日1回のはずが3回実行される問題が発生。原因はcrontab定義の重複登録とLaunchAgentの併存。検出方法と防止策をまとめた。

## 問題の概要

自宅Mac miniでAIエージェントの自動運用基盤を構築している。daily-memoryというcronジョブが1日1回実行されるべきところ、土曜日に3回実行されていることが発覚した。

## タイムライン

| 時刻 | イベント |
|------|---------|
| 検出日 前日 | daily-memoryが3回/日で実行されていることを確認 |
| 検出日 当日 | 根本原因の調査開始 |
| 現在 | 原因の候補を特定、修正検証中 |

## 考えられる根本原因

### 1. crontab重複登録

`crontab -l` で同一ジョブが複数行登録されているケース。セットアップスクリプトを複数回実行すると、冪等性がない場合に行が増殖する。

```bash
# 確認コマンド
crontab -l | sort | uniq -d
```

### 2. LaunchAgentとcrontabの併存

macOSでは `~/Library/LaunchAgents/` のplistファイルと `crontab` が同時に存在すると、同じジョブが2つのスケジューラから起動される。

```bash
# LaunchAgents確認
ls ~/Library/LaunchAgents/ | grep -i daily
launchctl list | grep daily
```

### 3. タイムゾーンの罠

crontabがUTC基準、LaunchAgentがローカルTZ基準で、意図しない時刻にそれぞれ発火するパターン。

```bash
# crontabのTZ確認
grep TZ /etc/crontab 2>/dev/null
# 環境変数確認
crontab -l | head -5
```

## 防止策

### 冪等なcrontab登録スクリプト

```bash
#!/bin/bash
CRON_JOB="0 1 * * * /path/to/daily-memory.sh"
(crontab -l 2>/dev/null | grep -v "daily-memory.sh"; echo "$CRON_JOB") | crontab -
```

ポイント: 既存のエントリを `grep -v` で削除してから追加する。これにより何回実行しても1行だけになる。

### ロックファイルによる重複実行防止

```bash
#!/bin/bash
LOCKFILE="/tmp/daily-memory.lock"
if [ -f "$LOCKFILE" ]; then
  LOCK_AGE=$(( $(date +%s) - $(stat -f %m "$LOCKFILE") ))
  if [ $LOCK_AGE -lt 3600 ]; then
    echo "Already running (lock age: ${LOCK_AGE}s). Exiting."
    exit 0
  fi
fi
touch "$LOCKFILE"
trap "rm -f $LOCKFILE" EXIT

# 本処理
```

### LaunchAgentの棚卸し

```bash
# 不要なLaunchAgentを無効化
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.example.duplicate.plist
rm ~/Library/LaunchAgents/com.example.duplicate.plist
```

## 教訓

1. **スケジューラは1つに統一する** — macOSではLaunchAgent OR crontab。両方使わない
2. **セットアップスクリプトは冪等に作る** — `grep -v` + 追加パターンでcrontab登録
3. **ロックファイルは保険として必ず入れる** — 根本原因を直しても、防御層として残す
4. **定期的に `crontab -l` を監査する** — 増殖は気づかないうちに起きる

## まとめ

cronの重複実行は「設定の重複」と「スケジューラの併存」が2大原因。冪等な登録スクリプトとロックファイルの2層防御で、根本的に防止できる。macOS環境では特にLaunchAgentとcrontabの併存に注意が必要。
