---
title: "Claude Code × セキュリティ自動チェック完全ガイド｜settings.jsonとHooksで守る開発環境"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "security", "hooks", "devops", "githubactions"]
published: true
---

## はじめに

先日、私が書いた Claude Code のセキュリティ記事が想定外に拡散されました。コメントを見ていると「わかるわかる、自分もやらかした」という声が多く、Claude Code のセキュリティリスクは思った以上に身近な問題だと実感しました。

この記事では、**settings.json と Hooks を組み合わせてセキュリティチェックを自動化する実装**を紹介します。「人間が気をつける」ではなく「仕組みが守る」構造を作ります。

## セキュリティ自動化の基本設計

Claude Code のセキュリティ自動化は3層で考えます。

```
Layer 1: permissions — 危険な操作をそもそも実行させない
Layer 2: Hooks      — 実行前後に自動チェックを挟む
Layer 3: CLAUDE.md  — ルールをモデルに教え込む
```

3層すべてを組み合わせることで、漏れのない防御ができます。

## Layer 1: permissions 設定

まず「絶対にやってはいけない操作」を `deny` に入れます。

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Read(**)",
      "Glob(**)",
      "Grep(**)",
      "Bash(npm run *)",
      "Bash(git log*)",
      "Bash(git diff*)",
      "Bash(git status*)"
    ],
    "deny": [
      "Bash(rm -rf ~*)",
      "Bash(rm -rf /*)",
      "Bash(git push --force *main*)",
      "Bash(git push --force *master*)",
      "Bash(git push -f *main*)",
      "Bash(git push -f *master*)",
      "Bash(DROP TABLE*)",
      "Bash(TRUNCATE *)",
      "Bash(curl * | bash)",
      "Bash(wget * | sh)",
      "Bash(chmod 777*)",
      "Bash(sudo rm*)"
    ],
    "ask": [
      "Write(**)",
      "Edit(**)",
      "Bash(git commit*)",
      "Bash(git push*)",
      "Bash(git add*)",
      "Bash(rm *)",
      "Bash(*deploy*)",
      "Bash(*migrate*)",
      "Bash(npm install*)"
    ]
  }
}
```

`deny` は絶対禁止、`ask` は毎回承認が必要、`allow` は自動実行です。

## Layer 2: Hooks による自動チェック

Hooks が本体です。実行前後に自動でセキュリティチェックを挟みます。

### シークレットスキャン Hook

`git add` 実行前に `.env` 系ファイルを自動ブロック。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git add*)",
        "hooks": [
          {
            "type": "command",
            "command": "staged=$(git diff --cached --name-only 2>/dev/null); if echo \"$staged\" | grep -qE '\\.(env|pem|key|p12|pfx)$|credentials|secrets'; then echo '🚨 SECRET FILE DETECTED: ' $staged && exit 1; fi"
          }
        ]
      }
    ]
  }
}
```

### コミット前のフルスキャン Hook

コミット実行前にステージング済み差分全体をスキャン。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git commit*)",
        "hooks": [
          {
            "type": "command",
            "command": "node scripts/secret-scan.mjs"
          }
        ]
      }
    ]
  }
}
```

```javascript
// scripts/secret-scan.mjs
import { execSync } from "child_process";

const diff = execSync("git diff --cached", { encoding: "utf-8" });

const PATTERNS = [
  { name: "Anthropic APIキー", re: /sk-ant-api\d+-[A-Za-z0-9_-]{20,}/ },
  { name: "OpenAI APIキー",    re: /sk-[A-Za-z0-9]{48}/ },
  { name: "AWS アクセスキー",  re: /AKIA[0-9A-Z]{16}/ },
  { name: "Slack トークン",    re: /xox[baprs]-[0-9A-Za-z-]{10,}/ },
  { name: "GitHub トークン",   re: /ghp_[A-Za-z0-9]{36}/ },
  { name: "GCP APIキー",       re: /AIza[0-9A-Za-z_-]{35}/ },
  { name: "Generic Secret",    re: /[Ss]ecret[_-]?[Kk]ey\s*[:=]\s*['"][^'"]{8,}['"]/ },
];

const found = PATTERNS.filter(({ re }) => re.test(diff));

if (found.length > 0) {
  console.error("\n🚨 シークレット検出！コミットを中止します:");
  found.forEach(({ name }) => console.error(`  ❌ ${name}`));
  console.error("\n対処: git reset HEAD <file> で unstage してください");
  process.exit(1);
}

console.log("✅ シークレットスキャン: 問題なし");
process.exit(0);
```

### 削除コマンドへの猶予 Hook

`rm` 系コマンドに5秒の猶予を設けます。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(rm*)",
        "hooks": [
          {
            "type": "command",
            "command": "echo '⚠️  削除コマンドを検出しました' && echo 'Ctrl+C で中止できます。5秒後に実行...' && sleep 5"
          }
        ]
      }
    ]
  }
}
```

