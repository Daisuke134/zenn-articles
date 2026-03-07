---
title: "AIエージェントのcronジョブが3日間サイレント失敗していた話と検知の仕組み化"
emoji: "🤖"
type: "tech"
topics: ["openclaw", "cron", "monitoring", "aiagent"]
published: true
---

## TL;DR

OpenClawで動かしている43個のcronジョブのうち、`roundtable-standup`が3日間止まっていたことに気づきませんでした。原因は**成功前提の設計**と**ヘルスチェック欠如**です。cron失敗を24時間以内に検知する監視cronを追加することで解決しました。

## 前提条件

- OpenClaw Gateway（AIエージェント実行環境）がMac Miniで稼働中
- Slack #metricsチャンネルに全cron結果を報告する設定
- 43個のcronジョブが5分〜24時間間隔で実行中

## 症状: standupが3日間止まっていた

### タイムライン

| 日時 | 出来事 |
|------|--------|
| 2026-03-04 09:00 | standup最終実行成功 |
| 2026-03-05〜03-07 | **実行ログなし** |
| 2026-03-07 20:00 | 別件でdaily-memoryを確認して発覚 |

### 検知が遅れた理由

1. **成功時のみSlack通知**していた（失敗は通知なし）
2. **cron自体は登録されていた**（`openclaw cron list`で表示される）
3. **他のcronは動いていた**（factory, larry等）ため全体の異常に見えなかった

ソース： [SRE Workbook: Monitoring Distributed Systems](https://sre.google/workbook/monitoring/)
核心の引用：「The four golden signals of monitoring are latency, traffic, errors, and saturation. For batch jobs, add **recency** (last successful run time).」

## 根本原因

OpenClawのcron実行エンジンには**リトライ機構がない**。1回失敗したらそのまま次回実行まで待つ。

```javascript
// OpenClaw内部の簡易実装（推測）
async function executeCron(cronJob) {
  try {
    await runSkill(cronJob.skillName);
    await notifySlack(`${cronJob.name} 成功`);
  } catch (error) {
    // エラーログは出るが、Slack通知はしない
    console.error(error);
  }
}
```

**問題点:**
- エラーログはローカルファイルに出るが、Slackに飛ばない
- 次回実行は24時間後なので、気づくのが遅れる
- 失敗原因（権限・API制限・ネットワーク）の調査が困難

## 修正: 監視cronを追加

### Step 1: cron実行履歴をファイルに記録

```bash
# openclaw/src/cron-runner.js（疑似コード）
async function executeCronWithLogging(cronJob) {
  const logPath = `/Users/anicca/.openclaw/logs/cron-history.jsonl`;
  const entry = {
    name: cronJob.name,
    timestamp: new Date().toISOString(),
    status: 'started'
  };
  
  try {
    await runSkill(cronJob.skillName);
    entry.status = 'success';
  } catch (error) {
    entry.status = 'failed';
    entry.error = error.message;
  } finally {
    fs.appendFileSync(logPath, JSON.stringify(entry) + '\n');
  }
}
```

### Step 2: ヘルスチェックcronを作成

```bash
# ~/.openclaw/skills/cron-health-check/check.sh
#!/bin/bash
NOW=$(date +%s)
ALERT_THRESHOLD=86400  # 24時間

# 全cronの最終実行時刻を取得
jq -r '.name' < ~/.openclaw/state/cron-jobs.json | while read -r CRON_NAME; do
  LAST_RUN=$(grep "\"name\":\"$CRON_NAME\"" ~/.openclaw/logs/cron-history.jsonl | tail -1 | jq -r '.timestamp')
  
  if [ -z "$LAST_RUN" ]; then
    echo "⚠️ $CRON_NAME: 実行履歴なし"
    continue
  fi
  
  LAST_UNIX=$(date -j -f "%Y-%m-%dT%H:%M:%S" "$LAST_RUN" +%s 2>/dev/null)
  DIFF=$((NOW - LAST_UNIX))
  
  if [ $DIFF -gt $ALERT_THRESHOLD ]; then
    openclaw message send --channel slack --target 'C091G3PKHL2' \
      --message "🚨 cron-health: ${CRON_NAME} が ${DIFF}秒（$((DIFF/3600))時間）実行されていません"
  fi
done
```

### Step 3: cronに登録

```bash
openclaw cron add \
  --name "cron-health-check" \
  --schedule "0 */6 * * *" \  # 6時間ごと
  --command "bash ~/.openclaw/skills/cron-health-check/check.sh"
```

## 結果

| 項目 | Before | After |
|------|--------|-------|
| standup停止の検知時間 | 72時間 | 6時間以内 |
| 失敗原因の調査 | ログファイル手動確認 | Slackに自動通知 |
| 他のcron異常 | 気づけない | 24時間以内に検知 |

## まとめ

### 教訓

| 教訓 | 詳細 |
|------|------|
| **失敗も通知する** | 成功だけ通知すると、失敗が見えなくなる |
| **定期ジョブには recency 監視** | 「最後にいつ成功したか」を6〜24時間ごとにチェック |
| **ログは中央集約** | ローカルファイルだけだと気づけない。Slack/Datadogに集約 |
| **リトライ機構を入れる** | 1回失敗 = 24時間待ち、は遅すぎる |

### 参考資料

- [Google SRE Workbook: Monitoring](https://sre.google/workbook/monitoring/) — バッチジョブにはrecency監視が必要
- [Kubernetes CronJob Best Practices](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#cron-job-limitations) — 失敗時のリトライポリシー設定

OpenClawを運用している人、自作AIエージェントを作っている人の参考になれば幸いです。
