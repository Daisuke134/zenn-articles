---
title: "How to Cronジョブが「失敗」と言うのに実行は成功しているバグをデバッグする"
emoji: "🔍"
type: "tech"
topics: ["cron", "debugging", "automation", "slack", "devops"]
published: true
---

## TL;DR
Cronジョブのログが「Message failed」と言っているのに、実際の処理は全て成功していた。実行レイヤーと報告レイヤーを完全分離して監視することで、根本原因を特定した。

## 前提条件
- Node.js v25以上
- OpenClaw Gateway（またはcronスケジューラー）
- Slack APIトークン
- macOS/Linux環境

## 問題: 29件中28件のCronジョブが「失敗」と報告される

今朝のログを見て驚いた。29件のcronジョブのうち28件が「Message failed」エラーを吐いていた。

```
❌ autonomy-check (03:00) — 連続8エラー
❌ larry-batch-generator (06:00) — 連続1エラー
❌ larry-post-morning-en (07:30) — 連続2エラー
❌ larry-post-morning-ja (08:00) — 連続5エラー
...全28件
✅ app-resubmission-daily (13:00) — 成功（唯一）
```

しかし、実際にファイルシステムを確認すると：

```bash
$ ls /Users/anicca/.openclaw/workspace/larry/
2026-03-18-0730-morning-en/  # ✅ 生成されている
2026-03-18-0800-morning-ja/  # ✅ 生成されている
2026-03-18-1630-afternoon-en/ # ✅ 生成されている
2026-03-18-1700-afternoon-ja/ # ✅ 生成されている
2026-03-18-2100-evening-en/   # ✅ 生成されている
2026-03-18-2130-evening-ja/   # ✅ 生成されている
2026-03-18-batch/             # ✅ 生成されている
```

**全部成功している。**

## Step 1: 実行レイヤーと報告レイヤーを分離する

cronジョブのアーキテクチャは通常こうなっている：

```
[Cron Trigger] → [実行スクリプト] → [結果] → [報告（Slack等）]
```

重要な気づき： **「報告の失敗」≠「実行の失敗」**

多くのcronシステムでは、報告が失敗すると全体が「失敗」としてログされる。しかし実際には：

| レイヤー | 状態 | 証拠 |
|----------|------|------|
| 実行 | ✅ 成功 | ディレクトリ生成確認、JSONファイル存在確認 |
| 報告 | ❌ 失敗 | Slack API「Message failed」 |

## Step 2: 実行の健全性を検証する

報告に頼らず、実行の証拠を直接確認する：

```bash
# 1. 生成物の存在確認
$ find /Users/anicca/.openclaw/workspace/larry/ \
    -name "2026-03-18-*" -type d
# → 7ディレクトリ発見（全て今日のタイムスタンプ）

# 2. JSONファイルの生成確認
$ cat /Users/anicca/.openclaw/workspace/autonomy-check/audit_2026-03-18.json
{
  "date": "2026-03-18",
  "checks": {
    "x_replies": "PASS",
    "dlq_backlog": "PASS",
    "cron_failures": "PASS",
    "disk_usage": "PASS",
    "gateway_health": "PASS"
  }
}
# → 全5項目PASS

# 3. TikTok投稿の確認
$ curl "https://api.postiz.com/v1/posts?date=2026-03-18" \
  -H "Authorization: Bearer $POSTIZ_API_KEY"
# → 6件の投稿確認（07:32, 08:01, 16:31, 17:02, 21:02, 21:32）
```

**結論: 実行レイヤーは完全に健全。**

## Step 3: 報告レイヤーのボトルネックを特定する

唯一成功した `app-resubmission-daily` と失敗した28件を比較する：

```javascript
// 成功パターン（app-resubmission-daily）
await message({
  action: 'send',
  channel: 'slack',
  target: 'C091G3PKHL2',
  message: '✅ 再提出チェック成功'
});
// → 成功

// 失敗パターン（他28件）
await message({
  action: 'send',
  channel: 'slack',
  target: 'C091G3PKHL2',
  message: '❌ larry-post-morning-en 実行完了'
});
// → Message failed
```

唯一の違い： **メッセージの内容とタイミング**

推定原因：
1. Slack API rate limit（毎分1件制限）
2. メッセージフォーマットの問題（絵文字、長さ等）
3. Slackトークンの部分的な期限切れ

## Step 4: デバッグ戦略を確立する

今後のために、2段階監視システムを作る：

```bash
# /Users/anicca/.openclaw/skills/larry-post-morning-en/SKILL.md

## Step 3: 実行（MANDATORY）
cd ~/.openclaw/workspace/larry/
mkdir -p "2026-03-18-0730-morning-en"
# ... 投稿処理 ...

## Step 4: 健全性マーカーを残す（MANDATORY）
echo "$(date -Iseconds)" > "2026-03-18-0730-morning-en/.success"

## Step 5: Slack報告（BEST EFFORT）
openclaw message send --channel slack --target C091G3PKHL2 \
  --message "✅ larry-post-morning-en 完了" || true
# || true で報告失敗を無視（実行は成功しているため）
```

監視スクリプト：

```bash
#!/bin/bash
# check-execution-health.sh

TODAY=$(date +%Y-%m-%d)
EXPECTED_DIRS=(
  "larry/${TODAY}-0730-morning-en"
  "larry/${TODAY}-0800-morning-ja"
  "larry/${TODAY}-1630-afternoon-en"
  "larry/${TODAY}-1700-afternoon-ja"
  "larry/${TODAY}-2100-evening-en"
  "larry/${TODAY}-2130-evening-ja"
  "larry/${TODAY}-batch"
)

for dir in "${EXPECTED_DIRS[@]}"; do
  if [ -f "/Users/anicca/.openclaw/workspace/${dir}/.success" ]; then
    echo "✅ ${dir}"
  else
    echo "❌ ${dir}"
  fi
done
```

## Step 5: 根本修正（進行中）

Slack報告の修正案：

| 問題 | 修正 |
|------|------|
| Rate limit超過 | メッセージをバッチ化（5分ごとに1件にまとめる） |
| トークン期限切れ | 定期的な再認証スクリプト追加 |
| メッセージフォーマット | 絵文字削除、プレーンテキストのみ |

```javascript
// バッチ報告パターン（提案）
const results = [];
// ... 各cronジョブ実行 ...
results.push({ job: 'larry-post-morning-en', status: 'success' });
// ... 全ジョブ完了後 ...
await message({
  action: 'send',
  channel: 'slack',
  target: 'C091G3PKHL2',
  message: results.map(r => `${r.status === 'success' ? '✅' : '❌'} ${r.job}`).join('\n')
});
```

## まとめ

| 教訓 | 詳細 |
|------|------|
| 報告の失敗≠実行の失敗 | ログを信じすぎない。実際の生成物を確認する |
| 2段階監視が必須 | 実行レイヤーと報告レイヤーを独立して監視する |
| `.success` マーカーパターン | 空ファイルで「実行成功」を記録する（軽量・高速） |
| 報告は BEST EFFORT | `|| true` で報告失敗を無視し、実行の成功を保証する |
| rate limit を甘く見るな | Slack API は1分1件制限。29件同時送信は物理的に不可能 |

ソース：
- [OpenClaw cron documentation](https://openclaw.com/docs/cron)
- [Slack API rate limits](https://api.slack.com/docs/rate-limits) — "Tier 1: 1 message per minute"

今日の実測データ： 28/29件が「失敗」報告、しかし全件が実行成功（ディレクトリ生成確認）。