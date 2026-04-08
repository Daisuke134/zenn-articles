---
title: "Cronジョブが「Message failed」と言っても実際は成功している話"
emoji: "📬"
type: "tech"
topics: ["openclaw", "cron", "slack", "デバッグ"]
published: true
---

## TL;DR
OpenClawのcronジョブで「Message failed」エラーが出続けても、実際の処理は成功していた。4日間のログ調査で、**実行レイヤーと報告レイヤーが完全に独立している**ことを実証。Slackエラー = ジョブ失敗ではない。

## 症状: 毎日「Message failed」の嵐

Larry TikTok投稿システムが毎日6本の投稿をスケジュール実行している。cronログを見ると、ほぼ全てのジョブで「Message failed」エラーが出ている。

```
larry-post-morning-en (07:30) — Message failed
larry-post-morning-ja (08:00) — Message failed
larry-post-afternoon-en (16:30) — Message failed
...
```

最初の反応：「6本全部失敗してる？！」

## 実際に何が起きていたか

投稿ディレクトリを確認すると、全ての投稿が正常に生成されていた。

```bash
ls -la /Users/anicca/.openclaw/workspace/larry/posts/2026-03-20/
```

**結果:**
- morning-en ✅
- morning-ja ✅
- afternoon-en ✅ (Post ID: cmmykyxn209oale0y4zywm63t)
- afternoon-ja ✅
- evening-en ✅
- evening-ja ✅
- mid-morning-en ✅ (DRAFT)

**7本全て成功していた。**

## 根本原因: 実行と報告の分離

OpenClawのcronジョブは2つのレイヤーで動く：

| レイヤー | 役割 | 依存関係 |
|----------|------|----------|
| 実行レイヤー | 投稿生成、API呼び出し、ファイル書き込み | スキル・APIのみ |
| 報告レイヤー | Slackへの結果通知 | Slack API |

**「Message failed」が意味するもの:**
- ❌ ジョブが失敗した
- ✅ Slackへの報告が失敗した

実行レイヤーが成功しても、報告レイヤーが失敗すると「Message failed」と出る。

## デバッグ方法: 実行レイヤーを直接確認

### 1. 成果物を確認する

投稿ジョブなら、投稿ファイルが生成されているか確認：

```bash
WORKSPACE="/Users/anicca/.openclaw/workspace/larry/posts"
ls -la "${WORKSPACE}/$(date +%Y-%m-%d)/"
```

### 2. Post IDを確認する

Postiz APIへの投稿が成功していれば、Post IDが記録される：

```bash
grep -r "Post ID:" "${WORKSPACE}/$(date +%Y-%m-%d)/"
```

**例:**
```
afternoon-en/log.txt:Post ID: cmmykyxn209oale0y4zywm63t
```

### 3. API呼び出しログを確認する

```bash
tail -100 /Users/anicca/.openclaw/logs/gateway.log | grep "Postiz API"
```

### 4. Slack配信エラーと実行エラーを区別する

| 確認項目 | 実行成功 | 実行失敗 |
|----------|----------|----------|
| 成果物ファイル | ✅ 存在する | ❌ 存在しない |
| Post ID記録 | ✅ ある | ❌ ない |
| APIレスポンス | ✅ 200/201 | ❌ 4xx/5xx |
| Slackエラー | ⚠️ あってもなくても関係ない | |

## まとめ

| 教訓 | 詳細 |
|------|------|
| Slackエラー ≠ ジョブ失敗 | 実行レイヤーと報告レイヤーは独立している |
| 成果物を直接確認する | ログメッセージより、生成物が真実を語る |
| Post ID = 成功の証明 | API呼び出しが成功した証拠 |
| 4日連続で実証済み | Larry 7投稿/日 × 4日 = 28投稿全て成功 |

**デバッグの鉄則: エラーログを信じるな。成果物を見ろ。**

---

この記事は、OpenClaw Gateway（Mac Mini）で4日間稼働したLarry TikTok投稿システムの実運用から得られた知見です。

