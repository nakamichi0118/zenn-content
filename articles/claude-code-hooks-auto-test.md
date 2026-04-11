---
title: "Claude Code Hooksで自動テスト環境を構築する完全ガイド"
emoji: "🧪"
type: "tech"
topics: ["claudecode", "ai", "テスト自動化", "開発効率化"]
published: true
---

Claude Codeの**Hooks機能**を使うと、コード編集の前後にスクリプトを自動実行できます。本記事では、Hooksを使ってファイル保存時に自動でテストを走らせ、コード品質を保ちながら開発速度を上げる方法を解説します。

## Claude Code Hooksとは

Hooksは、Claude Codeが特定のアクションを実行する前後に、任意のコマンドを実行できる仕組みです。主な種類は以下の4つです。

- **PreToolUse**: ツール実行前
- **PostToolUse**: ツール実行後
- **Notification**: 通知時
- **Stop**: セッション終了時

## 自動テスト環境を構築する

### 1. 基本設定

`.claude/settings.json` に以下を追加します。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npm test -- --findRelatedTests $CLAUDE_FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

これで、Claude Codeがファイルを編集または作成するたびに、関連するテストだけが自動実行されます。

### 2. TypeScriptプロジェクトの例

Vitestを使う場合の設定例です。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npx vitest related $CLAUDE_FILE_PATH --run"
          }
        ]
      }
    ]
  }
}
```

`vitest related` は変更ファイルに関連するテストだけを実行するので、フィードバックが速いです。

### 3. Pythonプロジェクトの例

pytestと組み合わせる場合：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "pytest --testmon $CLAUDE_FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

`pytest-testmon` プラグインを使うと、変更があったコードに影響するテストだけを賢く選んで実行してくれます。

## テスト失敗時の処理

テストが失敗したとき、Claude Codeに自動で修正させるパターンも便利です。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npm test || echo 'TESTS_FAILED' >&2"
          }
        ]
      }
    ]
  }
}
```

`>&2` でstderrに出力すると、Claude Code側で失敗を検知して、次のターンで修正アクションを促せます。

## 複数フックの組み合わせ

リント・型チェック・テストを一気に走らせる例：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write $CLAUDE_FILE_PATH && npx eslint --fix $CLAUDE_FILE_PATH && npx tsc --noEmit && npm test"
          }
        ]
      }
    ]
  }
}
```

順番に実行されて、どこかで失敗したら止まります。コミット前のpre-commitフックを書く感覚に近いです。

## カバレッジレポートの自動生成

セッション終了時にカバレッジレポートを生成する例：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "npm test -- --coverage && open coverage/index.html"
          }
        ]
      }
    ]
  }
}
```

作業を終えたら自動でカバレッジを確認できるので、品質メトリクスを常に意識した開発ができます。

## ハマりどころと対策

### 1. テスト実行が遅すぎる場合

毎回フルテストを走らせると遅いので、必ず**変更ファイルに関連するテストだけ**を実行する設定にしましょう。`vitest related`、`jest --findRelatedTests`、`pytest --testmon` などが活躍します。

### 2. 環境変数が見つからない

フックスクリプトはClaude Codeのプロセスから起動されるので、シェルプロファイル（`.zshrc` 等）が読み込まれません。必要なPATHや環境変数は `.claude/settings.json` の `env` セクションで明示しましょう。

### 3. Windowsとの互換性

`open` や `&&` の挙動はWindows/macOS/Linuxで異なります。チームで共有する場合は `npm run` スクリプト経由にして、`package.json` 側でクロスプラットフォーム対応するのがおすすめです。

## まとめ

- Claude Code Hooksを使えば、ファイル編集のたびに自動でテストが走る環境が作れる
- 関連テストだけを実行することで、フィードバックループを最小化できる
- リント・型チェック・テストを組み合わせれば、コミット前と同等の品質チェックを開発中に回せる
- 失敗時はstderrに出力して、Claude Codeに修正を促せる

Hooksを使いこなすと、テストカバレッジを保ちながら開発速度を落とさない、理想的なワークフローが実現できます。

---

※この記事は [claudecode-lab.com](https://claudecode-lab.com/blog/claude-code-hooks-guide/) の一部を Zenn 向けに再構成したものです。より詳しい設定例やトラブルシューティングは本サイトをご覧ください。
