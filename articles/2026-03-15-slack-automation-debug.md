---
title: "Slack自動通知が選択的に失敗する問題をデバッグする方法"
emoji: "🔍"
type: "tech"
topics: ["slack", "nodejs", "debugging", "automation"]
published: true
---

## TL;DR
同じチャンネル宛てのSlack通知で選択的エラーに遭遇（一部成功、一部失敗）。原因は非同期処理のタイミング問題とエラーハンドリングの不備。cronジョブの実行順序とメッセージペイロードサイズの影響を受けていた。

## 前提条件
- Node.js v25.6.1
- Slack SDK (`@slack/web-api`) または REST API
- cron による定期実行環境
- OpenClaw Gateway（またはNode.jsベースの自動化フレームワーク）

## 発生した症状

3月15日、22個のcronジョブのうち6件が実行され、以下のパターンが観測された：

| Cron | 実行時刻 | 結果 | 備考 |
|------|---------|------|------|
| factory-bp-revenue | 22:00 | ✅ 成功 | 通知配信成功 |
| larry-post-evening-en | 21:00 | ❌ 失敗 | 投稿完了、通知失敗 |
| larry-post-evening-ja | 21:30 | ❌ 失敗 | 投稿完了、通知失敗 |
| factory-bp-efficiency | 22:20 | ❌ 失敗 | 実行完了、通知失敗 |
| factory-bp-internal | 22:40 | ❌ 失敗 | 実行完了、通知失敗 |

**重要な特徴:**
- 全て同じSlackチャンネル (`#metrics`) 宛て
- タスク本体は全て完了している（ファイル生成・TikTok投稿確認済み）
- **通知配信のみ**が選択的に失敗

## Step 1: エラーログを確認する

```bash
# OpenClaw Gateway のログを確認
tail -100 ~/.openclaw/logs/gateway.log | grep -i "message failed"

# cron実行ログを個別に確認
ls -la ~/.openclaw/workspace/larry/2026-03-15-*
```

**発見:**
- `Message failed` エラーが記録されている
- 実行結果のファイルは全て正常に生成されている
- エラー詳細がログに残っていない（catch節で握りつぶされている可能性）

## Step 2: メッセージペイロードサイズを比較する

```javascript
// 成功したメッセージ (factory-bp-revenue)
const successMessage = `📝 factory-bp-revenue 実行完了
✅ BP検索: 5件
📄 revenue-bp-2026-03-15.md 作成`;

// 失敗したメッセージ (larry-post-evening-en)
const failedMessage = `🎬 Larry投稿完了
🌐 TikTok EN: https://...
📊 Hook: "7 signs you're healing..."
🎨 6枚のスライド生成完了`;

console.log('Success:', successMessage.length); // 72文字
console.log('Failed:', failedMessage.length);   // 120文字以上
```

**仮説1:** 長いメッセージがタイムアウトしている可能性

## Step 3: 実行タイミングを調べる

```bash
# cronジョブの実行間隔を確認
openclaw cron list | grep larry
openclaw cron list | grep factory-bp
```

**発見:**
- larry-post系は30分間隔 (21:00, 21:30)
- factory-bp系は20分間隔 (22:00, 22:20, 22:40)
- 成功したfactory-bp-revenueは**最初に実行されたもの**

**仮説2:** Slack API rate limitに達している可能性

## Step 4: エラーハンドリングを改善する

```javascript
// 修正前（エラー詳細が失われる）
try {
  await sendSlackMessage(channel, message);
} catch (error) {
  console.error('Message failed');
}

// 修正後（エラー内容を記録）
try {
  await sendSlackMessage(channel, message);
} catch (error) {
  console.error('Message failed:', {
    error: error.message,
    code: error.code,
    statusCode: error.statusCode,
    retryable: error.retryable
  });
}
```

## Step 5: リトライロジックを追加する

```javascript
async function sendSlackMessageWithRetry(channel, message, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      await sendSlackMessage(channel, message);
      return;
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      
      // Exponential backoff
      const delay = Math.pow(2, i) * 1000;
      console.log(`Retry ${i + 1}/${maxRetries} after ${delay}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

## Step 6: Rate limitを確認する

Slack APIのrate limitは以下の通り：

| Tier | Limit |
|------|-------|
| 標準 | 1リクエスト/秒 |
| Tier 2 | 20リクエスト/分 |
| Tier 3 | 50リクエスト/分 |

6件のcronジョブが100分間（21:00-22:40）で実行される場合、標準Tierでも十分なはずだが、**同時実行**が発生している可能性がある。

```javascript
// cronジョブ間にdelayを追加
setTimeout(() => sendSlackMessage(...), jobIndex * 5000);
```

## まとめ

| 教訓 | 詳細 |
|------|------|
| エラーログは詳細に | `error.message`だけでなく`error.code`, `error.statusCode`も記録する |
| リトライは必須 | ネットワーク通信は必ず失敗する前提で設計する |
| Rate limitを意識 | 同じチャンネルへの連続送信は間隔を空ける |
| 選択的失敗の原因 | 最初のリクエストは成功、後続がrate limitに引っかかるパターンが多い |
| タイムアウト設定 | デフォルト3秒では不十分な場合がある（10秒以上を推奨） |

**次のアクション:**
1. エラーハンドリングを改善してエラー詳細を記録
2. リトライロジックを全てのメッセージ送信に追加
3. cronジョブ間に5秒のdelayを設定
4. 明日のログで改善を確認する

