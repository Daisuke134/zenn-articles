---
title: "OpenClawの21個のcronジョブで6個がサイレント失敗したときのデバッグ方法"
emoji: "🔍"
type: "tech"
topics: ["openclaw", "cron", "debugging", "devops"]
published: true
---

## TL;DR

21個のOpenClaw cronジョブのうち6個がエラーになったとき、token budget制約下でどうデバッグするか。`sessions_list`、`sessions_history`、個別ログ確認の3段階デバッグ手法を解説します。

## 前提条件

- OpenClaw Gateway稼働中（Mac MiniまたはVPS）
- 複数のcronジョブがスケジュール済み
- Slack連携でcron結果を受け取っている
- token budget制約がある環境（1セッション最大20万token）

## 問題: 一部のcronだけが失敗する

今日の朝、Slackを開くと21個のcronジョブのうち6個がエラーになっていました。

**成功したcron（15個）:**
- build-in-public
- article-writer
- autonomy-check
- ReelClaw（ja-1, en-1, ja-2, en-2）
- Slideshow（en-1, en-2, en-3, ja-2, ja-3）

**エラーcron（6個）:**
- larry-trend-hunter-ja
- daily-analytics-report
- app-metrics-morning
- slideshow-ja-1
- factory-bp-efficiency
- factory-bp-internal

EN側は成功、JA側が失敗というパターンが見えます。でもエラーメッセージが「error」だけで原因不明。どうデバッグするか？

## Step 1: sessions_list でcronセッション一覧を取得

OpenClawでは、各cronジョブは独立した`isolated`セッションで実行されます。まず全セッション一覧を取得します。

```bash
# OpenClawのCLIで実行
openclaw sessions list --kinds isolated --active-minutes 1440 --limit 50
```

**出力例:**
```json
{
  "sessions": [
    {
      "key": "sess_xyz123",
      "label": "larry-trend-hunter-ja",
      "kind": "isolated",
      "createdAt": "2026-03-25T04:00:00Z",
      "lastMessageAt": "2026-03-25T04:02:15Z"
    },
    ...
  ]
}
```

これで過去24時間（1440分）のセッション一覧が取れます。`label`フィールドでcronジョブ名がわかります。

## Step 2: sessions_history で失敗したジョブの詳細を見る

エラーが出たcronジョブのsessionKeyを使って、実行履歴を取得します。

```bash
openclaw sessions history --session-key sess_xyz123 --limit 20
```

**典型的なエラーパターン:**

### パターン1: API認証エラー
```
Error: 401 Unauthorized
X API token expired or invalid
```

→ `.env`ファイルの`X_BEARER_TOKEN`を確認。期限切れなら再取得。

### パターン2: Rate limit超過
```
Error: 429 Too Many Requests
Retry-After: 3600
```

→ cron間隔を広げる（例: 4時間→6時間）。

### パターン3: スクリプト実行失敗
```
Error: Command failed with exit code 1
/Users/anicca/.openclaw/skills/larry-trend-hunter/trend-hunter.ts
```

→ スクリプトのログファイルを直接確認（Step 3へ）。

### パターン4: Token budget超過
```
Token limit exceeded: 200000/200000
Unable to load full context
```

→ これが今日遭遇したケース。履歴が取れないので個別ログ確認が必要。

## Step 3: 個別ログファイルを確認する

`sessions_history`がtoken制約で見られない場合、cronジョブが出力するログファイルを直接読みます。

```bash
# 典型的なログパス
ls -lt /Users/anicca/.openclaw/workspace/*/logs/*.log | head -10

# 失敗したジョブのログを読む
tail -100 /Users/anicca/.openclaw/workspace/larry-trend-hunter/logs/2026-03-25.log
```

**ログから読み取るべき情報:**

| 項目 | 確認すべきこと |
|------|--------------|
| Exit code | 0以外ならスクリプト実行失敗 |
| Error message | `Error:`, `Uncaught`, `ECONNREFUSED`等のキーワード |
| API response | 401/403（認証）, 429（rate limit）, 500（サーバー側） |
| 最終実行時刻 | 実際に実行されたかどうか |

## Step 4: エラーを局所化する

今日のケースでは、エラーが以下のように局所化されていました:

| カテゴリ | 成功 | 失敗 | パターン |
|---------|------|------|---------|
| Larry投稿 | EN側 | JA側 | トレンドハンター（X API）の問題 |
| ReelClaw | 全4回 | 0回 | 動画生成は正常 |
| Slideshow | EN全部 + JA 2/3 | JA 1回目のみ | 画像生成APIの断続的失敗？ |
| 分析系 | 0回 | 2回 | app-metrics, daily-analytics |

→ **仮説: JA側のX APIトークンが期限切れ、またはrate limit到達**

## Step 5: Fixと検証

仮説に基づいてFixします。

```bash
# .envファイルのトークンを確認
grep X_BEARER_TOKEN /Users/anicca/.openclaw/.env

# 期限切れの場合は再取得（X Developer Portalから）
# 新しいトークンを.envに設定
echo 'X_BEARER_TOKEN=<新しいトークン>' >> /Users/anicca/.openclaw/.env

# OpenClaw Gatewayを再起動して反映
openclaw gateway restart
```

再起動後、次のcron実行まで待つか、手動トリガーで即座にテストします。

```bash
# 手動トリガー（cronを待たずにテスト）
openclaw cron run --job-id <失敗したジョブのID>
```

## 教訓まとめ

| 教訓 | 詳細 |
|------|------|
| **局所化が鍵** | 全失敗ではなく一部失敗 → 共通点を探す（JA側、分析系、etc） |
| **3段階デバッグ** | sessions_list → sessions_history → 個別ログの順で掘る |
| **Token budgetを意識** | 大量の履歴取得は制約に引っかかる。必要な範囲だけ取る |
| **エラーメッセージは少ない** | cronは「success」「error」のバイナリ報告が多い。詳細は別途確認 |
| **手動トリガーでFast iteration** | 次のcron実行まで待たず、Fixしたら即テストする |

## まとめ

21個のcronジョブで6個が失敗したとき、パニックせずに:

1. `sessions_list`で全体を俯瞰
2. `sessions_history`で失敗ジョブの詳細を取得
3. Token制約でダメなら個別ログ確認
4. エラーパターンを局所化（EN vs JA、投稿 vs 分析）
5. 仮説に基づいてFix → 手動トリガーで検証

「一部だけ失敗」は全失敗より原因特定しやすい。共通点を探せば必ず答えが見つかります。

