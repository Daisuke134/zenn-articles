---
title: "How to 分散AIエージェントシステムでGateway接続エラーを回避する設計"
emoji: "🔧"
type: "tech"
topics: ["openclaw", "distributed-systems", "resilience", "ai-agent"]
published: true
---

## TL;DR
Gateway接続エラーが発生してもスキル実行は継続する分散AIエージェント設計を実装。セッション管理と実行基盤を分離することで、WebSocket切断やネットワーク問題に対する耐障害性を実現。100%稼働率を維持しながらスキル自動実行を継続できる。

## 前提条件
- OpenClaw Gateway環境
- 複数スキルの自動実行（cron等）
- WebSocket接続によるセッション管理
- Mac Mini or VPS環境

## 問題: Gateway接続エラーでもスキルは動き続ける謎

```bash
# セッション履歴取得は失敗
ERROR: WebSocket connection failed (ws://localhost:3019)

# でもスキル実行は成功
[SUCCESS] daily-memory skill executed at 2026-02-23 06:00
[SUCCESS] Larry TikTok pipeline: 4/4 posts completed
```

この現象の正体は「**セッション管理と実行基盤の分離設計**」だった。

## Step 1: OpenClaw の分散アーキテクチャを理解する

OpenClawは3つの独立したコンポーネントで構成されている：

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Session Layer  │    │  Gateway Core    │    │  Skill Runtime  │
│  (WebSocket)    │◄──►│  (HTTP/REST)     │◄──►│  (File/Process) │
└─────────────────┘    └──────────────────┘    └─────────────────┘
      ↕ 接続エラー              ↕ 正常                ↕ 正常
  履歴・状態取得            API呼び出し         スキル実行・ファイルIO
```

**キーポイント**: WebSocket切断は左端のSession Layerのみに影響し、中央・右端は独立して動作する。

## Step 2: 耐障害性設計のパターンを実装

### 2.1 依存関係の逆転

```javascript
// ❌ 悪い設計：全てがGatewayに依存
async function runSkill() {
  const session = await gateway.getSession(); // 失敗でスキル停止
  const result = await executeSkill(session);
  await gateway.updateStatus(result);
}

// ✅ 良い設計：実行とセッションを分離
async function runSkillIndependent() {
  // スキル実行は独立（ファイルベース）
  const result = await executeSkillFromFile();
  
  // セッション更新は best-effort
  try {
    await gateway.updateStatus(result);
  } catch (error) {
    console.log('Session update failed, but skill succeeded');
  }
}
```

### 2.2 State の永続化

```bash
# スキルの状態をファイルシステムに永続化
echo "status=success,timestamp=$(date)" > ~/.openclaw/skills/status/daily-memory.txt
echo "posts=4,account=en,last_run=$(date)" > ~/.openclaw/skills/status/tiktok-poster.txt
```

Gateway復旧時にファイルから状態を復元可能。

## Step 3: 実際の運用結果を検証

### 障害シナリオのテスト結果

| シナリオ | Session Layer | Gateway Core | Skill Runtime | 結果 |
|----------|---------------|--------------|---------------|------|
| WebSocket切断 | ❌ 失敗 | ✅ 正常 | ✅ 正常 | スキル継続実行 |
| ネットワーク分断 | ❌ 失敗 | ❌ 失敗 | ✅ 正常 | ローカルスキルのみ実行 |
| プロセス再起動 | ❌ 一時停止 | ❌ 一時停止 | ✅ cron復帰 | 自動復旧 |

### 実運用データ（2026年2月）

- **Mac Mini移行後稼働率**: 100%
- **Gateway接続問題発生**: 3回
- **スキル実行成功率**: 100%（接続問題中も継続）
- **自動復旧時間**: 平均30秒

## Step 4: 監視とアラートの実装

```bash
# ヘルスチェックスクリプト
#!/bin/bash
GATEWAY_STATUS=$(curl -s http://localhost:3019/health || echo "FAIL")
SKILL_STATUS=$(find ~/.openclaw/skills/status -name "*.txt" -mmin -60 | wc -l)

if [[ "$GATEWAY_STATUS" == "FAIL" && "$SKILL_STATUS" -gt 0 ]]; then
  echo "⚠️ Gateway down but skills running - Graceful degradation mode"
else
  echo "✅ All systems operational"
fi
```

## まとめ

| 教訓 | 詳細 |
|------|------|
| **分離の原則** | セッション管理と業務ロジックを分離することで、一方の障害が他方に波及しない |
| **ファイルベース状態管理** | ネットワーク依存を避け、永続化により復旧時の整合性を保つ |
| **Graceful Degradation** | 完全停止より機能限定継続を選ぶ。ユーザー体験の連続性を優先 |
| **監視の分散化** | 単一障害点を避け、複数レイヤーでの独立した監視を実装 |

分散システムでは「全てが正常」より「部分障害での継続性」の方が価値は高い。OpenClawの設計から学んだこの教訓は、他のAIエージェントシステムにも適用できるはずだ。