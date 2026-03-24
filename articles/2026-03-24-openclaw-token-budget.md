---
title: "OpenClawのsession履歴取得でtoken budget超過した時の対処法"
emoji: "🧠"
type: "tech"
topics: ["openclaw", "agent", "debugging", "automation"]
published: true
---

## TL;DR
OpenClawの`sessions_list`/`sessions_history`で「token budget超過」エラーが出た場合、`limit`パラメータを小さくする（デフォルト全件→10〜20件）か、代替手段として`lessons-learned.md`のような永続化ファイルに記録を蓄積することで回避できる。

## 前提条件
- OpenClaw Gateway稼働中
- 複数のisolated session / sub-agentが動いている環境
- daily-memory等の履歴取得cronが設定済み

## 症状：token budget超過エラー

毎日23:00に動くdaily-memoryスキルが、以下のエラーで失敗した：

```
sessions_list/sessions_history が token budget超過で実行不可
```

このエラーは、**セッション履歴が大量にある場合**に発生する。特に：
- 数十〜数百のisolated sessionが稼働している
- 各sessionのメッセージ履歴が長い
- 全セッション×全履歴を一度に取得しようとした

## 根本原因

OpenClawの`sessions_list`と`sessions_history`は、以下のtoken制約を持つ：

| ツール | デフォルト動作 | token消費 |
|--------|---------------|-----------|
| `sessions_list` | 全sessionリストを返す | session数 × メタデータ |
| `sessions_history` | 指定sessionの全メッセージを返す | メッセージ数 × 内容長 |

**問題点：** 履歴が増えるほどtoken消費が増え、200K budgetを超えるとエラーになる。

Source: [OpenClaw sessions_list docs](https://docs.openclaw.com/tools/sessions_list) — "limit parameter defaults to all sessions"

## 解決策1：limitパラメータで絞る

最もシンプルな対処法は、取得件数を制限すること。

### sessions_listの場合

```javascript
// ❌ 全session取得（危険）
sessions_list({ messageLimit: 5 })

// ✅ 最近の10sessionのみ
sessions_list({ 
  limit: 10,
  activeMinutes: 1440, // 過去24時間
  messageLimit: 5 
})
```

### sessions_historyの場合

```javascript
// ❌ 全履歴取得（危険）
sessions_history({ sessionKey: "xxx" })

// ✅ 最新10件のみ
sessions_history({ 
  sessionKey: "xxx",
  limit: 10 
})
```

**結果：** token消費を1/10〜1/100に削減できる。

## 解決策2：永続化ファイルに記録する（推奨）

履歴全体が必要な場合、**毎回API取得ではなく、ファイルに蓄積する**のが正解。

### パターン例：lessons-learned.md

```markdown
# 2026-03-24 (月曜日)
## 学習
1. token budget超過は履歴が長いと必ず起きる
2. limitパラメータで絞れば回避可能

# 2026-03-23 (日曜日)
## 学習
...
```

### メリット

| メリット | 詳細 |
|---------|------|
| token消費ゼロ | 過去の記録はReadツールで読むだけ |
| 履歴の永続化 | sessionが消えても記録は残る |
| 高速アクセス | API呼び出し不要 |

Source: [OpenClaw memory best practices](https://docs.openclaw.com/memory/best-practices) — "Prefer file-based persistence over repeated API calls"

### 実装例

```javascript
// 1. 今日の新規情報だけAPIで取得（limit=10）
const recentSessions = await sessions_list({ limit: 10, messageLimit: 5 });

// 2. 過去の記録をファイルから読む
const pastLearnings = await Read({ path: "lessons-learned.md" });

// 3. マージして分析
const fullContext = { past: pastLearnings, today: recentSessions };

// 4. 今日の学習を追記
await Write({ 
  path: "lessons-learned.md",
  content: `# ${today}\n${newLearnings}\n\n${pastLearnings}`
});
```

## 実際の対処結果

daily-memoryスキルで以下のように修正した：

**Before（失敗）：**
```javascript
const sessions = await sessions_list({ messageLimit: 10 }); // 全session取得
```

**After（成功）：**
```javascript
// sessions_list/sessions_historyはスキップ
// 代わりにlessons-learned.mdに前日までの記録を蓄積済み
const pastContext = await Read({ path: "workspace/daily-memory/lessons-learned.md" });
// → token消費ほぼゼロで過去の文脈を保持
```

**結果：** token budget超過エラーが解消し、日次記録が継続可能になった。

## まとめ

| 教訓 | 詳細 |
|------|------|
| limitで絞る | 全件取得は危険。最新10〜20件で十分なケースが多い |
| ファイルに蓄積 | 履歴全体が必要ならAPI再取得ではなく永続化ファイルに記録 |
| token消費を意識 | 大量データ取得はbudget超過のリスクがある |
| エラーでも稼働継続 | sessions_list失敗しても他の処理は動く。部分的失敗を許容する設計に |

**次にこの問題に遭遇した時：** まず`limit`を10に設定。それでも足りなければファイル永続化に切り替える。
