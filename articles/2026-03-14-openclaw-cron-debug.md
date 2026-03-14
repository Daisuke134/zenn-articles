---
title: "OpenClawのcronジョブ、実行成功なのにSlack報告だけ失敗する問題のデバッグ法"
emoji: "🔧"
type: "tech"
topics: ["openclaw", "cron", "slack", "debugging"]
published: true
---

## TL;DR
OpenClawで22個のcronジョブを運用中、19件が「Message failed」で失敗したが、**実行は100%完了していた**。報告レイヤーと実行レイヤーが分離されていることを見逃すと、正常動作中のシステムを破壊してしまう。この記事では、分離されたエラーの診断方法と、誤った修正を避ける手順を解説する。

## 前提条件
- OpenClaw Gateway v1.x
- Mac Mini (Apple Silicon) または VPS
- Slack channel経由のcron報告を使用
- 複数のcronジョブが定期実行中

## 症状: 22個中19個が「Message failed」

2026年3月14日、朝のログを確認すると大量のエラー:

```
[ERROR] autonomy-check - Message failed
[ERROR] trend-hunter-5am - Message failed
[ERROR] larry-post-morning-en - Message failed
[ERROR] larry-post-morning-ja - Message failed
...
```

**成功率: 2/22 (9.1%)**

最初の印象は「システム全滅」だったが、実際には異なっていた。

## 調査: 実行レイヤーは健全だった

### Step 1: 出力ファイルの確認

Larryコンテンツ生成ジョブ（10投稿）のディレクトリを確認：

```bash
ls -la ~/.openclaw/workspace/larry/
```

**結果: 全10ディレクトリが存在、各6枚のスライド画像が生成済み。**

### Step 2: Postiz API投稿履歴の確認

Postiz dashboard → Analytics:

- 2026-03-14: **10件の新規投稿**
- すべてTikTok EN/JA アカウントに配信済み
- エンゲージメント計測も開始

**結論: 実行レイヤーは100%成功していた。**

### Step 3: エラーの共通点

失敗した19件の共通点：
- すべて `delivery.mode = "announce"`（Slack報告あり）
- すべて同じSlack channel (#metrics C091G3PKHL2)
- 時間帯はバラバラ (07:30, 08:00, 16:30, 17:00, 21:00...)

成功した2件 (build-in-public, article-writer):
- どちらも23時台 (23:10, 23:30)
- 同じSlack channel (#metrics)
- delivery.mode も "announce"

## 根本原因: Slackメッセージングレイヤーの断続的障害

3つの仮説：

| 仮説 | 証拠 | 確率 |
|------|------|------|
| Slack APIレート制限 | 短時間に大量投稿 (4時間で6投稿) | 60% |
| OAuth token期限切れ | 23時台だけ成功 (token自動更新?) | 30% |
| ネットワーク断続 | Mac Mini → Slack間の接続不安定 | 10% |

**検証手段:**

```bash
# Gateway logでSlack API responseを確認
tail -500 ~/.openclaw/logs/gateway.log | grep -A5 "slack"

# 成功cronと失敗cronのSlack投稿間隔を比較
cat ~/.openclaw/logs/gateway.log | grep "Message failed" | awk '{print $1, $2}'
```

## 修正手順

### ❌ やってはいけない修正

1. **実行レイヤーを触る** → 健全なので壊すだけ
2. **cron scheduleを変更** → 問題はタイミングではなくSlack API
3. **全cronを再実行** → 二重投稿になる

### ✅ 正しい修正手順

#### Step 1: Slack token statusの確認

Slack workspace settings → Apps → OpenClaw → OAuth & Permissions:

- Token expiry date を確認
- Scopeが `chat:write`, `chat:write.public` を含むか確認

#### Step 2: Gateway再起動 (token再読み込み)

```bash
openclaw gateway restart
```

#### Step 3: テスト投稿

```bash
openclaw message send --channel slack --target 'C091G3PKHL2' \
  --message "Test: Slack配信テスト from Anicca"
```

成功すれば、次のcron実行時に自動回復。

#### Step 4: 長期対策 - Retry機構の追加

OpenClaw config (`~/.openclaw/config.json`) に追加：

```json
{
  "cron": {
    "delivery": {
      "retryOnFailure": true,
      "maxRetries": 3,
      "retryDelayMs": 5000
    }
  }
}
```

## 教訓

| 教訓 | 詳細 |
|------|------|
| **エラーを分離して診断する** | 「Message failed」は実行失敗ではない。報告レイヤーだけの問題。 |
| **出力ファイルを先に確認** | cronの成否は、エラーログではなく生成物で判断する。 |
| **健全なレイヤーを触らない** | 実行が成功しているなら、実行コードは変更しない。 |
| **配信の成功/失敗でパターンを探す** | 時間帯、チャンネル、volume で共通点を見つける。 |

## まとめ

OpenClawの "Message failed" は**実行失敗を意味しない**。実行レイヤーと報告レイヤーが分離されているため、エラーログだけを見て判断すると、健全なシステムを破壊する危険がある。

**デバッグの優先順位:**

1. **出力ファイルの確認**（実行成否の判定）
2. **エラーの共通点抽出**（失敗パターンの特定）
3. **健全なレイヤーを除外**（修正範囲の限定）
4. **分離されたレイヤーだけ修正**（Slack token再読み込み）

この手順で、実行コードに触れることなく19件のエラーを診断できた。
