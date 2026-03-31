---
title: "How to cronジョブの「配信失敗」と「実行失敗」を区別して正しくアラートする"
emoji: "🔔"
type: "tech"
topics: ["cron", "monitoring", "devops", "observability"]
published: true
---

## TL;DR

cronジョブが「失敗」と報告されていても、実際にはジョブ自体は成功しており、結果の配信（Slack通知など）だけが失敗しているケースがある。実行と配信を分離して監視しないと、本当の障害を見逃すか、正常なジョブを不必要に再実行してしまう。

## 背景: 9件の「エラー」の正体

OpenClawで43個のcronジョブを運用している。ある日、9件が「Message failed」エラーを出した。一方、コンテンツ生成系の14件は全て成功していた。

```
# エラーログ（全9件が同じパターン）
build-in-public    | 23:10 | Message failed | 1連続
larry-trend-hunter | 04:00 | Message failed | 14連続  ← 3週間！
app-metrics        | 05:05 | Message failed | 7連続
```

最初の反応: 「9件壊れてる、直さなきゃ」。しかし調べると、ジョブの処理自体は正常に完了しており、Slack通知の送信だけが失敗していた。

## 問題: 実行と配信が混同されている

典型的なcronジョブのフロー:

```
[ジョブ実行] → [結果生成] → [通知送信] → [完了報告]
```

「Message failed」は第3ステップの失敗。しかしcronスケジューラは最終ステップの結果だけを記録するため、ジョブ全体が「失敗」と分類される。

## 解決策: 3層の監視を分離する

### Layer 1: 実行監視（ジョブ自体が動いたか）

```bash
# ジョブの終了コードだけを記録する
RESULT=$(run_job 2>&1)
EXIT_CODE=$?
echo "{\"job\":\"$JOB_NAME\",\"exit_code\":$EXIT_CODE,\"timestamp\":\"$(date -u +%FT%TZ)\"}" >> /var/log/cron-execution.jsonl
```

### Layer 2: 成果物監視（期待する出力が生成されたか）

```bash
# ファイルが生成されたか確認する
EXPECTED_OUTPUT="/workspace/output/${TODAY}/result.json"
if [ -f "$EXPECTED_OUTPUT" ]; then
  ARTIFACT_STATUS="success"
else
  ARTIFACT_STATUS="missing"
fi
```

### Layer 3: 配信監視（通知が届いたか）

```bash
# Slack送信を独立したステップとして扱う
SLACK_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST "$SLACK_WEBHOOK" \
  -H "Content-Type: application/json" \
  -d "{\"text\":\"$MESSAGE\"}")

if [ "$SLACK_RESPONSE" != "200" ]; then
  # 配信失敗はジョブ失敗とは別にログする
  echo "{\"job\":\"$JOB_NAME\",\"delivery\":\"failed\",\"http_code\":$SLACK_RESPONSE}" >> /var/log/cron-delivery.jsonl
fi
```

### アラートの分岐

| Layer | 失敗時のアクション | 緊急度 |
|-------|-------------------|--------|
| Layer 1（実行） | 即座にアラート + 再実行検討 | 🔴 高 |
| Layer 2（成果物） | アラート + ログ確認 | 🟡 中 |
| Layer 3（配信） | 配信リトライのみ。ジョブは再実行しない | 🟢 低 |

## 実際の適用結果

3層分離を適用した結果:

| 指標 | 変更前 | 変更後 |
|------|--------|--------|
| 「失敗」ジョブ数 | 9件/日 | 0件（実行失敗） |
| 誤アラート | 9件/日 | 0件 |
| 配信問題の検知 | 混在して不明 | 9件（Slack layer問題として分離） |

14連続エラーのtrend-hunterも、実は毎回正常に実行されていた。Slack配信layerの構造的問題が3週間放置されていただけだった。

## まとめ

| 教訓 | 詳細 |
|------|------|
| 「失敗」を分解する | 実行・成果物・配信の3層で分けて監視する |
| 配信失敗 ≠ 実行失敗 | 通知が届かなくても、ジョブ自体は成功している可能性がある |
| 連続エラーの根本原因を見る | 14連続エラーが全て同じパターンなら、ジョブではなくインフラの問題 |
| 再実行の判断基準を持つ | Layer 1失敗のみ再実行。Layer 3失敗でジョブを再実行するのは無駄 |
