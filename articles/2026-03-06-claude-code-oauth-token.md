---
title: "Claude Code OAuth トークン期限切れを30秒で復旧させる方法"
emoji: "🔑"
type: "tech"
topics: ["claudecode", "anthropic", "oauth", "macos", "devops"]
published: true
---

## TL;DR
Claude CodeをSSH経由で使うとOAuthトークンが期限切れでエラーになる問題を解説します。macOS Keychainから最新トークンを取得し環境変数に注入すれば即座に解決します。30秒で復旧でき、tmuxセッションも壊れません。

## 前提条件
- macOS（Keychainが使える環境）
- Claude Code CLI v2.1.45以降がインストール済み
- SSH経由でMacにアクセスしている、またはtmuxセッションを使っている
- Claude Codeで過去に `claude setup-token` を実行済み（OAuthトークンを生成済み）

## 問題: SSH 経由だと OAuth トークンが読めない

Claude CodeはOAuthトークンをmacOS Keychainに保存します（サービス名： `Claude Code-credentials`）。しかし、SSH経由で実行するとKeychainがロックされており、トークンにアクセスできません。

```bash
$ echo "test" | claude -p
Error: Authentication required. Run 'claude auth login'
```

`claude auth login` を実行するとブラウザが開くため、SSH経由では実行不可能です。

## 解決策: 環境変数で直接トークンを渡す

Claude Codeは `CLAUDE_CODE_OAUTH_TOKEN` 環境変数があればそれを優先して使います。Keychainからトークンを取得して環境変数に設定すれば、SSH経由でも動作します。

## Step 1: Keychain からトークンを取得

Macのローカルターミナル（SSHではない）で以下を実行します：

```bash
security find-generic-password -s 'Claude Code-credentials' -w
```

出力例（JSON形式）:
```json
{"accessToken":"sk-ant-oat01-qCV5O13G...bcGbVQAA","refreshToken":"...","expiresAt":"2027-02-18T07:00:00.000Z"}
```

このJSON全体をコピーします。

## Step 2: 環境変数に設定

SSH経由でMacに接続している場合、以下のコマンドで環境変数に設定します：

```bash
export CLAUDE_CODE_OAUTH_TOKEN='{"accessToken":"sk-ant-oat01-qCV5O13G...bcGbVQAA","refreshToken":"...","expiresAt":"2027-02-18T07:00:00.000Z"}'
```

## Step 3: tmux セッションに直接注入（tmux 使用時のみ）

tmuxセッションでClaude Codeを実行している場合、tmuxの環境変数を更新します：

```bash
tmux set-environment -t mobileapp-factory CLAUDE_CODE_OAUTH_TOKEN '{"accessToken":"sk-ant-oat01-qCV5O13G...bcGbVQAA","refreshToken":"...","expiresAt":"2027-02-18T07:00:00.000Z"}'
```

tmuxセッション内で環境変数を読み込みます：

```bash
export CLAUDE_CODE_OAUTH_TOKEN=$(tmux show-environment -t mobileapp-factory CLAUDE_CODE_OAUTH_TOKEN | cut -d= -f2-)
```

## Step 4: 動作確認

```bash
echo "test prompt" | claude -p --allowedTools Bash,Read,Write
```

正常に動作すれば成功です。

## 永続化: `.zshrc` と `.openclaw/.env` に保存

毎回設定するのは面倒なので、以下のファイルに保存します：

### Mac Mini の場合
```bash
# ~/.zshrc に追加
export CLAUDE_CODE_OAUTH_TOKEN='{"accessToken":"sk-ant-oat01-qCV5O13G...bcGbVQAA","refreshToken":"...","expiresAt":"2027-02-18T07:00:00.000Z"}'
```

### OpenClaw Gateway の場合
```bash
# ~/.openclaw/.env に追加
CLAUDE_CODE_OAUTH_TOKEN='{"accessToken":"sk-ant-oat01-qCV5O13G...bcGbVQAA","refreshToken":"...","expiresAt":"2027-02-18T07:00:00.000Z"}'
```

OpenClaw Gatewayは起動時に `.openclaw/.env` を自動で読み込みます。

## トラブルシューティング

| 症状 | 原因 | 対処 |
|------|------|------|
| `security: SecKeychainSearchCopyNext: The specified item could not be found in the keychain.` | `claude setup-token` を実行していない | ローカルターミナルで `claude setup-token` を実行し、ブラウザで認証 |
| `Error: Authentication required` | 環境変数が設定されていない | Step 2 を再実行 |
| tmux セッションで環境変数が反映されない | tmux の環境変数が更新されていない | Step 3 を実行 |
| トークンの有効期限が切れた | `expiresAt` が過去の日付 | Step 1 から再実行（Keychain には最新トークンが保存されている） |

## まとめ

| 教訓 | 詳細 |
|------|------|
| **Keychain は SSH からアクセスできない** | Claude Code のトークンは macOS Keychain に保存されるため、SSH 経由では読めない |
| **環境変数で解決** | `CLAUDE_CODE_OAUTH_TOKEN` にトークンを設定すれば SSH / tmux でも動作 |
| **永続化が重要** | `.zshrc` や `.openclaw/.env` に保存すれば毎回設定不要 |
| **復旧時間30秒** | Keychain から取得 → 環境変数設定 → tmux 注入で即座に復旧 |
| **tmux セッションを壊さない** | 環境変数注入だけで復旧するため、実行中のプロセスを停止する必要なし |

この手順により、mobileapp-factory-daily cronがClaude Code OAuthトークン期限切れで失敗した際も、30秒で復旧し、Ralph.shの実行を再開できました。
