---
title: "Claude Code チーム導入完全ガイド｜CLAUDE.md設計・権限共有・オンボーディング"
emoji: "👥"
type: "tech"
topics: ["claudecode", "チーム開発", "ai", "devops", "onboarding"]
published: true
---

## はじめに

Claude Code を個人で使い始めたエンジニアが次にぶつかる壁は「**チームで統一した使い方ができない**」問題です。Aさんは何でも許可、Bさんは権限絞りすぎ、Cさんは CLAUDE.md すら知らない——チームでバラバラに使っていると、生産性向上どころか混乱の元になります。

この記事では、**チームで Claude Code を標準化して導入するための設計・設定・オンボーディング手順**を一通り解説します。

## 設計思想: 「チーム共有」と「個人設定」を分ける

Claude Code の設定ファイルには2つの層があります。

| ファイル | 管理 | 用途 |
|---------|------|------|
| `./CLAUDE.md` | **git管理 (共有)** | プロジェクト規約、禁則事項、コンテキスト |
| `./.claude/settings.json` | **git管理 (共有)** | 権限設定、Hooks |
| `~/.claude.json` | **個人 (非共有)** | MCPサーバー、個人トークン |
| `~/.claude/memory/` | **個人 (非共有)** | 長期記憶、個人の好み |

**チームで共有するのは前者2つだけ**。個人の設定や認証情報はリポジトリに入れません。

## ステップ1: プロジェクト CLAUDE.md を設計する

CLAUDE.md はチームの「Claude Code への説明書」です。新メンバーが加わっても、この1ファイルを読めばプロジェクト固有のルールが全てわかるようにします。

### テンプレート

```markdown
# [プロジェクト名] Claude Code 規約

## プロジェクト概要
- 概要: (何を作っているか1行で)
- スタック: TypeScript / Next.js / PostgreSQL / Vercel
- リポジトリ構成: monorepo (apps/web, apps/api, packages/*)

## ディレクトリ構造
apps/web/          ← フロントエンド (Next.js App Router)
apps/api/          ← バックエンド (Hono + Drizzle)
packages/ui/       ← 共通UIコンポーネント
packages/types/    ← 共通型定義

## コーディング規約
- 言語: TypeScript strict モード (any 禁止)
- フォーマット: Biome (prettier の代替)
- 命名: コンポーネントは PascalCase、関数は camelCase、定数は UPPER_SNAKE
- テスト: Vitest、カバレッジ 80% 以上

## 禁則事項 (Claude Code が絶対にやってはいけないこと)
- .env ファイルを読んで内容を出力する
- git push --force を実行する
- 本番 DATABASE_URL に対して DROP/TRUNCATE を実行する
- node_modules/ を直接編集する
- パッケージバージョンをユーザー確認なしに上げる

## ブランチ戦略
- main: 本番、直接 push 禁止
- develop: 開発統合ブランチ
- feature/xxx: 機能開発 (develop からブランチ)
- fix/xxx: バグ修正

## PR 作成のルール
- タイトル: [feat/fix/chore]: 変更内容の要約
- 本文: 変更理由・テスト方法・スクリーンショット(UI変更時)
- PR を作ったら必ず gh pr create で URL を報告

## よく使うコマンド
npm run dev        # 開発サーバー起動
npm run test       # テスト実行
npm run typecheck  # 型チェック
npm run lint       # Biome lint
bash scripts/deploy.sh  # 本番デプロイ (要承認)
```

このファイルがある状態で `claude` を起動するだけで、チーム全員が同じルールの下で作業できます。

## ステップ2: 権限設定を共有する

`.claude/settings.json` をリポジトリに入れ、**全員が同じ制限で動く**ようにします。

```json
{
  "permissions": {
    "allow": [
      "Read(**)",
      "Glob(**)",
      "Grep(**)",
      "Bash(npm run*)",
      "Bash(git log*)",
      "Bash(git diff*)",
      "Bash(git status*)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(git push --force*)",
      "Bash(git push -f*)",
      "Bash(DROP TABLE*)",
      "Bash(TRUNCATE*)",
      "Bash(curl * | bash)",
      "Bash(wget * | sh)"
    ],
    "ask": [
      "Write(**)",
      "Edit(**)",
      "Bash(git commit*)",
      "Bash(git push*)",
      "Bash(npm publish*)",
      "Bash(*deploy*)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git add*)",
        "hooks": [{
          "type": "command",
          "command": "git diff --cached --name-only | grep -E '^\\.env' && echo '🚨 .env のステージを検出！中止します' && exit 1 || exit 0"
        }]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [{
          "type": "command",
          "command": "npx tsc --noEmit 2>/dev/null || true"
        }]
      }
    ]
  }
}
```

### 環境別に上書きできる仕組みも用意

