---
title: "How to OpenClawのcronジョブでSlack配信エラーをデバッグする"
emoji: "🔧"
type: "tech"
topics: ["openclaw", "slack", "debugging", "cron"]
published: true
---

## TL;DR

OpenClawのcronジョブが「Message failed」エラーで報告失敗する問題に遭遇。22個のcronジョブのうち13個が同じエラーで失敗したが、軽量タスクは成功。実行レイヤーと報告レイヤーの分離、Slackメッセージングツールの選択的失敗パターンを発見した。

**学んだこと:**
- 報告失敗 ≠ タスク失敗（実行は成功している可能性大）
- Slack配信の選択的失敗パターンを見極める方法
- OpenClaw message toolのログ確認手順

## 前提条件

- OpenClaw Gateway（Mac Mini等で稼働中）
- 複数のcronジョブを運用している環境
- Slack #metricsチャンネルへの自動報告設定済み

## 症状: 13個のcronジョブが「Message failed」

3月16日、22個のcronジョブのうち13個が同じエラーパターンで失敗：

```
❌ エラー（13件、全て "Message failed"）:
- daily-memory (23:00) — 連続3エラー
- autonomy-check (03:00) — 連続6エラー
- app-metrics-morning (05:05) — 連続6エラー
- larry-post-afternoon-ja (17:00) — 連続6エラー
... etc
```

一方で以下のジョブは成功：

```
✅ 成功（4件）:
- build-in-public (23:10) — X投稿成功
- article-writer (23:30) — 記事生成成功
- larry-post-morning-en (07:30) — TikTok投稿成功
```

**疑問点:**
- なぜ同じ #metricsチャンネル宛てなのに選択的に失敗するのか？
- 連続6エラーのジョブと1エラーのジョブの違いは？

## Step 1: 実行と報告の分離を確認する

最初に確認すべきは「タスク自体は成功しているか」。

```bash
# Larry TikTok投稿が実際に実行されたか確認
ls -la ~/.openclaw/workspace/hooks/tiktok-slots/2026-03-16/

# app-metricsのデータが生成されたか確認
ls -la ~/.openclaw/workspace/app-metrics/2026-03-16/
```

**結果判定:**
- ファイルが存在 → タスク実行成功、報告レイヤーだけ失敗
- ファイルなし → タスク自体の失敗

この例では、Larry投稿ファイルとapp-metricsデータは存在していた。**つまり実行成功、報告だけ失敗していた。**

## Step 2: 選択的失敗パターンを分析する

成功と失敗のcronを比較：

| cron | 結果 | 特徴 |
|------|------|------|
| build-in-public | ✅ | 軽量（X投稿1件、短文） |
| article-writer | ✅ | 軽量（記事生成、Slack報告は簡潔） |
| app-metrics | ❌ | 重い（RevenueCat + ASC API呼び出し、長文レポート） |
| larry-post-* | ❌ | 重い（画像生成 + Postiz API + 詳細ログ） |
| autonomy-check | ❌ | 重い（複数APIチェック + 長文レポート） |

**パターン発見:**
- 軽量タスク（短文報告、外部API呼び出し少）→ 成功
- 重いタスク（長文報告、複数API呼び出し）→ 失敗

仮説： **Slackメッセージング層がタイムアウトまたはペイロードサイズ制限に到達している。**

## Step 3: OpenClaw message toolのログを確認

```bash
# OpenClaw Gatewayのログを確認（Mac Miniの場合）
tail -100 ~/Library/Logs/openclaw/gateway.log | grep "message.*failed"

# 特定cronの実行ログを確認
tail -50 ~/.openclaw/workspace/logs/app-metrics-morning-2026-03-16.log
```

**期待される情報:**
- タイムアウトエラー（`ETIMEDOUT`）
- Slack API エラーコード（`429 Too Many Requests`, `413 Payload Too Large`）
- OpenClaw message tool の内部エラー

## Step 4: 一時的な対処法

根本原因がSlack配信層にある場合の暫定対処：

### 対処法A: 報告を簡潔にする

```javascript
// ❌ 長文報告（失敗しやすい）
await message.send({
  channel: 'slack',
  target: 'C091G3PKHL2',
  message: `📊 app-metrics 実行完了\n\n詳細レポート:\n${longReport}\n\nMRR: $${mrr}\nTrials: ${trials}\n...（1000行以上）`
});

// ✅ 簡潔報告（成功しやすい）
await message.send({
  channel: 'slack',
  target: 'C091G3PKHL2',
  message: `📊 app-metrics 完了 | MRR: $${mrr} | Trials: ${trials}\n詳細: workspace/app-metrics/${today}/`
});
```

### 対処法B: 報告レイヤーをスキップする設定

OpenClaw cron設定で `delivery.mode: "none"` を使う：

```javascript
{
  "schedule": { "kind": "cron", "expr": "5 5 * * *" },
  "payload": { "kind": "agentTurn", "message": "Execute app-metrics skill" },
  "delivery": { "mode": "none" },  // Slack報告をスキップ
  "sessionTarget": "isolated"
}
```

ただしこの場合は別途監視システムで成功/失敗を追跡する必要がある。

## Step 5: 根本原因の調査（長期対策）

この問題を完全に解決するには：

1. **OpenClaw message toolのソースコード確認**
   - `~/.openclaw/node_modules/openclaw/src/tools/message.js` 等を読む
   - Slack API呼び出しのタイムアウト設定を確認

2. **Slack API制限を確認**
   - [Slack API rate limits](https://api.slack.com/docs/rate-limits) を読む
   - 特に `chat.postMessage` の制限（1 message/sec）

3. **OpenClaw Issueを確認**
   - GitHub: [github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
   - 同様の報告がないか検索： `"Message failed" delivery slack`

## まとめ

| 教訓 | 詳細 |
|------|------|
| 報告失敗 ≠ タスク失敗 | まず workspace/ のファイルを確認してタスク実行を検証 |
| 選択的失敗は手がかり | 成功/失敗パターンから共通要因を見つける（サイズ/API数/タイムアウト） |
| 報告を簡潔にする | 長文レポートは別ファイル保存 + Slackは要約のみ |
| 根本原因の追跡 | Gatewayログ → OpenClaw message tool → Slack API の順で調査 |

**次のステップ:**
- OpenClaw message toolのログフォーマットを確認する
- 失敗したcronを手動実行して再現性を確認する
- Slack API制限に到達していないか `openclaw gateway` のメトリクスを確認する

この問題に遭遇した場合は、まず「実行は成功しているか」を確認することから始めましょう。報告レイヤーの失敗は、タスク自体の失敗よりも修正が容易です。
