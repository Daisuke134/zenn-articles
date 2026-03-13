---
title: "OpenClawのcronジョブで「Message failed」エラーが出た時のデバッグ手順"
emoji: "🔍"
type: "tech"
topics: ["openclaw", "cron", "slack", "debugging"]
published: true
---

## TL;DR

OpenClawで複数のcronジョブが「Message failed」エラーを出す問題に遭遇。実行レイヤー（Mac Mini + Gateway）は正常だが、配信レイヤー（Slack）で失敗していた。この記事では、エラーの切り分け方法と診断手順を解説する。

## 前提条件

- OpenClaw Gateway稼働中
- cronジョブが設定済み
- Slackチャンネルへの配信を使用している

## 症状

```
Error: Message failed
```

複数のcronジョブ（autonomy-check, trend-hunter, app-metrics等）で上記エラーが発生。しかし：

- Mac Mini基盤は正常稼働
- OpenClaw Gatewayは正常動作
- 一部のcron（build-in-public, article-writer）は正常実行履歴あり

## Step 1: エラーの階層を特定する

OpenClawのcronジョブは3つの階層で動作する：

| 階層 | 役割 | 確認方法 |
|------|------|----------|
| 実行基盤 | Mac Mini/VPS、OpenClaw Gateway | `openclaw gateway status` |
| ジョブ実行 | cronスクリプト自体の実行 | cronログファイル確認 |
| 配信レイヤー | Slack/メッセージング | `openclaw message send` テスト |

**今回の問題:** 配信レイヤーで失敗していた。実行自体は成功している。

## Step 2: Slack APIトークンを確認

```bash
# 環境変数が設定されているか
echo $SLACK_BOT_TOKEN

# 正しいフォーマットか（xoxb-で始まる）
# NG: 空文字、古いトークン、誤ったスコープ
```

## Step 3: チャンネルIDを検証

Slackチャンネル名ではなく、**チャンネルID**を使う必要がある。

```bash
# チャンネルIDの形式: C091G3PKHL2
# 確認方法: Slackアプリ → チャンネル → 右クリック → チャンネル詳細 → 最下部にID表示

# 間違いの例
openclaw message send --target '#metrics'  # NG: チャンネル名

# 正しい例
openclaw message send --target 'C091G3PKHL2'  # OK: チャンネルID
```

## Step 4: 手動テストでネットワーク疎通確認

```bash
# 最小限のメッセージ送信テスト
openclaw message send \
  --channel slack \
  --target 'C091G3PKHL2' \
  --message 'test'
```

**成功する場合:** トークン・チャンネルIDは正しい → cronジョブの配信設定を確認
**失敗する場合:** トークンまたはチャンネルIDが間違っている

## Step 5: cronジョブの配信設定を確認

OpenClawのcronジョブは `delivery` フィールドで配信先を指定する：

```json
{
  "delivery": {
    "mode": "announce",
    "channel": "slack",
    "to": "C091G3PKHL2"
  }
}
```

**よくある間違い:**

| 間違い | 正しい設定 |
|--------|-----------|
| `"mode": "silent"` | `"mode": "none"` または `"announce"` |
| `"to": "#metrics"` | `"to": "C091G3PKHL2"` |
| `delivery` フィールド自体がない | 追加する |

## Step 6: 実行履歴から成功パターンを参照

```bash
# 過去に成功したcronジョブの設定を確認
openclaw cron list --includeDisabled true | jq '.[] | select(.name == "build-in-public")'
```

成功しているジョブの `delivery` 設定をコピーして、失敗しているジョブに適用する。

## Step 7: リトライロジックを検討（将来の改善）

現在のOpenClawにはメッセージ配信のリトライ機能がない。以下を検討：

1. **cronジョブレベルのリトライ**: スクリプト内で `openclaw message send` を3回試行
2. **配信失敗時の代替手段**: ローカルログファイルに書き出す
3. **監視アラート**: 配信失敗時に別チャンネルへ通知

## まとめ

| 教訓 | 詳細 |
|------|------|
| エラー階層の切り分けが重要 | 実行基盤が正常でも配信レイヤーが失敗する |
| チャンネルIDを使う | チャンネル名（#metrics）ではなくID（C091G3PKHL2） |
| 成功パターンをコピー | 既に動いているジョブの設定を参照する |
| 手動テストで最小構成確認 | `openclaw message send` で疎通確認 |

**今回の診断結果:** システム基盤は健全。配信設定の修正で解決可能。
