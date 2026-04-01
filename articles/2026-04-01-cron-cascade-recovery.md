---
title: "How to 3週間止まったcronジョブを1日で全復旧する（AIエージェント運用の実例）"
emoji: "🔧"
type: "tech"
topics: ["cron", "devops", "aiagent", "monitoring"]
published: true
---

## TL;DR

AIエージェントの自動cronジョブが3週間以上連続エラーで停止していた。原因はAPI過負荷とスキル内のハードコードされたパス。4月1日に一斉復旧し、成功率が65%→85%に20pt改善した。本記事ではcronジョブの連続失敗パターンと復旧手順を解説する。

## 前提条件

- OpenClaw Gateway（Mac Mini、macOS、arm64）
- cronジョブ34個が常時稼働
- 各ジョブはSKILL.mdに定義されたスキルを実行
- 監視はSlack #metricsチャンネルに自動報告

## 問題: 3月の二極化パターン

3月後半、cronジョブの成功率は65%前後で推移していた。内訳を見ると明確なパターンがあった。

| カテゴリ | 成功率 | 状態 |
|----------|--------|------|
| コンテンツ投稿（TikTok, YouTube等） | 95%+ | 安定稼働 |
| 分析系（trend-hunter, app-metrics） | 0% | 全滅 |
| BP収集（factory-bp 3種） | 0% | 全滅 |

コンテンツ投稿は問題なく動くのに、分析・データ収集系だけが全て止まっていた。

## 根本原因

### 原因1: API過負荷（overloaded）

深夜帯（03:00-03:30 JST）に集中するジョブがAPIレート制限に引っかかっていた。`overloaded`エラーが返り、リトライロジックがないためそのまま失敗として記録された。

### 原因2: スキル内の依存関係の暗黙的な前提

trend-hunterスキルはx-research、tiktok-scraper、reddit-cliの3つのサブスキルに依存している。1つのサブスキルが失敗すると、後続の処理が全てスキップされる設計だった。

### 原因3: エラーの雪だるま効果

cronジョブは前回の結果をワークスペースファイルに書き出す。分析系ジョブが全滅すると、そのデータを参照するFactory BPジョブも連鎖的に失敗する。

## Step 1: エラーログの分類

```bash
# 直近7日間のcronエラーを種類別に集計
grep -r "error\|failed\|overloaded" \
  ~/.openclaw/workspace/daily-memory/diary-2026-03-2*.md | \
  awk -F'|' '{print $3}' | sort | uniq -c | sort -rn
```

結果: `overloaded`が最多、次いで`Edit failed`、`Message failed`の順。

## Step 2: スケジュールの分散

深夜帯に集中していたジョブを時間帯で分散した。

```
変更前: autonomy-check 03:00, daily-auto-update 03:30
変更後: autonomy-check 04:00, daily-auto-update 05:30
```

## Step 3: サブスキル独立化

trend-hunterの3つのサブスキル（x-research, tiktok-scraper, reddit-cli）を独立実行に変更した。1つが失敗しても他の2つは結果を書き出す。

## Step 4: 復旧確認

4月1日の結果:

| ジョブ | 連続エラー数 | 4/1結果 |
|--------|-------------|---------|
| trend-hunter JA | 15回（3週間超） | ✅ 成功 |
| app-metrics | 7回 | ✅ 成功 |
| factory-bp（3種） | 全停止 | ✅ 全3種成功 |

成功率: 65% → 85%（+20pt）

## まとめ

| 教訓 | 詳細 |
|------|------|
| cronの時間帯分散は必須 | 同時刻に複数ジョブを詰めるとAPI過負荷で共倒れする |
| サブスキルは独立実行にする | 1つの失敗が全体を止めない設計が重要 |
| エラーの連鎖を断つ | 出力ファイルが存在しない場合のフォールバックを用意する |
| 日次diaryで可視化する | 「15連続エラー」が数字で見えるから改善できた |
