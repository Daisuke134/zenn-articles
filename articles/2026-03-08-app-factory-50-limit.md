---
title: "AIアプリビルダーが50イテレーションで止まる謎を解明した話"
emoji: "🔍"
type: "tech"
topics: ["ai", "debugging", "claude", "automation"]
published: true
---

## TL;DR

Mobile App Factoryが50イテレーション前後で止まる原因を調査。結論： **制限は存在しない**。Claude Codeが効率的すぎて1イテレーションで複数タスクを完了 → ralph.shが「1イテレーション=1タスク」ルールで自動リセット → `prd.json`のフラグが信頼できなくなる → 追記型の`progress.txt`が真の完了記録。

## 症状: 謎の「50イテレーション制限」

先週から疑問だった： **なぜFactoryは50イテレーション前後で止まるのか？**

Desk Stretch Timerの`prd.json`を見ると、US-008b/c/d/eが全て`passes: false`で、notesに`[reset: 1 US per iteration rule]`と書かれていた。

```json
{
  "id": "US-008e",
  "title": "Preflight + TestFlight",
  "passes": false,
  "notes": "[reset: 1 US per iteration rule]"
}
```

しかし`progress.txt`（追記型ログ）を確認すると、US-008eは2026-03-06に完了済みと記録されている。矛盾。

## 調査: 3つのソースを読む

### 1. progress.txt（追記型ログ）

```
2026-03-06 US-008e GREENLIT (0 blocking)
2026-03-06 US-008e asc validate Errors=0
2026-03-06 US-008e TestFlight beta review WAITING_FOR_REVIEW
2026-03-06 US-008e Public link: https://testflight.apple.com/join/78sH7CE9
```

→ **US-008eは完了している**

### 2. prd.json

```json
{
  "id": "US-008e",
  "passes": false
}
```

→ **完了フラグはfalse（矛盾）**

### 3. ralph.sh（制御スクリプト）

ここで真実が判明：

```bash
if NEW_COUNT > 1:
    echo "🔴 1 US enforcement 違反: $NEW_COUNT US が1イテレーションで完了"
    # 最初に完了した1つだけ残して、残りをリセット
    KEEP_FIRST=$(echo "$NEW_PASSES" | xargs | awk '{print $1}')
    # 残りを passes: false に戻す
```

## 根本原因: 1 iteration = 1 US 強制ルール

Claude Codeが効率的に動きすぎて、**1イテレーションで複数のUser Storyを完了**してしまう。

ralph.shは品質担保のため「1イテレーション=1 US」ルールを強制。2つ目以降のUSを自動で`passes: false`にリセットする。

**結果:**
- `prd.json`: 自動リセット対象 → 信頼性低
- `progress.txt`: 追記型 → **こちらが正本**

## Fix: progress.txt を正本として扱う

```bash
# 進捗確認はこっち
cat progress.txt | grep "US-008e"

# prd.json の passes フラグは参考程度
cat prd.json | jq '.userStories[] | select(.id=="US-008e")'
```

実際のDesk Stretch Timerの状態：
- US-001 〜 US-008e: **完了済み**（progress.txtで確認）
- US-009 (App Privacy + submit): **未着手**

## 教訓

| 教訓 | 詳細 |
|------|------|
| **追記型ログが真実** | 上書き可能なフラグ（`prd.json`）は自動リセット対象。追記型（`progress.txt`）こそが信頼できる完了記録 |
| **効率化の副作用** | Claude Codeの高速完了能力 → 1イテレーション=1 USルールでの自動リセット → 見かけ上の「進まない」状態 |
| **制限は存在しない** | 「50イテレーション制限」は誤解。真の問題は品質担保の設計（1 US enforcement）だった |

## まとめ

「max 50」制限は存在しなかった。真の原因は、Claude Codeの効率性 × 品質担保の1イテレーション=1 USルールによる自動リセット挙動。

**デバッグのポイント:**
1. 複数のソース（prd.json, progress.txt, スクリプト）を読む
2. 上書き可能なフラグより追記型ログを信頼する
3. 「制限」と思われる挙動の背後にある設計意図を確認する

静かな日曜に過去ログを深掘りしたことで、見えなかったシステム動作が明らかになった。
