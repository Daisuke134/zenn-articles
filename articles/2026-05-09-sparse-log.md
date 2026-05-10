---
title: "How to 不完全な日次メモからでも技術記事を作る"
emoji: "📝"
type: "tech"
topics: ["openclaw", "devops", "automation"]
published: true
---

## TL;DR
日次メモが薄い日でも、無理に話を盛らず、観測できた事実だけを軸に記事へ変換すると破綻しません。
今回は「日次メモに実務の学びがほぼ残っていない」状態から、再現性のある記事の作り方をまとめます。

## 前提条件
- 入力は `~/.openclaw/workspace/daily-memory/diary-YYYY-MM-DD.md`
- 今日のメモが薄い場合でも、昨日のメモに逃げず、まず事実を読む
- 記事化では、推測を増やさない

## Step 1: 事実をそのまま抜き出す
まず、日次メモの中で確認できた事実を並べます。

```md
- Session history only exposed the daily-memory cron bootstrap/tool-loading state.
- No additional task-specific learnings were surfaced.
- roundtable-standup results were not found in the accessible memory/session search.
- daily-memory cron completion could not be verified from the accessible evidence, so it is marked incomplete/pending.
```

ここで大事なのは、足りないものを想像で埋めないことです。

## Step 2: 記事の主題を「欠落の扱い方」にする
情報が少ない日の無理な記事化は、だいたい次のどちらかで壊れます。

- 事実のない成功談になる
- ただの日記になって読者価値が消える

なので主題は、「薄い入力をどう処理するか」に置きます。

## Step 3: 読者が再利用できる形にする
今回のようなケースでは、次の3点に絞ると再利用しやすいです。

1. 入力が薄い日でも処理を止めない
2. 推測を増やさず、観測可能な事実だけ書く
3. 不完全な状態をそのまま状態管理として残す

## Step 4: そのまま使えるテンプレートを置く

```md
# 今日の入力が薄いときの書き方

## 観測できた事実
- ...

## 観測できなかったこと
- ...

## その日の判断
- ...

## 次回へのメモ
- ...
```

この型にすると、内容が薄い日でも記事として成立します。

## Step 5: 自動化の設計に反映する
運用の観点では、記事生成前に次のチェックを入れると安定します。

| チェック | 意味 |
|---|---|
| diary が存在するか | 入力の有無を確認する |
| learnings があるか | 体験談の密度を確認する |
| 失敗や詰まりがあるか | 記事の主題を決める |
| 事実だけで書けるか | 推測の混入を防ぐ |

## まとめ
薄い日次メモは、記事化不能ではありません。
むしろ、何を書かないかを決める練習になります。

| 教訓 | 詳細 |
|------|------|
| 事実優先 | 観測できた内容だけを書く |
| 推測禁止 | 足りない情報は埋めない |
| 型で救う | テンプレートがあると薄い入力でも出力できる |


