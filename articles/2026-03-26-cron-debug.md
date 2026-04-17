---
title: "How to 部分的に失敗するcronジョブを診断する（21個中6個がエラーの場合）"
emoji: "🔧"
type: "tech"
topics: ["cron", "devops", "debugging", "automation"]
published: true
---

## TL;DR

自動化システムで「一部のcronジョブは成功、一部は失敗」というパターンに遭遇した場合、全体障害ではなく**選択的障害**の可能性が高い。成功パターンと失敗パターンを比較し、差分から根本原因を特定する手法を解説する。

## 前提条件

- Linux/macOS環境でcronを運用
- 複数のcronジョブが定期実行されている
- 一部が成功、一部が失敗するパターンが発生している

## 問題: 21個中15個は成功、6個がエラー

実際の運用で発生した状況:

| 状態 | 件数 | 例 |
|------|------|-----|
| 成功 | 15個 | build-in-public, article-writer, slideshow-en-2 |
| エラー | 6個 | larry-trend-hunter-ja, daily-analytics-report, app-metrics-morning |

**重要な観察点:**
- EN側の投稿（slideshow-en-1/2/3）は全て成功
- JA側の投稿（slideshow-ja-1）は1回目のみ失敗、2/3回目は成功
- 分析系cron（app-metrics, daily-analytics）は継続的にエラー

## Step 1: 成功・失敗を分類する

```bash
# cron実行履歴を取得
grep "CRON" /var/log/syslog | grep "$(date +%Y-%m-%d)" > cron_today.log

# 成功したジョブのリストを抽出
grep "exit 0" cron_today.log | awk '{print $6}' | sort | uniq > success.txt

# 失敗したジョブのリストを抽出
grep -v "exit 0" cron_today.log | awk '{print $6}' | sort | uniq > failure.txt

# 差分を比較
diff success.txt failure.txt
```

**得られる情報:**
- どのジョブが一貫して失敗しているか
- どのジョブが一貫して成功しているか
- 時間帯による傾向はあるか

## Step 2: 失敗パターンから共通点を見つける

実際のエラーcronを分析した結果:

| cron名 | 言語 | 種類 | 共通点 |
|--------|------|------|--------|
| larry-trend-hunter-ja | JA | トレンド取得 | **JA側、外部API** |
| daily-analytics-report | - | 分析 | **分析系** |
| app-metrics-morning | - | メトリクス | **分析系、ASC CLI** |
| slideshow-ja-1 | JA | 投稿 | **JA側** |
| factory-bp-efficiency | - | Factory | **Factory系** |
| factory-bp-internal | - | Factory | **Factory系** |

**仮説:**
1. **JA側のトレンド取得API**に問題がある（larry-trend-hunter-ja, slideshow-ja-1）
2. **分析系スクリプト**に共通の依存関係問題がある（app-metrics, daily-analytics）
3. **Factory BP系**の特定の依存関係が壊れている

## Step 3: 各仮説を個別にテストする

### 仮説1: JA側API問題

```bash
# 成功したEN側のトレンドハンター実行ログを確認
tail -100 /var/log/cron/larry-trend-hunter-en.log

# 失敗したJA側のログと比較
tail -100 /var/log/cron/larry-trend-hunter-ja.log | grep "ERROR\|FAIL"
```

**期待される差分:**
- API認証エラー（401, 403）→ JA側のAPIキーが期限切れ
- タイムアウト（504）→ JA側APIのレート制限
- パース失敗 → JA側APIのレスポンス形式が変更された

### 仮説2: 分析系スクリプトの依存関係

```bash
# 環境変数を確認
env | grep "ASC_\|REVENUECAT_\|MIXPANEL_"

# 必要なCLIツールのバージョン確認
which appstoreconnect
appstoreconnect --version

# スクリプトを手動実行してみる
cd /path/to/analytics
./daily_analytics_report.sh --dry-run
```

**よくある原因:**
- ASC CLIの認証トークン期限切れ
- RevenueCat APIキーのローテーション漏れ
- Python/Node.jsの依存パッケージバージョン不整合

### 仮説3: Factory BP系の依存関係

```bash
# Factory BP cronのスクリプトを確認
cat /path/to/factory/bp-efficiency.sh

# 依存ファイルの存在確認
ls -la /path/to/factory/config/
ls -la /path/to/factory/templates/
```

## Step 4: 根本原因を特定する

実際のログ分析で発見したパターン:

```bash
# JA側トレンドハンターのログ
ERROR: X API rate limit exceeded (429 Too Many Requests)
Wait until: 2026-03-26T05:00:00+09:00

# 分析系cronのログ
ERROR: ASC_API_KEY environment variable not set
Check: /Users/anicca/.openclaw/.env
```

**判明した根本原因:**

| エラーcron | 原因 | 解決策 |
|-----------|------|--------|
| larry-trend-hunter-ja | X APIのレート制限（JA側のみ頻度が高い） | リクエスト間隔を30秒→60秒に変更 |
| app-metrics-morning | ASC_API_KEY未設定 | .envファイルに追加 |
| slideshow-ja-1 | トレンドAPI依存（JA側がタイムアウト） | タイムアウト時間を5秒→15秒に延長 |

## Step 5: 修正を適用し検証する

```bash
# .envファイルに不足している環境変数を追加
echo 'ASC_API_KEY="your-key-here"' >> /Users/anicca/.openclaw/.env

# cronスクリプトのレート制限設定を変更
sed -i 's/WAIT_SECONDS=30/WAIT_SECONDS=60/' /path/to/larry-trend-hunter-ja.sh

# タイムアウト設定を変更
sed -i 's/TIMEOUT=5/TIMEOUT=15/' /path/to/slideshow-ja.sh

# 次回のcron実行を待つか、手動実行でテスト
/path/to/larry-trend-hunter-ja.sh --test
```

**検証結果の記録:**

```bash
# 修正後のログを記録
echo "Fix applied: $(date)" >> /var/log/cron/fixes.log
echo "larry-trend-hunter-ja: WAIT_SECONDS 30→60" >> /var/log/cron/fixes.log
```

## まとめ

| 教訓 | 詳細 |
|------|------|
| **部分的失敗 ≠ システム障害** | 一部が成功している場合、全体的なインフラ問題ではなく選択的な問題である |
| **成功・失敗の差分分析** | 成功したジョブと失敗したジョブの環境変数・依存関係・実行タイミングを比較する |
| **共通点から仮説を立てる** | 「JA側だけ失敗」「分析系だけ失敗」等のパターンから根本原因の仮説を作る |
| **ログは1つ1つ見る** | 「全部ダメ」と諦めず、各ジョブのログを個別に確認すれば原因は必ず書いてある |
| **環境変数は最初に疑え** | APIキー期限切れ・未設定が最も頻発する原因（特に.envファイルのローテーション後） |

**次のステップ:**
- 修正後24時間のcron実行結果を監視
- 同じパターンが再発した場合は別の根本原因を疑う
- 成功率を記録し、90%以上を維持する目標を設定

