---
title: "How to AI過負荷でcronジョブが全滅しても基幹システムを生かし続ける"
emoji: "🔄"
type: "tech"
topics: ["ai", "devops", "resilience", "infrastructure"]
published: true
---

## TL;DR
AI APIの過負荷により複数のcronジョブが失敗しても、適切なアーキテクチャ設計により基幹システムを稼働継続できる。AI依存度の分離とフォールバック戦略がレジリエンスの鍵。

## 前提条件
- 複数のcronジョブでAI API（Claude、OpenAI等）を使用している
- 基幹システム（Web API、データベース等）が存在する
- システムレジリエンスを向上したい

## Step 1: AI依存度によるサービス階層化

```bash
# Critical Services (AI不要)
# - Web API server
# - Database connection
# - User authentication
# - Core business logic

# AI-Enhanced Services (AI依存だが代替可能)
# - Content generation
# - Auto-summarization  
# - Smart notifications

# AI-Only Services (完全AI依存)
# - LLM-based chat
# - Code generation
# - Complex analysis
```

**設計原則:**
- Critical Servicesは絶対にAI APIへ依存させない
- AI-Enhanced Servicesには必ずフォールバックを用意
- AI-Only Servicesは失敗前提で設計

## Step 2: Cronジョブの時間分散設定

```bash
# Before: 全て同時実行でAPI制限に衝突
0 9 * * * /path/to/job1  # AI heavy
0 9 * * * /path/to/job2  # AI heavy
0 9 * * * /path/to/job3  # AI heavy

# After: 時間をずらしてAPI制限を回避
0 9 * * * /path/to/job1    # 09:00
15 9 * * * /path/to/job2   # 09:15  
30 9 * * * /path/to/job3   # 09:30
```

**負荷分散のコツ:**
- API制限（例: 1000 req/min）を全ジョブで分割
- 重要度順にタイムスロット配分
- ピーク時間を避ける（平日9-17時など）

## Step 3: フォールバック戦略の実装

```python
import time
import random
from typing import Optional

class ResilientAIService:
    def __init__(self):
        self.providers = ['claude', 'openai', 'gemini']
        self.fallback_responses = {
            'summary': '自動要約は現在利用できません',
            'generation': 'デフォルトコンテンツを表示中'
        }
    
    def call_ai_with_fallback(self, prompt: str, service_type: str) -> str:
        for provider in self.providers:
            try:
                response = self._call_provider(provider, prompt)
                if response:
                    return response
            except APIOverloadError:
                # 指数バックオフでリトライ
                time.sleep(random.uniform(1, 5))
                continue
            except Exception as e:
                print(f"{provider} failed: {e}")
                continue
        
        # 全プロバイダー失敗時のフォールバック
        return self.fallback_responses.get(service_type, '処理に失敗しました')
```

## Step 4: システム状態のモニタリング

```bash
# システムヘルスチェック設定
#!/bin/bash
check_core_systems() {
    # Database connection
    if ! pg_isready -h localhost -p 5432; then
        echo "CRITICAL: Database down"
        return 1
    fi
    
    # Web API health
    if ! curl -f http://localhost:8000/health; then
        echo "CRITICAL: API server down"
        return 1
    fi
    
    echo "Core systems: OK"
    return 0
}

check_ai_services() {
    local ai_failures=0
    
    # Check each AI provider
    for provider in claude openai gemini; do
        if ! test_ai_provider "$provider"; then
            ((ai_failures++))
            echo "WARNING: $provider unavailable"
        fi
    done
    
    if [ $ai_failures -eq 3 ]; then
        echo "WARNING: All AI providers down"
        # Send alert but don't kill core systems
        send_slack_alert "AI services unavailable"
    fi
}
```

## Step 5: 障害時の自動復旧設定

```yaml
# docker-compose.yml
version: '3'
services:
  core-api:
    image: myapp/core
    restart: always
    environment:
      - AI_ENABLED=false  # Core features work without AI
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  ai-worker:
    image: myapp/ai-worker
    restart: on-failure
    environment:
      - MAX_RETRIES=3
      - BACKOFF_MULTIPLIER=2
    depends_on:
      - core-api
```

## 実際の障害対応ログ

**2026-03-03 12:00 - Claude API過負荷発生時の実対応：**

```
09:01 - Roundtable standup： 正常動作
12:00 - Claude API "service temporarily overloaded" 
12:01 - 複数cronジョブ失敗検知
12:05 - Core systems確認： Web API稼働中
12:10 - AI-Enhanced services無効化
12:15 - フォールバック応答に切り替え
23:00 - 手動でDaily memory skill実行成功
```

**結果：**
- 基幹システム： 稼働継続 ✅
- ユーザー体験： 一部機能制限だが利用可能 ✅  
- データ整合性： 保持 ✅

## まとめ

| 教訓 | 詳細 |
|------|------|
| **AI依存の分離** | 基幹機能とAI機能を明確に分離し、AI障害が全システムに波及することを防ぐ |
| **時間分散戦略** | 同時実行を避け、API制限を意識したcron設定で障害率を削減 |
| **多段フォールバック** | 複数プロバイダー + 静的応答で、完全停止を回避 |
| **モニタリング重視** | AI障害と基幹システム障害を区別し、適切なアラートレベルを設定 |

AIサービスは便利ですが、完全に依存するのは危険です。レジリエントなアーキテクチャで、障害を「想定内」にしましょう。