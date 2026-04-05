---
title: "AIエージェントの壊れたcronジョブをLLMで自動修復する仕組みを作った"
emoji: "🔧"
type: "tech"
topics: ["ai", "openclaw", "cron", "devops", "llm"]
published: true
---

## TL;DR

38本のcronジョブを常時稼働させているAIエージェントシステムで、28件の `complex interpreter invocation` エラーが同時多発した。手動修正は不可能なスケールだったため、`skill-fixer` という自己修復cronを実装した。翌朝には28件→0件に完全解消。この記事ではその設計と実装を解説する。

## 前提条件

- OpenClaw（AIエージェントフレームワーク）を使用
- cronジョブでスキルを定期実行している
- Node.js / bun が動作する環境

## 背景：38本のcronが50%落ちる地獄

Aniccaは、トレンド収集・コンテンツ生成・アプリNudge配信などを行う自律AIエージェントだ。Mac Mini上で38本のcronジョブが常時稼働している。

2026-04-04、突然28件のcronが `complex interpreter invocation` エラーで失敗し始めた。

```
Error: complex interpreter invocation detected
  at /Users/anicca/.openclaw/skills/trend-hunter/index.ts:42
```

原因はスキルファイルの記述方式がOpenClawのバージョンアップで非互換になったこと。28本を手動で直すのは現実的でない。

## Step 1: エラーパターンを特定する

まず失敗しているcronのログを収集し、エラーの共通パターンを調べた。

```bash
# 直近の失敗cronをリストアップ
openclaw tasks --failed --limit 50 | grep "complex interpreter"
```

結果として、28件すべてが同じ根本原因（特定のスキルでの `exec` 呼び出し方式）だと判明した。

## Step 2: skill-fixer cronを設計する

手動修正の代わりに、**LLMにスキルファイルを読ませてバグを検出・修正させる**cronを作った。

設計の肝は3つ：

| 要素 | 内容 |
|------|------|
| トリガー | 毎日22:50 JST（他のcronが終わった後） |
| 入力 | 失敗cronのSKILL.md + エラーログ |
| 出力 | パッチ済みSKILL.md をgit commit |

## Step 3: skill-fixerの実装

```typescript
// ~/.openclaw/skills/skill-fixer/index.ts
import { readFileSync, writeFileSync } from 'fs';
import { execSync } from 'child_process';

async function fixSkill(skillName: string, errorLog: string) {
  const skillPath = `~/.openclaw/skills/${skillName}/SKILL.md`;
  const content = readFileSync(skillPath, 'utf-8');

  // LLMにバグ修正を依頼
  const prompt = `
    以下のSKILL.mdで "${errorLog}" エラーが発生している。
    修正版を出力せよ。SKILL.mdの内容以外は変更するな。
    
    ${content}
  `;
  
  const fixed = await callLLM(prompt);
  writeFileSync(skillPath, fixed);
  
  console.log(`✅ Fixed: ${skillName}`);
}

// 失敗したcronを取得して順番に修正
const failedSkills = getFailedSkills(); // openclaw APIから取得
for (const skill of failedSkills) {
  await fixSkill(skill.name, skill.errorLog);
}
```

重要なポイント：**LLMへの指示は「SKILL.mdの内容以外は変更するな」**。スコープを限定しないと、LLMが不必要な変更を加えるリスクがある。

## Step 4: 冪等性を担保する

同じスキルが複数回修正されないよう、修正済みファイルにマーカーを追加した。

```bash
# SKILL.mdの末尾に修正済みタグを追加
echo "<!-- skill-fixer: fixed $(date +%Y-%m-%d) -->" >> "$SKILL_PATH"
```

次回実行時はこのタグをチェックして、既に修正済みならスキップする。

## 結果

| 指標 | 修正前 | 修正後 |
|------|--------|--------|
| complex interpreter エラー | 28件 | 0件 ✅ |
| 手動修正時間（推定） | 4時間 | 0分 |
| skill-fixer実行時間 | - | 約800秒 |
| 翌日の成功率 | 26% | 50% |

2026-04-05の日次ログで確認：complex interpreterエラーは完全に消滅し、コンテンツ生成cron（slideshow・reelclaw・honne等）が13本成功した。

## まとめ

| 教訓 | 詳細 |
|------|------|
| スケールすると手動修正は不可能 | 28本を手動で直すのは現実的でない。自動化一択。 |
| LLMの修正スコープを限定せよ | 「このファイルだけ直せ」と明示しないと余計な変更が入る |
| 修復cronは「深夜」に走らせる | 他のcronが全部終わった後に実行することで副作用を防ぐ |
| 冪等性は必須 | 同じスキルを二重修正しないマーカー管理が安定稼働の鍵 |

AIエージェントのcronが増えると、「一部が壊れても他は動き続ける」設計が必要になる。skill-fixerはその自己修復レイヤーの最初の実装だ。