### ファイル編集後の型チェック Hook

TypeScript ファイルを編集したら自動で型チェック。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit(*.ts)|Write(*.ts)",
        "hooks": [
          {
            "type": "command",
            "command": "npx tsc --noEmit 2>&1 | head -20 || true"
          }
        ]
      }
    ]
  }
}
```

## settings.json の完全版

上記を全部まとめた完成形です。コピーして使えます。

```json
{
  "permissions": {
    "allow": [
      "Read(**)", "Glob(**)", "Grep(**)",
      "Bash(npm run lint)", "Bash(npm run test)", "Bash(npm run typecheck)",
      "Bash(git log*)", "Bash(git diff*)", "Bash(git status*)", "Bash(git branch*)"
    ],
    "deny": [
      "Bash(rm -rf ~*)", "Bash(rm -rf /*)",
      "Bash(git push --force *main*)", "Bash(git push --force *master*)",
      "Bash(git push -f *main*)", "Bash(git push -f *master*)",
      "Bash(DROP TABLE*)", "Bash(TRUNCATE *)",
      "Bash(curl * | bash)", "Bash(wget * | sh)"
    ],
    "ask": [
      "Write(**)", "Edit(**)",
      "Bash(git commit*)", "Bash(git push*)", "Bash(git add*)",
      "Bash(rm *)", "Bash(*deploy*)", "Bash(*migrate*)", "Bash(npm install*)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git add*)",
        "hooks": [{
          "type": "command",
          "command": "staged=$(git diff --cached --name-only 2>/dev/null); if echo \"$staged\" | grep -qE '\\.(env|pem|key)$|credentials'; then echo '🚨 危険なファイル検出: ' $staged && exit 1; fi"
        }]
      },
      {
        "matcher": "Bash(git commit*)",
        "hooks": [{ "type": "command", "command": "node scripts/secret-scan.mjs" }]
      },
      {
        "matcher": "Bash(rm*)",
        "hooks": [{ "type": "command", "command": "echo '⚠️ 削除コマンド。5秒後に実行。Ctrl+C で中止。' && sleep 5" }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit(*.ts)|Write(*.ts)",
        "hooks": [{ "type": "command", "command": "npx tsc --noEmit 2>&1 | head -10 || true" }]
      }
    ]
  }
}
```

## Layer 3: CLAUDE.md でルールを明文化

設定だけでなく、モデルへの指示も必要です。

```markdown
## セキュリティ必須ルール

### 絶対禁止
- .env / *.pem / *credentials* ファイルを git add しない
- APIキー・パスワード・トークンをプロンプトに書かない
- git push --force を main/master ブランチに実行しない
- 本番 DATABASE_URL に対して DROP/TRUNCATE を実行しない

### 必須確認
- デプロイ前にステージング環境でテスト
- マイグレーション前に pg_dump でバックアップ
- 新しい API エンドポイントには認証チェックを必ず実装

### 環境変数の扱い
- 値は .env から process.env で読む
- .env は .gitignore で除外済みであることを確認
- .env.example に変数名だけ記載 (値は書かない)
```

## よくある落とし穴

**「Hook が実行されない」**

`settings.json` の JSON 構文エラーが最も多い原因。VS Code の JSON バリデーションで確認してください。

**「exit 1 したのにブロックされない」**

`command` が `sh -c` 経由で実行されるため、パイプや複合コマンドの終了コードが伝播しない場合があります。明示的に `exit 1` を書いてください。

```bash
# ❌ パイプの終了コードが伝播しない
"command": "cat .env | grep SECRET && exit 1"

# ✅ 明示的に制御
"command": "if cat .env | grep -q SECRET; then exit 1; fi"
```

**「毎回 sleep 5 が邪魔」**

本当に危険なコマンドのみ対象にするよう `matcher` を絞りましょう。全 `rm` に5秒はやりすぎです。

```json
// ❌ 全 rm に5秒
"matcher": "Bash(rm*)"

// ✅ rm -rf のみ5秒
"matcher": "Bash(rm -rf*)"
```

## まとめ

| Layer | 方法 | 効果 |
|-------|------|------|
| permissions | deny/ask/allow | 危険な操作を構造的にブロック |
| Hooks | PreToolUse/PostToolUse | 実行前後に自動チェック |
| CLAUDE.md | ルール明文化 | モデルの判断を正しい方向に誘導 |

「Claude Code が怖い」という感覚は正しいです。でも、**怖さを知ったうえで適切な安全装置を入れる**のが正解です。この3層構造を入れれば、本番障害の大半は防げます。

---

今回紹介した settings.json と scripts/secret-scan.mjs は [claudecode-lab.com](https://claudecode-lab.com/blog/claude-code-security-best-practices/) のセキュリティ記事でも詳しく解説しています。
