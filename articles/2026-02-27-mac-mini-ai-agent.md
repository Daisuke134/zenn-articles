---
title: "How to Mac MiniでAIエージェントを安定運用する（4日間70%成功率を達成）"
emoji: "🤖"
type: "tech"
topics: ["mac-mini", "automation", "ai-agent", "openclaw"]
published: true
---

## TL;DR
Mac MiniでOpenClawベースのAIエージェントを運用し、4日間で約70%のcron成功率を達成。VPS→Mac Mini移行後の自律運用が定着し、複数スキルの並列実行、Blotato API経由のX投稿、Buddhist principlesマーケティングまで自動化に成功。

## 前提条件
- Mac Mini（M1/M2以降推奨）
- OpenClaw Gateway
- 複数の自動実行スキル（x-poster、roundtable-standup、moltbook-interact等）
- Blotato APIアクセス

## Step 1: Mac Mini環境セットアップ

### LaunchAgentによる自動起動設定
```bash
# OpenClaw Gatewayの自動起動設定
mkdir -p ~/Library/LaunchAgents
cat > ~/Library/LaunchAgents/com.openclaw.gateway.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openclaw.gateway</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/openclaw</string>
        <string>gateway</string>
        <string>start</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
EOF

# LaunchAgent読み込み
launchctl load ~/Library/LaunchAgents/com.openclaw.gateway.plist
```

### システム設定
```bash
# 自動ログイン有効化（停電時の自動復旧用）
sudo defaults write /Library/Preferences/com.apple.loginwindow autoLoginUser -string "$(whoami)"

# スリープ無効化
sudo pmset -a sleep 0
sudo pmset -a hibernatemode 0
```

## Step 2: 複数スキルの並列cron設定

### cronジョブ設定例
```bash
# crontab設定
crontab -e

# 投稿系（朝・夜の2回）
0 9 * * * /usr/local/bin/openclaw skills run x-poster-morning
0 21 * * * /usr/local/bin/openclaw skills run x-poster-evening

# 分析系（朝のスタンドアップ）
30 8 * * * /usr/local/bin/openclaw skills run roundtable-standup

# コミュニティ活動（4時間間隔）
0 */4 * * * /usr/local/bin/openclaw skills run moltbook-interact

# デッドライン監視（日次）
0 10 * * * /usr/local/bin/openclaw skills run naist-deadline-scanner
```

## Step 3: API依存性の安定化

### Blotato API設定
```bash
# 環境変数設定
export BLOTATO_API_KEY="your_api_key"
export BLOTATO_ACCOUNT_ID_EN="your_account_id"

# APIエンドポイント（重要: api.blotato.com は廃止済み）
BLOTATO_BASE_URL="https://backend.blotato.com"
```

### エラーハンドリング
```bash
# スキル実行時の成功/失敗判定例
execute_skill_with_retry() {
    local skill=$1
    local max_retries=3
    local count=0
    
    while [ $count -lt $max_retries ]; do
        if openclaw skills run "$skill"; then
            echo "✅ $skill 成功"
            return 0
        else
            count=$((count + 1))
            echo "❌ $skill 失敗 (試行 $count/$max_retries)"
            sleep 60
        fi
    done
    
    echo "🚨 $skill 最大試行回数に達しました"
    return 1
}
```

## Step 4: 監視とメトリクス

### Slack通知設定
```bash
# 成功率レポート例
report_daily_metrics() {
    local success_count=$(grep "✅" /var/log/openclaw.log | wc -l)
    local total_count=$(grep -E "(✅|❌)" /var/log/openclaw.log | wc -l)
    local success_rate=$(echo "scale=1; $success_count * 100 / $total_count" | bc)
    
    openclaw message send --channel slack --target 'C091G3PKHL2' \
        --message "📊 日次レポート: 成功率 ${success_rate}% (${success_count}/${total_count})"
}
```

### 主要メトリクス
| 項目 | 目標値 | 実測値（4日間平均） |
|------|--------|------------------|
| cron成功率 | 80%+ | ~70% |
| 自動復旧時間 | <5分 | 2分未満 |
| API応答時間 | <3秒 | 1.5秒平均 |

## Step 5: Buddhist Principlesマーケティング自動化

### コンテンツ戦略
```json
{
  "approach": "共感先行、押し売り回避",
  "themes": [
    "マインドフルネス実践法",
    "科学的根拠に基づく不安軽減",
    "3分深呼吸の効果実証"
  ],
  "delivery_frequency": "morning: 9:00, evening: 21:00"
}
```

## まとめ

| 教訓 | 詳細 |
|------|------|
| Mac Miniの安定性 | 4日間連続稼働、停電→再起動→自動復旧確認済み |
| 並列実行設計 | 複数スキルが衝突せず独立実行可能 |
| API依存性管理 | Blotato安定、メッセージ配信は要改善 |
| Buddhist Marketing | 押し売りではない共感アプローチが効果実証中 |
| 監視の重要性 | Slack通知により問題の早期発見が可能 |

**70%成功率は改善余地あり**。目標80%に向けて、エラーハンドリング強化とリトライ機構の導入が次のステップ。