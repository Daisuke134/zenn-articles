---
title: "Mobile App Factory が50イテレーションで止まる謎を30分で解明した方法"
emoji: "🏭"
type: "tech"
topics: ["ci", "automation", "debugging", "claude"]
published: true
---

## TL;DR

Mobile App Factory が約50イテレーションで止まる現象を調査し、「max 50 制限」は存在せず、実際は「1イテレーション = 1 User Story」ルールの自動リセット挙動が原因と判明。`prd.json` の `passes` フラグは信頼性が低く、`progress.txt` が真の完了記録（append-mode log）であることを発見した。

## 前提条件

- Mobile App Factory: Claude Code (CC) ベースの iOS アプリ自動生成システム
- `ralph.sh`: Factory の制御スクリプト
- `prd.json`: User Story (US) の完了フラグを管理
- `progress.txt`: append-mode の完了ログ

## 問題: なぜ Factory は50前後で止まるのか？

先週から疑問だった「Desk Stretch Timer の prd.json を見ると US-008b/c/d/e が全て `passes: false` だが、notes には完了記録がある」という矛盾。

**症状:**
```json
{
  "id": "US-008e",
  "passes": false,
  "notes": "[reset: 1 US per iteration rule]"
}
```

## Step 1: progress.txt を確認

```bash
cat ~/anicca-project/mobile-apps/desk-stretch-timer/progress.txt
```

**発見:**
- US-001 から US-008e まで全て完了記録あり
- US-008e（TestFlight 提出）は 2026-03-06 に完了
- TestFlight public link も生成済み

## Step 2: ralph.sh のコードを読む

```bash
grep -A 10 "1 US enforcement" ~/anicca-project/mobile-apps/desk-stretch-timer/ralph.sh
```

**発見: 1イテレーション = 1 US 強制ルール**

```python
if NEW_COUNT > 1:
    echo "🔴 1 US enforcement 違反: $NEW_COUNT US が1イテレーションで完了"
    # 最初に完了した1つだけ残して、残りをリセット
    KEEP_FIRST=$(echo "$NEW_PASSES" | xargs | awk '{print $1}')
    # 残りを passes: false に戻す
```

## Step 3: CC のセッションログを確認

```bash
tail -100 ~/anicca-project/mobile-apps/desk-stretch-timer/logs/ralph.log
```

**発見:**
- US-008e 完了時、CC は Greenlight / `asc validate` / release-review を全てパス
- TestFlight beta review 提出成功 (WAITING_FOR_REVIEW)

## 根本原因

**「max 50 制限」は存在しない。**

CC が効率的すぎて 1イテレーションで複数 US を完了してしまうため、ralph.sh が「1イテレーション = 1 US」ルールを強制し、2つ目以降を自動リセットしていた。

## 教訓

| 教訓 | 詳細 |
|------|------|
| **prd.json は信頼性低** | 自動リセット対象。`passes` フラグは Factory の内部状態管理用 |
| **progress.txt が正本** | append-mode log。削除・上書きされない真の完了記録 |
| **1 US per iteration ルールの意図** | CC の暴走を防ぎ、各 US に十分な検証時間を確保する安全装置 |
| **デバッグは append-mode log から** | 状態管理ファイル（JSON等）より、時系列の完全ログを優先 |

## まとめ

Factory が「50で止まる」と思っていたのは誤解で、実際は CC が高速に動作し、ralph.sh が品質担保のために自動調整していただけだった。

**次回同じ問題に遭遇したら:**
1. append-mode log (`progress.txt`) を最初に確認
2. 状態管理ファイル (`prd.json`) は参考程度に
3. 制御スクリプト (`ralph.sh`) のルールを読む


