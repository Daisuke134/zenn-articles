---
title: "OpenClawのcronジョブが週末に沈黙する原因と対処法"
emoji: "🔇"
type: "tech"
topics: ["openclaw", "cron", "devops", "debugging"]
published: true
---

## TL;DR

OpenClawエージェントのcronが週末に突然実行されなくなる問題を調査し、「週末用cronスケジュール設定の欠落」が原因と判明。平日5日限定スケジュールを7日化することで解決。cronは「設定通りに動く」ため、意図しない沈黙は設計見直しのサイン。

## 前提条件

- OpenClaw Gateway 実行中（Mac Mini / VPS）
- cron 管理コマンド: `openclaw cron list`
- 調査対象: trend-hunter, Factory BP, app-metrics等の定常cron

## 症状: 週末の沈黙

2026-03-28（土）と03-29（日）に、普段14件/日実行されるcronが突然0件に。

```bash
$ openclaw sessions list --activeMinutes 1440
# → daily-memory のみ（平日は14+件表示される）
```

## Step 1: cronスケジュールを確認する

```bash
openclaw cron list --includeDisabled
```

**出力例:**

```json
{
  "name": "trend-hunter-morning",
  "schedule": {
    "kind": "cron",
    "expr": "0 5 * * 1-5",  // ← Mon-Fri のみ！
    "tz": "Asia/Tokyo"
  },
  "enabled": true
}
```

**根本原因:** `1-5`（月〜金）指定で、土日（6-7）が除外されていた。

## Step 2: 7日化する

```bash
openclaw cron update --jobId <job-id> --patch '{"schedule": {"expr": "0 5 * * *"}}'
```

| Before | After |
|--------|-------|
| `0 5 * * 1-5` | `0 5 * * *` |
| Mon-Fri only | Every day |

## Step 3: 他のcronも確認

```bash
openclaw cron list | grep '"expr"' | grep '1-5'
```

全12件のcronで `1-5` が使われていた → 一括更新。

## Step 4: 動作確認

```bash
# 翌日（月曜 03-30）
openclaw sessions list --activeMinutes 1440
# → 14+件のセッション確認、正常復旧
```

## よくある間違い

| 間違い | 正解 |
|--------|------|
| `0 5 * * 1-5` を「毎日5時」と誤解 | `1-5` = 月〜金のみ |
| 「週末は手動実行すればいい」 | 自律性の喪失、cronの意味がない |
| 「平日だけ動けばOK」 | トレンド収集は週末こそ活発、データ欠落のリスク |

## 長期エラーへの波及

この問題により以下が2週間以上停止していた:

- **trend-hunter**: 12連続エラー → トレンドデータ欠落
- **Factory BP系**: 5連続エラー → ベストプラクティス自動収集停止
- **app-metrics**: 5連続エラー → メトリクス取得失敗

cronが「静かに失敗」していたため、発見が遅れた。

## 教訓

| 教訓 | 詳細 |
|------|------|
| cron式は必ず7日を明示 | `* * *` or `0-6` で全日指定 |
| 週末動作を意図的に止めるなら明記 | コメント or 設計ドキュメントに理由を残す |
| daily-memory は最後の砦 | 静穏期でも唯一動くメタ監視cron、異常検知の要 |
| 「動いてるはず」は危険 | `sessions list` で実行状況を定期確認 |

## まとめ

OpenClawのcronは「設定通り」に動く。週末沈黙の原因は「週末用設定の欠落」。意図しない静穏は設計見直しのサイン。cron式は `* * *` で7日を明示し、異常検知にはメタ監視cron（daily-memory等）を活用すること。


