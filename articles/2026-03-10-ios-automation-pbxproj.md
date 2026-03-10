---
title: "How to iOS開発を自動化する（.pbxproj ファイルを壊さずに）"
emoji: "🏗️"
type: "tech"
topics: ["ios", "xcode", "automation", "ai"]
published: true
---

## TL;DR

AI エージェントを使った iOS 開発の自動化で、Xcode プロジェクトファイル（`.pbxproj`）を壊さないための実践的アプローチを紹介します。App Store Connect API の自動化、Xcode ビルドプロセスの分析、そして「AI に触らせてはいけないファイル」の明確化により、安全な自動化を実現します。

## 前提条件

- Xcode 15+
- OpenClaw または Claude Code などの AI 開発ツール
- App Store Connect アカウント
- 基本的な Xcode プロジェクト構成の理解

## 問題: .pbxproj の壊れやすさ

Xcode の `.pbxproj` ファイルは、プロジェクトの構成情報（ファイル、ターゲット、ビルド設定）を保持する重要なファイルです。しかし、このファイルは以下の特性があります：

- **XML ライクだが独自フォーマット**: 手動編集が非常に困難
- **行数が多い**: 数千行に及ぶことも珍しくない
- **マージコンフリクト頻発**: 複数人での開発では Git コンフリクトの温床
- **AI が苦手**: LLM が編集すると高確率で構文エラーを起こす

Source: [Kris Puckett: iOS Development Best Practices](https://www.krispuckett.com)
核心の引用：「.pbxproj をAIに触らせない」

## Step 1: 自動化範囲の明確化

以下の境界線を引きます：

| 自動化OK | 手動のみ |
|----------|----------|
| ソースコード（.swift）の生成・編集 | .pbxproj の直接編集 |
| テストコード（.swift）の生成 | ターゲットへのファイル追加 |
| ドキュメント生成 | 新規ターゲットの追加 |
| App Store Connect API 操作 | Build Settings の変更 |
| ビルドログの分析 | Info.plist の schema 変更 |

**鉄則**: AI にファイル追加・削除をさせない。ファイルは作るが、Xcode プロジェクトへの登録は人間が Xcode GUI で行う。

## Step 2: App Store Connect の自動化（asc CLI）

`.pbxproj` に触れることなく自動化できる領域として、App Store Connect API があります。

```bash
# asc CLI のインストール（Homebrew 経由）
brew install rudrankriyam/formulae/asc

# API キーの設定
export ASC_KEY_ID="your-key-id"
export ASC_ISSUER_ID="your-issuer-id"
export ASC_PRIVATE_KEY_PATH="/path/to/AuthKey_XXXXX.p8"

# アプリ情報の取得
asc apps list

# TestFlight ビルドの取得
asc builds list --app-id 12345678

# レビューステータスの確認
asc app-store-versions show --app-id 12345678
```

Source: [rudrankriyam/asc GitHub](https://github.com/rudrankriyam/asc)

**メリット**:
- Xcode プロジェクトに一切触れない
- レビュー状況・メトリクスの監視を完全自動化
- CI/CD パイプラインに統合可能

## Step 3: Xcode ビルドの分析（XcodeBuildMCP）

ビルドプロセスの最適化も、`.pbxproj` を編集せずに実現できます。

```bash
# XcodeBuildMCP を使ったビルド分析
# （MCP サーバー経由で Claude に接続）

# ビルド時間の分析
xcodebuild -showBuildSettings -project MyApp.xcodeproj

# 依存関係グラフの可視化
xcodebuild -project MyApp.xcodeproj -scheme MyApp -showBuildTimingSummary

# 警告・エラーの抽出
xcodebuild build | grep "warning:"
```

Source: [XcodeBuildMCP Documentation](https://github.com/blazickjp/XcodeBuildMCP)

**AI エージェントができること**:
- ビルドログの解析
- 警告の優先順位付け
- 依存関係の最適化提案
- ビルド時間のボトルネック特定

**AI エージェントができないこと**:
- Build Settings の自動変更
- Framework の自動追加・削除

## Step 4: ファイル生成の安全なワークフロー

```bash
# 1. AI にコードを生成させる
claude "Views/NewFeatureView.swift を生成してください"

# 2. 生成されたファイルを確認
cat Views/NewFeatureView.swift

# 3. Xcode GUI で手動追加
# File → Add Files to "MyApp"...
# ✅ Copy items if needed
# ✅ Add to targets: MyApp

# 4. Git でコミット
git add Views/NewFeatureView.swift
git add MyApp.xcodeproj/project.pbxproj  # Xcode が変更したもの
git commit -m "feat: Add NewFeatureView"
```

**重要**: `.pbxproj` の変更は Xcode GUI が行ったもののみをコミットする。AI が生成した `.pbxproj` 変更は即座に破棄すること。

## Step 5: factory-bp-efficiency パターン（継続的 BP 収集）

自動化の質を維持するために、定期的にベストプラクティスを収集・適用します。

```bash
# Cron で定期実行（毎日 22:20）
# ~/.openclaw/skills/factory-bp-efficiency/SKILL.md

# 1. iOS 自動化の最新 BP を検索
web_search "iOS development automation best practices 2026"
web_search "Xcode project file automation"

# 2. ClawHub からスキルをインストール
clawhub search "ios xcode"
clawhub install app-store-connect --dir ~/.openclaw/skills
clawhub install xcode-build-analyzer --dir ~/.openclaw/skills

# 3. BP を適用し、SKILL.md を更新
```

**効果**: 手動で BP を追いかける必要がなくなり、常に最新の安全な自動化パターンが適用されます。

## まとめ

| 教訓 | 詳細 |
|------|------|
| **AI の得意領域を見極める** | コード生成は OK、プロジェクト構成変更は NG |
| **境界線を明確にする** | `.pbxproj` は Xcode GUI のみが触る |
| **自動化の範囲を段階的に広げる** | App Store Connect API → ビルド分析 → コード生成の順 |
| **継続的な BP 収集** | factory-bp-efficiency パターンで最新手法を追従 |

Source: [The Twelve-Factor App](https://12factor.net/config)
核心の引用：「an application should have a single source of truth」

`.pbxproj` のような脆弱なファイルについては、AI ではなく公式ツール（Xcode GUI）を SSOT（Single Source of Truth）として扱うことが、安全な自動化の鍵です。
