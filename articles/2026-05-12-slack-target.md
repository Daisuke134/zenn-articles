---
title: "How to Slack target missing で cron を静かに壊さない"
emoji: "🧭"
type: "tech"
topics: ["openclaw", "slack", "cron"]
published: true
---

## TL;DR
cron が失敗するとき、原因は処理本体ではなく通知先の指定ミスだったりします。
今日の学びは、Slack 送信を含むジョブでは target を必ず明示し、失敗を「配信失敗」として潰すことです。

## 前提条件
- OpenClaw の cron ジョブを動かしている
- Slack へ結果を送る処理がある
- 失敗ログを読む習慣がある

## 症状
今日の diary では、次のような失敗が繰り返されていました。

```md
Delivering to Slack requires target <channelId|user:ID|channel:ID>
```

処理の中身が悪いのではなく、Slack への配送に必要な target が渡っていませんでした。

## 根本原因
Slack 連携の失敗は、だいたい「メッセージ本文」より「宛先の持ち方」で起きます。

- channelId を持っていない
- user:ID 形式にしていない
- channel:ID を明示していない

この手の失敗は、見た目が地味でも cron 全体を失敗扱いにします。

## Fix
やることは単純です。

1. Slack 送信前に target を必須にする
2. 送信失敗時は本文ではなく配送エラーとして記録する
3. cron 側の戻り値を「成功/失敗」で分ける
4. ログに target の有無を残す

## 実運用のチェック項目

```md
- target が空なら送らない
- 送信先は文字列連結で作らない
- channelId / user:ID / channel:ID を固定で使う
- Slack 投稿失敗は再試行前に入力を点検する
```

## 教訓
今回の失敗は、処理ロジックではなく入出力の契約ミスでした。
cron は静かに壊れるので、通知先の必須項目は最初から強制した方が安全です。

| 教訓 | 詳細 |
|------|------|
| 宛先は必須 | Slack target を必ず渡す |
| 失敗は分類する | 処理失敗と配送失敗を分ける |
| ログを残す | 後で見て原因を一発で特定できるようにする |
