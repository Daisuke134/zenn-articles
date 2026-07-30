---
title: "緑のテストだけで「学習した」と呼ばない：請求書確認ループの5つの証拠"
emoji: "🧪"
type: "tech"
topics: ["ai", "agents", "testing", "software"]
published: false
---

## 次回起動までを判定する

次の起動が採用版を読んだ証跡まで取れなければ、請求書確認ループを「学習済み」と判定しません。反復タスクでは、次の3段階を分けて記録します。

請求書確認担当者は、末尾の証拠記録チェックリストを使い、次回runの入力hash・採否・startup hashまで取得してから判定してください。

| 段階 | 何が起きたか | 最低限の証拠 |
| --- | --- | --- |
| 単発テスト | 1回の入力を採点した | 入力、出力、評価器の版、score |
| 評価付き更新 | 候補版を固定データで比較し、採用した | base/candidate hash、split、score_before/after、decision |
| 閉じた改善ループ | 次の起動が採用版を読み、次の結果まで結び付いた | startup hash、次回run_id、同じtask_idの結果 |

この記事では、最後の段階だけを「学習した」と呼びます。モデルの重みが変わった、という意味ではありません。採用したプロンプトまたはスキルの版が変わり、その版を次の起動が実際に読んだ、という意味です。

## まず、実運用の請求書observerを1件読む

公開された本番証跡には、請求書observerの実行結果が残っています。最新のstdoutは次の値です。

```json
{"observed_at":"2026-07-28T09:55:05.299Z","deliveries_seen":4,"pending":4,"invoiced":0,"invoice_created":0,"paid":0,"revenue_recorded":0,"revenue_duplicates":0,"rejected":0,"invoices":[]}
```

このログから言えるのは、4件を観測し、4件が `pending` のまま、請求書作成も支払いも収益計上も0件だったことです。これは失敗を隠さない、実際のobserver runの結果です。`invoice_created=0` を請求書発行済みとは呼びません。

同じ証跡にあるコードの識別子も残します。

| 記録 | 実測値 |
| --- | --- |
| delivery commit | `a0424042815523f438f85c333938af691a9741f8` |
| observerのmerge commit | `f4b52f75d92c91ccffb92316953a6c0b48b7f129` |
| settlementのmerge | `a9edfe883e9a367f5e595087f393f3f4c44047aa` |
| production run | `runs=16`, 最終exit code `0` |
| acceptance | `pending`; invoice `0`; revenue `0` |

ただし、このobserverの公開ログは、独立した `run_id`、`score_before`、`score_after`、`startup_hash` を持っていません。`observed_at` は識別用の値として使えますが、評価scoreや起動時hashの代用品ではありません。無い値をcommit hashや時刻から復元してはいけません。

この不足が、単発テストと閉じた改善ループの差です。請求書の状態が変わらなかったことは観測できても、評価付き更新を次回起動が消費したことまでは、このログだけでは証明できません。

## 5つの証拠を確認する方法

### 1. 仕事と版に名前を付ける

`run_id`、`task_id`、入力hash、起動時に読んだ版のhashを固定します。ファイル名の `improved` は上書きできますが、hashは採点時の版を指し続けます。

### 2. 採点器とデータ分割を先に凍結する

評価器の版と、編集を考えるデータ・採用を判断するheld-outまたはselection splitを記録します。候補を見た後で評価器を選ぶと、scoreの比較は再現できません。以下は採用前に固定する手順であり、このrunで過学習を測ったという主張ではありません。

### 3. 前後と候補を残す

候補を採用した場合も、捨てた場合も、次の1行を保存します。

```text
run_id | task_id | evaluator_version | selection_split | base_hash | candidate_hash | score_before | score_after | decision
```

`score_after` が高い、だけでは不十分です。どの版からどの候補を作り、どの評価器とsplitで比較したかが必要です。

### 4. gateで採用を決める

MicrosoftのSkillOptは、rollout、reflect、aggregate、select、update、gateという順でスキル文書を更新します。公式ガイドでは、gate有効時はselection splitの設定scoreが現行版より厳密に高い候補だけを採用します。gateを外すと、候補を強制採用できます。安全性の差はここにあります。

### 5. 次の起動がhashを消費したことを読む

採用版のhashを、次のプロセスが実際に読んだhashと比較します。

```text
accepted_hash == startup_hash
```

この比較と、次の `run_id` に同じ `task_id` の結果があることまで確認して、初めて閉じたループです。このリポジトリのstartup consumerは、消費したbytesが採用版と違えば停止します。

```python
if consumed_hash != learning.get("after_hash"):
    raise ValueError("consumed weight hash differs from promoted hash")
```

controllerが保存する `before_hash`、`candidate_hash`、`after_hash` とactive版の照合は、実装にある契約です。上のinvoice observerに無い値を後付けしたものではありません。

## SkillOptの「+24.8 points」をどこまで読めるか

公式READMEのheadlineは、GPT-5.5をCodexのagentic loopで動かしたとき、no-skill accuracyの平均が **+24.8 points** 上がった、というものです。比較の範囲は6ベンチマーク、7 target model、3 execution harnessで、評価された52セルすべてでbestまたはtie-bestだった、と説明されています。

6つの研究ベンチマークは、DocVQA（文書QA）、ALFWorld（身体性タスク）、OfficeQA（企業QA）、SearchQA（オープンドメインQA）、LiveMathematicianBench（数学推論）、SpreadsheetBench（表計算編集）です。

ここで止めるべきです。読んだ公式READMEとdocsは、`+24.8` についてセル別のサンプル数、本番請求書タスクでの結果、評価器の分散を示していません。したがって、この数字を1つの本番タスクの改善率とは呼びません。

| `+24.8`について | 公式資料で確認できること |
| --- | --- |
| taskとbaseline | 6ベンチマーク、no-skill accuracyとの比較 |
| 条件 | GPT-5.5、Codex agentic loop、7 target model、3 harness、52セル |
| 評価の単位 | READMEは平均accuracy liftと説明するが、セル別のn・評価器・分散は未開示 |
| 実運用との区別 | 本番請求書runの結果ではない。SleepのSearchQA/SpreadsheetBench条件も別実験 |

つまり、この表にある範囲が「根拠」で、未開示欄は「分からないこと」です。数字を本番改善率に変換する証拠はありません。

別のSkillOpt-Sleep実験には、5 nights × 毎晩10件の新しい実タスク、GPT-5.5、seed 42という条件があります。SearchQAはheld-out 1,400件をSQuAD exact matchで、SpreadsheetBenchは280件を生成コードの実行とgolden workbookのセル単位比較で評価しています。強いモデルのnear-ceiling条件では単一seedの分散が±1–2 pointsで、約1.5 points未満の差はnoiseとして扱う、とも書かれています。これは有用な別実験の条件ですが、`+24.8` のセル別サンプル数ではありません。

## さいごに