```json
// .claude/settings.local.json (gitignore 済み、個人の追加設定用)
{
  "permissions": {
    "allow": [
      "Bash(psql*)"  // ← ローカルDBを叩きたい人だけ追加
    ]
  }
}
```

## ステップ3: 環境変数のチーム共有ルールを決める

```
# .env.example (git管理、値は空欄)
DATABASE_URL=
ANTHROPIC_API_KEY=
STRIPE_SECRET_KEY=
RESEND_API_KEY=

# .env (gitignore 済み、各自がコピーして値を埋める)
# → 値は 1Password / Bitwarden / AWS Secrets Manager 等で共有
```

**おすすめの共有方法**: 1Password の Shared Vault に `.env` の値を保存。新メンバーには Vault へのアクセスを付与するだけ。Claude Code がシークレットを見ることはない。

## ステップ4: チームオンボーディング手順

新メンバーが参加した際の手順書を CLAUDE.md と同じリポジトリに置いておくと、Claude Code 自身が説明してくれます。

```markdown
# ONBOARDING.md

## セットアップ手順 (所要 30 分)

### 1. 基本セットアップ
git clone <repo>
cd <repo>
cp .env.example .env
# → 1Password の [Project] Vault から値をコピー

### 2. 依存関係インストール
npm install

### 3. Claude Code セットアップ
# Claude Code をインストール済みであること
# ~/.claude.json に個人の ANTHROPIC_API_KEY を設定

### 4. 動作確認
npm run dev   # 起動確認
npm run test  # テスト全通過確認
claude        # Claude Code 起動、CLAUDE.md が読み込まれることを確認

### 5. 最初のタスク
claude -p "このプロジェクトの構成を説明して。その後 apps/web/README.md を作成して"
```

このドキュメントがあれば、**Claude Code 自身に「セットアップを手伝って」と頼める**ようになります。

## ステップ5: チームでの使い方ルールを決める

### おすすめのルール例

```markdown
# チーム Claude Code 利用規約

## やって良いこと
- 機能実装・リファクタリング・テスト生成
- ドキュメント生成・PR 説明文の自動作成
- デバッグ・エラー調査

## やってはいけないこと
- 本番環境への直接デプロイ (必ず CI/CD 経由)
- 他のメンバーのブランチへの push
- コードレビューを省略するための「Claude に書かせた = OK」という判断

## AI 生成コードのレビュー基準
- 「Claude が書いた」は免罪符にならない
- 本人がコードを理解し、説明できること
- テストが通ることは最低条件、意図が正しいかを確認

## 利用ログ (任意)
weekly-ai-log.md に「今週 Claude Code を使って何を達成したか」を記録。
チームの学習を共有する文化を作る。
```

## よくある落とし穴

**「CLAUDE.md が長くなりすぎる」問題**

CLAUDE.md が 1000行を超えると、毎回コンテキストをフルロードするコストが増えます。**200〜400行を目安**に、詳細は別ファイル (`ARCHITECTURE.md`, `TESTING.md` など) に切り出して CLAUDE.md からリンクしましょう。

**「メンバーごとに違う Claude Code バージョン」問題**

```json
// package.json に開発ツールとして記載
{
  "devDependencies": {
    "@anthropic-ai/claude-code": "latest"
  },
  "scripts": {
    "claude": "claude"
  }
}
```

`npx claude` を使えばバージョン統一も容易です。

**「新人が過剰に信頼してコードをそのまま使う」問題**

CLAUDE.md に明示: 「Claude が生成したコードは必ず自分でレビューし、意図を理解してからコミットすること」。チームの文化として「AI生成 = 素案」という認識を根付かせましょう。

## まとめ: チーム導入のロードマップ

```
Week 1: 基盤設定
  - CLAUDE.md 初版作成 (リードエンジニアが書く)
  - .claude/settings.json で権限を設定
  - .gitignore と .env.example を整備

Week 2: 試用期間
  - 全員が使い始める
  - CLAUDE.md の抜け・漏れを補完
  - settings.json の allow/deny を実態に合わせて調整

Week 3-: 定着・改善
  - 週次で「困ったこと・うまくいったこと」を共有
  - CLAUDE.md をリビングドキュメントとして更新し続ける
  - 新メンバーのオンボーディングで実際に使ってみる
```

CLAUDE.md は「書いて終わり」ではなく、**プロジェクトとともに育てる生きたドキュメント**です。チームで使い続けることで、Claude Code はどんどんプロジェクト固有の知識を蓄積し、より的確な支援ができるようになります。

## 参考資料

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/claude/docs/claude-code)
- [CLAUDE.md ベストプラクティス](https://zenn.dev/masa_claudecodelab/articles/claude-md-best-practices)
- [Claude Code セキュリティ対策完全ガイド](https://claudecode-lab.com/blog/claude-code-security-best-practices/)
