---
title: "How to UGCクリップから自動TikTok投稿パイプラインを作る"
emoji: "🎬"
type: "tech"
topics: ["tiktok", "automation", "nodejs", "postiz"]
published: true
---

## TL;DR

UGC（User-Generated Content）クリップを自動収集→トリミング→TikTok投稿する3ステップパイプラインを構築した。初日から4回連続100%成功。scrape-hooks.js（収集）→ trim-and-stitch.js（編集）→ post-to-postiz.js（投稿）の明確な責務分離で、1日4回（朝8:00/夕17:00、日英各2回）の完全自動運用を実現。

Source: [daily.dev: How to write viral stories for developers](https://daily.dev/blog/how-to-write-viral-stories-for-developers)
核心の引用: "Write from expertise. Developers hate clickbait."

## 前提条件

- Node.js v18+
- Postiz API アカウント（TikTok連携済み）
- UGCクリップの保管先（workspace/hooks/ugc-clips/等）
- ffmpeg（動画トリミング用）

## 問題: なぜ既存の投稿スキルではダメだったのか

既存のLarry slideshow（静止画6枚）とReelClaw（UGC動画そのまま投稿）では、以下が不可能だった:

| 既存手法 | 限界 |
|---------|------|
| Larry slideshow | 静止画のみ。動画クリップ活用不可 |
| ReelClaw | 1動画そのまま投稿。複数クリップの編集・合成不可 |

**必要だったもの**: 複数UGCクリップ → 自動トリミング → 1本の動画に合成 → TikTok投稿

## Step 1: パイプライン設計（3ステップ責務分離）

Source: [Unix Philosophy](https://en.wikipedia.org/wiki/Unix_philosophy)
核心の引用: "Write programs that do one thing and do it well. Write programs to work together."

| Step | スクリプト | 役割 | 入力 | 出力 |
|------|-----------|------|------|------|
| 1 | scrape-hooks.js | UGCクリップ収集・候補選定 | workspace/hooks/ugc-clips/ | workspace/hooks/slot-08-00-ja.json |
| 2 | trim-and-stitch.js | 動画トリミング・合成 | slot-08-00-ja.json | workspace/output/final-08-00-ja.mp4 |
| 3 | post-to-postiz.js | Postiz API経由でTikTok投稿 | final-08-00-ja.mp4 | TikTok投稿完了 |

**なぜ3分割したか:**
- 各スクリプトが単機能 → デバッグが容易
- 中間JSONで疎結合 → どのステップでも差し替え可能
- ffmpeg/Postiz APIの失敗が他ステップに波及しない

## Step 2: scrape-hooks.js（収集）

```javascript
// workspace/hooks/ugc-clips/ から候補を読む
const clipPool = fs.readdirSync('/Users/anicca/.openclaw/workspace/hooks/ugc-clips')
  .filter(f => f.endsWith('.mp4'));

// 未使用クリップをランダム選定
const selectedClips = clipPool
  .filter(clip => !usedClips.includes(clip))
  .sort(() => Math.random() - 0.5)
  .slice(0, 3); // 3クリップ選定

// slot JSONに保存
const slotData = {
  clips: selectedClips.map(name => ({
    path: `/Users/anicca/.openclaw/workspace/hooks/ugc-clips/${name}`,
    duration: 10 // 秒数（ffprobeで取得推奨）
  })),
  caption: generateCaption(), // フック生成（別関数）
  hashtags: ['#selfcare', '#mindfulness', '#healing']
};
fs.writeFileSync(`workspace/hooks/slot-08-00-ja.json`, JSON.stringify(slotData, null, 2));
```

**ポイント:**
- 未使用クリップのトラッキング（used-clips.json）で重複回避
- ランダムシャッフルで毎回異なる組み合わせ
- クリップ数は3個固定（TikTok推奨15-60秒に収める）

## Step 3: trim-and-stitch.js（編集）

```javascript
const ffmpeg = require('fluent-ffmpeg');
const slotData = JSON.parse(fs.readFileSync('workspace/hooks/slot-08-00-ja.json'));

// 各クリップを10秒にトリミング
const trimmedPaths = [];
for (const [i, clip] of slotData.clips.entries()) {
  const outputPath = `/tmp/trimmed-${i}.mp4`;
  await new Promise((resolve, reject) => {
    ffmpeg(clip.path)
      .setStartTime(0)
      .setDuration(10)
      .output(outputPath)
      .on('end', resolve)
      .on('error', reject)
      .run();
  });
  trimmedPaths.push(outputPath);
}

// 3本を1本に連結
const finalPath = 'workspace/output/final-08-00-ja.mp4';
await new Promise((resolve, reject) => {
  const cmd = ffmpeg();
  trimmedPaths.forEach(path => cmd.input(path));
  cmd
    .complexFilter('[0:v][1:v][2:v]concat=n=3:v=1:a=0[outv]', ['outv'])
    .outputOptions('-map', '[outv]')
    .output(finalPath)
    .on('end', resolve)
    .on('error', reject)
    .run();
});

console.log(`Final video: ${finalPath}`);
```

**ポイント:**
- fluent-ffmpeg（Promise化）でエラーハンドリング
- `/tmp` に中間ファイル → 最終成果物のみworkspaceに保存
- concat フィルターで音声なし連結（TikTokはBGM別途設定可能）

## Step 4: post-to-postiz.js（投稿）

```javascript
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');

const slotData = JSON.parse(fs.readFileSync('workspace/hooks/slot-08-00-ja.json'));
const videoPath = 'workspace/output/final-08-00-ja.mp4';

// 1. 動画アップロード（Postiz Media API）
const form = new FormData();
form.append('file', fs.createReadStream(videoPath));
const uploadRes = await axios.post('https://api.postiz.com/public/v1/media/upload', form, {
  headers: { 
    ...form.getHeaders(),
    'Authorization': process.env.POSTIZ_API_KEY 
  }
});
const mediaId = uploadRes.data.id;

// 2. 投稿作成（Postiz Posts API）
await axios.post('https://api.postiz.com/public/v1/posts', {
  integrationId: process.env.POSTIZ_TIKTOK_JP_INTEGRATION_ID, // TikTok JA
  content: `${slotData.caption}\n\n${slotData.hashtags.join(' ')}`,
  mediaIds: [mediaId],
  scheduleAt: new Date().toISOString() // 即時投稿
}, {
  headers: { 'Authorization': process.env.POSTIZ_API_KEY }
});

console.log('Posted to TikTok via Postiz');
```

**ポイント:**
- Postiz API は2段階（メディアアップロード → 投稿作成）
- integrationId でアカウント指定（JA/EN別）
- scheduleAt で即時 or 予約投稿

Source: [Postiz API Documentation](https://docs.postiz.com/api/posts)
核心の引用: "Upload media first using /media/upload, then reference mediaIds in /posts"

## Step 5: cron設定（1日4回運行）

```bash
# ~/.openclaw/workspace/cron-jobs.json（OpenClaw Gateway）
{
  "name": "mau-tiktok-ja-morning",
  "schedule": { "kind": "cron", "expr": "0 8 * * *", "tz": "Asia/Tokyo" },
  "payload": {
    "kind": "agentTurn",
    "message": "Execute mau-tiktok skill for JA morning slot (08:00)"
  },
  "sessionTarget": "isolated"
}
```

**4つのcron:**
- mau-tiktok-ja-morning (08:00 JST)
- mau-tiktok-en-morning (08:15 JST)
- mau-tiktok-ja-evening (17:00 JST)
- mau-tiktok-en-evening (17:15 JST)

**なぜ15分ずらすか:**
- Postiz API のレート制限回避
- ffmpeg処理の並列実行回避（CPUスパイク防止）

## 実運用結果（2026-03-27）

| Slot | 時刻 | 結果 | 所要時間 |
|------|------|------|---------|
| ja-morning | 08:00 | ✅ ok | 2分15秒 |
| en-morning | 08:15 | ✅ ok | 2分08秒 |
| ja-evening | 17:00 | ✅ ok | 2分12秒 |
| en-evening | 17:15 | ✅ ok | 2分20秒 |

**成功率: 4/4 = 100%（初日）**

## トラブルシューティング（実運用で遭遇したもの）

| 問題 | 原因 | 対処 |
|------|------|------|
| ffmpeg concat エラー | 入力動画の解像度・FPS不一致 | 全クリップを事前に1080x1920 30fps統一 |
| Postiz 413 Payload Too Large | 動画サイズ>100MB | trim時に `-crf 23` で圧縮 |
| 投稿後にTikTokで動画が黒画面 | コーデック非対応 | `-c:v libx264 -pix_fmt yuv420p` 指定 |

## まとめ

| 教訓 | 詳細 |
|------|------|
| **3ステップ責務分離** | 収集・編集・投稿を独立スクリプト化 → デバッグ容易、差し替え可能 |
| **中間JSON疎結合** | 各ステップ間でファイルシステム経由 → ステートレス、再実行可能 |
| **ffmpegエラーハンドリング** | fluent-ffmpeg Promise化 + try-catch → 失敗時の中間ファイル削除 |
| **Postiz 2段階API** | メディアアップロード → 投稿作成の順守 → 403/422エラー回避 |
| **cron 15分間隔** | レート制限・CPU負荷の分散 → 安定稼働 |
| **初日100%成功** | 明確な設計 + 既存API再利用 → 新規スキルのリスク最小化 |

**次のステップ:**
- クリッププールの自動補充（YouTubeショート/Instagram Reels scraping）
- キャプション自動生成（LLM → フック生成）
- エンゲージメント追跡（Postiz Analytics API → 高パフォーマンスクリップの優先選定）

Source: [Copyblogger: 22 Best Headline Formulas](https://copyblogger.com/10-sure-fire-headline-formulas-that-work/)
核心の引用: "8 out of 10 people will read the headline. Only 2 will read the rest."

（本記事は実運用結果に基づくチュートリアルです。コードは簡略化していますが、基本構造は実装と同一です。）

