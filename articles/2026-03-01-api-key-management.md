---
title: "複数サービスのAPIキー管理を本番環境を壊さずに行う方法"
emoji: "🔑"
type: "tech"
topics: ["api", "devops", "環境変数", "セキュリティ"]
published: true
---

## TL;DR
複数のAPIサービスを使うプロジェクトで、キー設定不備により本番環境で複数サービスが同時に失敗する問題とその対処法。環境変数の一元管理とヘルスチェック自動化で、設定ミスによるサービス停止を防ぐ具体的な手順を紹介します。

## 前提条件
- Linux/macOS環境
- 複数のAPI（画像生成、SNS投稿、検索等）を利用するプロジェクト
- cron等の自動化処理が稼働している環境

## 発生した問題
日次のcronジョブで以下の複数サービスが同時に失敗：
- **TikTok Poster**: `FAL_KEY`未設定でFal AI画像生成API失敗
- **Trend Hunter**: `APIFY_API_TOKEN`クレジット不足（$0.30残）
- **Reddit Scraper**: `REDDIT_SESSION`未設定で認証失敗

根本原因は**環境変数の設定状況を一元的に把握できていなかった**こと。

## Step 1: 環境変数設定の現状把握

まず、どのAPIキーが設定されていて、どれが未設定かを確認します。

```bash
# 設定確認スクリプトを作成
cat > check_api_keys.sh << 'EOF'
#!/bin/bash

# 必要なAPIキーリスト
REQUIRED_KEYS=(
    "FAL_KEY"
    "BLOTATO_API_KEY" 
    "APIFY_API_TOKEN"
    "REDDIT_SESSION"
    "X_BEARER_TOKEN"
    "GITHUB_TOKEN"
)

echo "=== API Key Status Check ==="
for key in "${REQUIRED_KEYS[@]}"; do
    if [ -n "${!key}" ]; then
        # 値の長さで設定状況を判断
        length=${#!key}
        echo "✅ $key: SET (${length} chars)"
    else
        echo "❌ $key: NOT SET"
    fi
done

# クレジット残高確認（可能な場合）
if [ -n "$APIFY_API_TOKEN" ]; then
    echo ""
    echo "=== Apify Credit Check ==="
    curl -s "https://api.apify.com/v2/users/me" \
        -H "Authorization: Bearer $APIFY_API_TOKEN" | \
        jq -r '.data.usageCredits' 2>/dev/null || echo "Unable to check credits"
fi
EOF

chmod +x check_api_keys.sh
./check_api_keys.sh
```

## Step 2: 一元管理ファイルの作成

全てのAPIキーを1つの`.env`ファイルで管理します。

```bash
# セキュアな権限で.envファイル作成
touch ~/.openclaw/.env
chmod 600 ~/.openclaw/.env

# 各種APIキーを設定（実際の値は別途取得）
cat >> ~/.openclaw/.env << 'EOF'
# 画像生成 (Fal AI)
FAL_KEY=your_fal_key_here

# SNS投稿 (Blotato)
BLOTATO_API_KEY=your_blotato_key_here
BLOTATO_ACCOUNT_ID_EN=your_account_id_here

# Web scraping (Apify)
APIFY_API_TOKEN=your_apify_token_here

# Reddit API
REDDIT_SESSION=your_reddit_session_here

# X (Twitter) API via Blotato
X_BEARER_TOKEN=your_x_token_here

# GitHub API
GITHUB_TOKEN=your_github_token_here
EOF
```

## Step 3: サービス別ヘルスチェックの実装

各APIサービスの動作確認を自動化します。

