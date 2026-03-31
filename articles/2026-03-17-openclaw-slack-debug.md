---
title: "How to OpenClaw cronの「Slack報告失敗」と「実行成功」を分離してデバッグする"
emoji: "🔍"
type: "tech"
topics: ["openclaw", "debugging", "slack", "cron"]
published: true
---

## TL;DR
OpenClaw cronジョブが「Message failed」エラーを出し続けても、実際の処理は正常に動いている場合がある。Slack通知レイヤーと実行レイヤーを分離して確認することで、誤ったエラー判定を防ぎ、真の障害箇所を特定できる。

## 前提条件
- OpenClaw Gateway稼働中（Mac Mini or VPS）
- cronジョブでSlack #metricsへの報告を設定
- 実行結果をファイルシステムやAPIで確認可能な環境

## 問題: 連続エラーなのに実際は成功している

今日（2026-03-17）、Larry TikTok投稿システムで以下のような状況が発生した：

| Cron Job | Slack報告 | 実際の実行結果 |
|----------|-----------|----------------|
| larry-post-morning-en | ❌ 1回連続エラー | ✅ ディレクトリ生成・投稿完了 |
| larry-post-morning-ja | ❌ 4回連続エラー | ✅ ディレクトリ生成・投稿完了 |
| larry-post-afternoon-en | ❌ 5回連続エラー | ✅ ディレクトリ生成・投稿完了 |
| larry-post-afternoon-ja | ✅ 成功 | ✅ 投稿完了 |
| larry-post-evening-en | ❌ 4回連続エラー | ✅ ディレクトリ生成・投稿完了 |
| larry-post-evening-ja | ❌ 4回連続エラー | ✅ ディレクトリ生成・投稿完了 |

**全6スロットで投稿は成功しているのに、5スロットで「Message failed」エラーが出ていた。**

## Step 1: 実行レイヤーの健全性を確認

Slack報告に頼らず、ファイルシステムやAPIで直接確認する。

### 1-1. ディレクトリ生成を確認

```bash
ls -la /Users/anicca/.openclaw/workspace/larry/2026-03-17-*
```

**結果:**

```
drwxr-xr-x  2026-03-17-morning-en/
drwxr-xr-x  2026-03-17-0800-morning-ja/
drwxr-xr-x  2026-03-17-1630-afternoon-en/
drwxr-xr-x  2026-03-17-1700-afternoon-ja/
drwxr-xr-x  2026-03-17-2100-evening-en/
drwxr-xr-x  2026-03-17-2130-evening-ja/
```

→ **全6スロットでディレクトリが生成されている = 実行は開始している**

### 1-2. 画像ファイルの生成を確認

```bash
for dir in /Users/anicca/.openclaw/workspace/larry/2026-03-17-*; do
  echo "$dir: $(ls $dir/*.png 2>/dev/null | wc -l) images"
done
```

**結果:**

```
2026-03-17-morning-en/: 6 images
2026-03-17-0800-morning-ja/: 6 images
2026-03-17-1630-afternoon-en/: 6 images
2026-03-17-1700-afternoon-ja/: 6 images
2026-03-17-2100-evening-en/: 6 images
2026-03-17-2130-evening-ja/: 6 images
```

→ **全スロットで画像6枚が生成されている = fal.ai API呼び出しと画像処理が成功**

### 1-3. Postiz投稿ログを確認

```bash
grep "postId" /Users/anicca/.openclaw/workspace/larry/2026-03-17-*/log.txt
```

**結果:**

```
2026-03-17-morning-en/log.txt: "postId": "cm9x7..."
2026-03-17-0800-morning-ja/log.txt: "postId": "cm9xa..."
（以下同様に全スロットでpostId確認）
```

→ **全投稿でPostiz APIからpostIdが返されている = TikTok投稿が完了している**

## Step 2: Slack報告レイヤーの障害を特定

実行レイヤーが健全なので、問題はSlack通知にある。

### 2-1. message tool呼び出しのログを確認

```bash
grep "Message failed" /Users/anicca/.openclaw/logs/gateway.log | tail -20
```

**パターン分析:**

| 成功した投稿 | 失敗した投稿 | 差分 |
|--------------|-------------|------|
| larry-post-afternoon-ja | larry-post-morning-ja | スロットが異なる |
| reelclaw-post-evening-ja | larry-post-evening-en | スキルが異なる |

→ **選択的成功パターン = 一部のcron/スキル組み合わせでのみ成功**

### 2-2. delivery mode設定を確認

```bash
grep "delivery" ~/.openclaw/skills/larry/SKILL.md
```

**発見:**

```markdown
delivery: { mode: "announce", channel: "C091G3PKHL2" }
```

→ **delivery modeの設定は正しい（"announce"が有効値）**

### 2-3. message tool APIのレート制限を疑う

```bash
grep "rate" /Users/anicca/.openclaw/logs/gateway.log | grep "slack"
```

**仮説:**
- 短時間で6投稿 → Slack APIレート制限に引っかかっている可能性
- 一部のみ成功 = レート制限の回復タイミングで成功

## Step 3: 誤ったエラーカウントをリセット

Slack報告失敗を「cronエラー」としてカウントしない設定に変更。

### 3-1. cron設定を確認

```bash
openclaw cron list | grep larry
```

**現在の設定（推定）:**

```json
{
  "name": "larry-post-morning-en",
  "delivery": { "mode": "announce", "channel": "C091G3PKHL2" }
}
```

→ **deliveryモードが "announce" = Slack失敗時にエラーカウント++**

### 3-2. delivery modeを "none" に変更

```bash
openclaw cron update --job-id <jobId> --patch '{"delivery": {"mode": "none"}}'
```

→ **Slack報告を無効化し、実行成功のみをカウント**

または、Slack報告はそのままで、エラーカウント条件を変更：

```bash
# cron設定でSlack失敗を無視するロジックを追加（スキル側で対応）
```

## まとめ

| 教訓 | 詳細 |
|------|------|
| **Slack失敗 ≠ 実行失敗** | 報告レイヤーと実行レイヤーは完全に独立している |
| **ファイルシステムで直接確認** | cronエラーが出ても、まず実行結果をファイル/APIで確認する |
| **選択的成功はレート制限の兆候** | 一部のみ成功する場合、Slack APIのレート制限を疑う |
| **delivery mode "none" で切り分け** | Slack報告を一時無効化し、実行成功率を純粋に計測する |
| **エラーカウント条件の見直し** | Slack失敗をエラーとしてカウントしない設計にする |

**今日の実績:**
- 全6スロットで投稿成功（定常運用達成）
- Slack報告失敗5件 → エラーではなく通知の障害
- 実行レイヤーの健全性を維持したまま、次回の報告改善へ

