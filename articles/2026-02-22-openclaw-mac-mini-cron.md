---
title: "AIエージェントをVPSからMac Miniに移行したら全43cronジョブが壊れた話"
emoji: "💻"
type: "tech"
topics: ["openclaw", "macos", "devops", "ai", "cronジョブ"]
published: true
---

# TL;DR

Ubuntu VPS（`/home/anicca`）で動いていたAIエージェント「Anicca」をMac Mini（`/Users/anicca`）に移行したら、43個のcronジョブが全部おかしくなった。根本原因は**パスのハードコードではなく `delivery.mode: "announce"` だった**。今日はその調査と修正の話を書く。

## 背景

[OpenClaw](https://openclaw.dev) というLLMエージェントフレームワークで自律AIエージェント「Anicca」を運用している。毎朝のメトリクスレポート、Twitterへの投稿、Slack通知、アプリユーザーへのNudge送信など43個のcronジョブを実行している。

これをLinux VPS（Ubuntu 22.04, `anicca@46.225.70.241`）からMac Mini（macOS 15.6, `anicca@100.99.82.95`）に移行することにした。理由はコストと安定性。VPSは月額課金だが、Mac Miniは電気代だけ。

## 何が起きたか

移行後、cronジョブを一通り手動テストしたところ、複数のスキルが期待通りに動いていないことが判明した。

### 症状

```
- Slack に届くメッセージのフォーマットが崩れている
- 一部のスキルがエラーなく終了しているのに結果が届かない
- x-poster が「NXDOMAIN」エラーで失敗している
```

最初は `/home/anicca` のパスがハードコードされているせいだと疑っていた。実際、VPS上のパスを Mac Mini にそのままコピーすると、シェルスクリプトやSKILL.md内の絶対パスが `/home/anicca/...` のままになっていた。

## 調査: 真の根本原因

`cron/jobs.json` を全件確認したところ、全43ジョブの設定に以下が入っていた。

```json
{
  "delivery": {
    "mode": "announce"
  }
}
```

**これが真犯人だった。**

OpenClaw の `delivery.mode` には2種類ある。

| モード | 動作 |
|--------|------|
| `announce` | エージェントの応答を自動でSlackに投稿する。フォーマット制御不可 |
| `none` | エージェントの応答は投稿しない。スキル内で `exec` を使って自前でSlack投稿する |

VPS時代は `announce` でも動いていた。なぜかというと、Slack APIのレスポンス処理がLinuxとmacOSで微妙に違うわけではなく、**移行作業中に `announce` モードでエージェントを再起動した結果、セッション管理が初期化されてメッセージ配信のコンテキストが壊れていた**からだ。

さらに、`announce` モードはエージェントの「思考過程」も全部Slackに垂れ流すため、移行後の動作確認中に大量のデバッグ出力がSlackに届いていた。これがフォーマット崩れの原因だった。

## 修正

全43ジョブを `delivery.mode: "none"` に変更した。

```bash
# jq で一括置換
jq '
  .jobs |= map(
    .delivery.mode = "none"
  )
' ~/.openclaw/cron/jobs.json > /tmp/jobs_fixed.json
mv /tmp/jobs_fixed.json ~/.openclaw/cron/jobs.json
```

各スキルの SKILL.md 内では、Slack投稿を明示的に `exec` で行うように設計してある。

```
Step 6: Slack #metrics に報告（MANDATORY）
  openclaw message send --channel slack --target "C091G3PKHL2" \
    --message "..."
```

## ボーナスバグ: Blotato APIの廃止

x-poster スキルが `NXDOMAIN` で落ちていた原因は別にあった。

```
ERROR: api.blotato.com: NXDOMAIN
```

Twitter/X への投稿に使っている [Blotato](https://blotato.com) のAPIエンドポイントが変更されていた。

| 状態 | エンドポイント |
|------|--------------|
| 廃止（NXDOMAIN） | `https://api.blotato.com` |
| 現行 | `https://backend.blotato.com/v2` |

SKILL.md の1行を修正するだけで直った。

```diff
- BASE_URL="https://api.blotato.com"
+ BASE_URL="https://backend.blotato.com/v2"
```

## 教訓

| 教訓 | 詳細 |
|------|------|
| `announce` は開発時限定 | 本番cronは全部 `none` に統一する |
| 移行時はエンドポイントも再確認 | 外部APIは予告なく変わる |
| 一括置換は `jq` で | sed より安全。JSONを壊さない |
| VPSはフォールバックとして残す | Mac Mini障害時に `systemctl --user start` で即復旧 |

## 現在の構成

```
ダイス（どこでも）
    ↓ openclaw tui
Mac Mini（家） ← Anicca稼働中（43 cronジョブ）
    フォールバック:
    VPS（46.225.70.241） ← 停止中、enabled状態（即起動可能）
```

今はMac Miniがメインで動いており、VPSはホットスタンバイとして残してある。Mini障害時は3コマンドで切り戻せる。

---

移行作業自体は2日かかったが、`delivery.mode` の問題に気づいたのが一番の収穫だった。VPS時代から `announce` を使っていたのに問題が出なかったのは、当時はスキルが少なくてSlackの出力量が許容範囲内だったから。スケールすると問題が顕在化する、典型的なパターンだった。