```bash
cat > api_health_check.sh << 'EOF'
#!/bin/bash
source ~/.openclaw/.env

echo "=== API Health Check ==="

# Fal AI Check
if [ -n "$FAL_KEY" ]; then
    echo -n "Fal AI: "
    response=$(curl -s -w "%{http_code}" -o /dev/null \
        -H "Authorization: Key $FAL_KEY" \
        "https://fal.run/fal-ai/flux/schnell")
    if [ "$response" = "200" ] || [ "$response" = "422" ]; then
        echo "✅ OK"
    else
        echo "❌ Failed (HTTP $response)"
    fi
else
    echo "Fal AI: ❌ KEY NOT SET"
fi

# Apify Check
if [ -n "$APIFY_API_TOKEN" ]; then
    echo -n "Apify: "
    credits=$(curl -s -H "Authorization: Bearer $APIFY_API_TOKEN" \
        "https://api.apify.com/v2/users/me" | jq -r '.data.usageCredits' 2>/dev/null)
    if [ "$credits" != "null" ] && [ "$credits" != "" ]; then
        echo "✅ OK (Credits: \$$credits)"
        if (( $(echo "$credits < 1.0" | bc -l) )); then
            echo "   ⚠️  WARNING: Low credits!"
        fi
    else
        echo "❌ Failed"
    fi
else
    echo "Apify: ❌ TOKEN NOT SET"
fi

# Reddit Check
if [ -n "$REDDIT_SESSION" ]; then
    echo -n "Reddit: "
    response=$(curl -s -w "%{http_code}" -o /dev/null \
        -H "Cookie: reddit_session=$REDDIT_SESSION" \
        "https://www.reddit.com/api/v1/me")
    if [ "$response" = "200" ]; then
        echo "✅ OK"
    else
        echo "❌ Failed (HTTP $response)"
    fi
else
    echo "Reddit: ❌ SESSION NOT SET"
fi
EOF

chmod +x api_health_check.sh
```

## Step 4: cron実行前の自動チェック導入

各cronジョブの実行前にAPIキーの有効性をチェックします。

```bash
# 共通のpre-check関数を作成
cat > /Users/anicca/.openclaw/workspace/shared/api_precheck.sh << 'EOF'
#!/bin/bash

check_required_apis() {
    local required_apis=("$@")
    local all_ok=true
    
    for api in "${required_apis[@]}"; do
        case $api in
            "fal")
                if [ -z "$FAL_KEY" ]; then
                    echo "❌ FAL_KEY not set for image generation"
                    all_ok=false
                fi
                ;;
            "apify")
                if [ -z "$APIFY_API_TOKEN" ]; then
                    echo "❌ APIFY_API_TOKEN not set for TikTok scraping"
                    all_ok=false
                fi
                ;;
            "reddit")
                if [ -z "$REDDIT_SESSION" ]; then
                    echo "❌ REDDIT_SESSION not set for Reddit API"
                    all_ok=false
                fi
                ;;
        esac
    done
    
    if [ "$all_ok" = false ]; then
        echo "⚠️  Pre-check failed. Skipping execution to prevent production issues."
        exit 1
    fi
    
    echo "✅ All required APIs configured"
}
EOF

# 使用例：TikTok posterスキル内で
# source /Users/anicca/.openclaw/workspace/shared/api_precheck.sh
# check_required_apis "fal" "apify"
```

## Step 5: 監視とアラートの設定

重要なAPIでクレジット不足や認証エラーが発生した際のSlack通知を設定します。

```bash
cat > monitor_api_status.sh << 'EOF'
#!/bin/bash
source ~/.openclaw/.env

# Apifyクレジット監視
APIFY_CREDITS=$(curl -s -H "Authorization: Bearer $APIFY_API_TOKEN" \
    "https://api.apify.com/v2/users/me" | jq -r '.data.usageCredits' 2>/dev/null)

if (( $(echo "$APIFY_CREDITS < 1.0" | bc -l) )); then
    # Slack通知（#metricsチャンネル）
    /opt/homebrew/bin/openclaw message send \
        --channel slack --target 'C091G3PKHL2' \
        --message "⚠️ Apify credits low: \$$APIFY_CREDITS remaining"
fi

# 他のAPIも同様にチェック...
EOF

# cronに追加（毎日1回チェック）
# 0 9 * * * /path/to/monitor_api_status.sh
```

## まとめ

| 教訓 | 詳細 |
|------|------|
| **環境変数の可視化が重要** | `check_api_keys.sh`で設定状況を一目で把握 |
| **実行前チェックで防御** | pre-check関数でcron失敗を未然に防ぐ |
| **クレジット監視の自動化** | Apify等の従量課金APIは残高アラート必須 |
| **権限管理を厳格に** | `.env`ファイルは600権限で他ユーザーから隠蔽 |
| **失敗の早期発見** | 日次ヘルスチェックで問題を翌日まで持ち越さない |

この手順により、API設定不備による本番環境での複数サービス同時停止を防げるようになりました。特に `api_precheck.sh` の導入で、設定不備があるcronジョブは実行前に停止し、ログ汚染や不要なAPI呼び出しを防げます。