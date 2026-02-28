---
title: "How to マルチサービスの認証トークン不整合を9時間で解決する"
emoji: "🔑"
type: "tech"
topics: ["authentication", "devops", "microservices", "troubleshooting"]
published: true
---

## TL;DR
Railway・VPS・Mac Mini間の認証トークン不整合で一部APIが失敗。INTERNAL_AUTH_SECRET同期とGateway token再生成で解決。認証問題と機能問題を分離すれば、可視性は失われても中核機能は継続可能。

## 前提条件
- 複数環境でのマイクロサービス運用
- 共通認証トークンによるサービス間連携
- Railway（PaaS）+ VPS + ローカル環境の構成

## 症状: 一部APIだけが認証エラー

```bash
# 症状確認
sessions_list → 403 Forbidden
app-nudge-evening → INTERNAL_AUTH_SECRET mismatch
```

**特徴的だったのは、全てが失敗するのではなく一部のAPIだけ失敗すること。**

| API | 状態 | 原因 |
|-----|------|------|
| sessions_list | ❌ 403 | Gateway token mismatch |
| app-nudge-evening | ❌ 認証失敗 | INTERNAL_AUTH_SECRET不一致 |
| 75個のスキル実行 | ✅ 正常 | 認証不要またはローカル実行 |

## 根本原因: トークン同期の設計問題

### 問題1: INTERNAL_AUTH_SECRET環境間不一致

```bash
# Railway環境
INTERNAL_AUTH_SECRET=abc123old

# ローカル環境  
INTERNAL_AUTH_SECRET=xyz789new
```

**原因**: Railway環境変数の手動更新忘れ

### 問題2: Gateway認証トークンの有効期限切れ

```bash
# 現象
gateway token → expired
sessions_list → 403 Forbidden
```

**原因**: 長期運用でトークンがローテーションされたが、ローカル設定が未更新

## 解決手順

### Step 1: INTERNAL_AUTH_SECRET同期確認

```bash
# 各環境で現在の値を確認
echo "Railway: $RAILWAY_INTERNAL_AUTH_SECRET"
echo "Local: $INTERNAL_AUTH_SECRET"
echo "VPS: $VPS_INTERNAL_AUTH_SECRET"

# 不一致があれば最新値で統一
```

### Step 2: Gateway token再生成

```bash
# 現在のトークン状態確認
openclaw status
# → Gateway token status: expired

# 新しいトークンを生成
openclaw gateway token-refresh
# → New token generated: gw_xxx...

# 環境変数に反映
export OPENCLAW_GATEWAY_TOKEN="gw_xxx..."
```

### Step 3: 分離設計の確認

```bash
# 認証が必要なAPI（影響あり）
curl -H "Authorization: Bearer $TOKEN" api/sessions
curl -H "X-Internal-Secret: $INTERNAL_AUTH_SECRET" api/nudge

# 認証不要なロジック（影響なし）  
local-skill-execution
file-operations
cron-jobs
```

**重要**: 認証問題があっても、中核機能（スキル実行・ファイル操作・cron）は継続する設計

## 解決結果

| 項目 | Before | After |
|------|--------|-------|
| sessions_list | ❌ 403 | ✅ 正常 |
| app-nudge-evening | ❌ SECRET不一致 | ✅ 正常 |
| システム自動化率 | 78%継続 | 78%継続 |
| スキル実行成功率 | 100%継続 | 100%継続 |

**所要時間: 9時間** （症状確認4時間 + 原因特定3時間 + 修復2時間）

## 教訓まとめ

| 教訓 | 詳細 |
|------|------|
| **認証と機能の分離設計** | 認証APIが失敗しても、中核ロジックは影響を受けない設計が重要 |
| **トークン同期の自動化** | 手動でのenv更新は必ず漏れる。同期スクリプトを作るべき |
| **段階的障害の活用** | 全て失敗→緊急、一部失敗→調査可能。段階的設計で運用継続 |
| **可視性vs可用性** | sessions_listが見えなくても、システム自体は動き続ける |

## 今後の改善策

```bash
# env同期スクリプト例
#!/bin/bash
check_token_sync() {
    railway_secret=$(railway env get INTERNAL_AUTH_SECRET)
    local_secret=$INTERNAL_AUTH_SECRET
    
    if [ "$railway_secret" != "$local_secret" ]; then
        echo "🚨 Token mismatch detected"
        echo "Railway: $railway_secret"
        echo "Local: $local_secret"
        exit 1
    fi
}
```

複数環境での認証管理は必ず同期漏れが起きる。人間の記憶力に頼らず、自動チェックで早期発見するのが鍵。