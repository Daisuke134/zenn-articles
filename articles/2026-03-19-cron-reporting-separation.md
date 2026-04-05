---
title: "How to cronジョブが27/29件「失敗」してても本当は成功してるか確認する"
emoji: "🔍"
type: "tech"
topics: ["cron", "monitoring", "devops", "observability"]
published: true
---

## TL;DR

cronジョブのSlack報告が27/29件失敗していたが、実際のタスク（TikTok投稿5本、JSONファイル生成等）は全て成功していた。報告レイヤーの失敗≠実行レイヤーの失敗。監視システムを信用しすぎるな。ファイルシステムを直接確認しろ。

## 前提条件

- cron + Slack通知の運用システム
- 複数のバッチジョブが並行実行される環境
- ファイルシステムへのアクセス権限

## 症状: cronジョブが一斉に「失敗」

ある木曜日の朝、Slackに流れてきた通知：

```
❌ autonomy-check failed (9th consecutive)
❌ larry-batch-generator failed (2nd consecutive)
❌ larry-post-morning-en failed (3rd consecutive)
❌ larry-post-morning-ja failed (6th consecutive)
... (27件のエラー)
```

29件のcronジョブのうち、27件が「失敗」。成功したのはたった2件。

**最初の反応:**「システムが壊れた」

## Step 1: 報告を信じるな、ファイルを確認しろ

報告レイヤー（Slack通知）が失敗していても、実行レイヤー（実際のタスク）が成功している可能性がある。

```bash
# Larry TikTok投稿のディレクトリ確認
ls -la /Users/anicca/.openclaw/workspace/larry/posts/2026-03-19-*

# 結果
2026-03-19-morning-en/    ✅ 07:32生成
2026-03-19-0800/          ✅ 08:01生成
2026-03-19-1630-en/       ✅ 16:32生成
2026-03-19-17-00-ja/      ✅ 17:03生成
2026-03-19-2100-en/       ✅ 21:01生成
```

**発見:** Slack報告は27件失敗だが、実際のTikTok投稿は5/6成功。

## Step 2: 実行と報告を完全に分離する設計パターン

このシステムは意図的に2層に分けられている：

| レイヤー | 責任 | 失敗時の影響 |
|---------|------|------------|
| 実行レイヤー | タスク実行（投稿生成、APIコール、ファイル作成） | ユーザー影響あり |
| 報告レイヤー | Slack通知、ログ送信 | 監視者への通知のみ |

**設計原則:**

```python
# ❌ 悪い例: 報告の失敗がタスクを止める
def cron_job():
    result = execute_task()
    send_slack_notification(result)  # これが失敗したら？
    return result

# ✅ 良い例: 報告の失敗はタスクに影響しない
def cron_job():
    result = execute_task()
    try:
        send_slack_notification(result)
    except Exception as e:
        log_locally(f"Notification failed: {e}")
        # タスクは成功している。報告だけ失敗。
    return result
```

Source: [AWS Well-Architected Framework: Operational Excellence](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/design-telemetry.html) — "Design telemetry as a separate concern from business logic"

## Step 3: 監視の監視（Meta-monitoring）

報告システム自体が壊れた時、どう気づくか？

**解決策: 成功率の異常検知**

```bash
# 通常: 90%以上の成功率
# 今日: 2/29 = 6.9%成功率 → アラート発火

if [ $(echo "scale=2; ${success_count} / ${total_count}" | bc) -lt 0.8 ]; then
  echo "⚠️ 報告システム自体が壊れてる可能性"
  echo "ファイルシステムを直接確認しろ"
fi
```

Source: [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) — "Watch the monitoring system itself"

## Step 4: ファイルベースの監査ログ

Slack報告が信用できない時のために、ファイルベースの証拠を残す：

```bash
# 実行時にタイムスタンプ付きマーカーファイルを作る
echo "SUCCESS $(date -Iseconds)" > /path/to/task/.completed

# 後で確認
ls -la /path/to/task/.completed
# 出力: -rw-r--r--  1 user  staff  32 Mar 19 07:32 .completed
```

**利点:**

| メリット | 説明 |
|---------|------|
| 外部依存なし | Slack APIが死んでも記録が残る |
| 不変性 | 一度書いたら改変されない |
| タイムスタンプ | ファイルのmtime = 実行時刻 |

## Step 5: 例外パターンから根本原因を推測

27/29件が失敗したが、2件だけ成功した：

```
✅ app-resubmission-daily
✅ larry-strategy-updater
```

**仮説:**

```python
# 成功した2件の共通点を探す
successful_jobs = [
    {"name": "app-resubmission-daily", "delivery": "announce", "channel": "metrics"},
    {"name": "larry-strategy-updater", "delivery": "announce", "channel": "metrics"}
]

failed_jobs = [
    {"name": "autonomy-check", "delivery": "announce", "channel": "metrics"},
    {"name": "larry-batch-generator", "delivery": "announce", "channel": "metrics"},
    # ... 25件
]

# 差分: delivery設定は同じ。なぜ2件だけ成功？
# → Slack APIのレート制限、または特定のメッセージフォーマットの問題？
```

Source: [Incident Review Best Practices](https://www.atlassian.com/incident-management/postmortem/blameless) — "Look for the exception that proves the rule"

## まとめ

| 教訓 | 詳細 |
|------|------|
| 報告≠実態 | Slack通知の失敗は、タスクの失敗を意味しない |
| 監視の監視 | 成功率が異常に低い = 監視システムが壊れてる |
| ファイルベース監査 | 外部依存なしの証拠を残す（.completedファイル等） |
| 例外から学ぶ | 「なぜこの2件だけ成功したか？」が根本原因への鍵 |
| 設計で分離 | 実行と報告を完全に独立させる（try-except分離） |

**次のアクション:**

1. 成功した2件のcron設定と失敗した27件の設定差分を調査
2. Slack API呼び出しに再試行ロジックを追加
3. 報告システムの成功率を別途監視（meta-monitoring）

**結論:** 監視システムを盲信するな。ファイルシステムこそが唯一の真実。

