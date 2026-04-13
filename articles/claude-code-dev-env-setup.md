---
title: "Claude Codeで開発環境を一瞬でセットアップする実践ガイド"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "ai", "開発効率化", "環境構築", "docker"]
published: true
---

新しいPCを買ったとき、新しいプロジェクトに参加したとき、開発環境のセットアップに半日かかった経験はありませんか？ Claude Code なら**数分で完了**します。

## 新規プロジェクトのスキャフォールド

ゼロからプロジェクトを立ち上げるのも Claude Code に任せられます。

```bash
mkdir my-app && cd my-app
claude -p "
Next.js 15 + TypeScript + Tailwind CSS + Prisma + PostgreSQL の
プロジェクトをセットアップしてください。

含めてほしいもの:
- ESLint + Prettier 設定
- Vitest 設定
- Docker Compose（PostgreSQL）
- .env.example
- GitHub Actions CI
- CLAUDE.md

npm install まで実行してください。
"
```

5分でベストプラクティスに沿ったプロジェクトが出来上がります。

## 既存プロジェクトの環境構築

チームのリポジトリを clone した後の環境構築を自動化します。

```bash
git clone https://github.com/team/project.git
cd project
claude -p "
このプロジェクトの開発環境をセットアップしてください。
README.md と package.json を読んで必要な手順を実行してください。

- 依存パッケージのインストール
- 環境変数の設定（.env.example → .env）
- データベースのセットアップ
- マイグレーション実行
- 動作確認（dev サーバー起動）

エラーが出たら自動で解決してください。
"
```

README を読み、package.json を解析し、必要なコマンドを順番に実行してくれます。途中でエラーが出ても自動で対処します。

## Docker 環境の構築

Docker Compose の設定も一発です。

```bash
claude -p "
以下のサービスを含む docker-compose.yml を作成してください:
- PostgreSQL 16（ポート5432）
- Redis 7（ポート6379）
- MinIO（S3互換、ポート9000/9001）

各サービスのヘルスチェック、ボリューム永続化、
.env からの環境変数読み込みも含めてください。
docker compose up -d まで実行してください。
"
```

## エディタ設定の統一

チーム全員のエディタ設定を揃えるのも簡単です。

```bash
claude -p "
VS Code の設定ファイルを作成してください。

.vscode/settings.json:
- 保存時にPrettier自動フォーマット
- TypeScript の strict モード
- import の自動ソート

.vscode/extensions.json:
- チームに推奨する拡張機能リスト

.editorconfig:
- インデントはスペース2つ
- 末尾の空白を削除
"
```

これを Git にコミットすれば、チーム全員が同じ設定で作業できます。

## 環境変数の漏れを防ぐ

新しい環境変数を追加したときに `.env.example` の更新を忘れる問題、ありますよね。

```bash
claude -p "
プロジェクト内のすべてのファイルをスキャンして、
process.env.XXX で参照している環境変数を全てリストアップ。

.env.example にダミー値付きで出力してください。
既存の .env.example があれば差分だけ追加してください。
"
```

これで環境変数のドリフトがゼロになります。

## CI/CD パイプラインも自動生成

```bash
claude -p "
このプロジェクト用の GitHub Actions ワークフローを作成:

push時: lint → typecheck → test → build
PR時: 上記 + カバレッジレポート
main マージ時: 上記 + Cloudflare Pages デプロイ
"
```

数秒で `.github/workflows/ci.yml` が出来上がります。

## CLAUDE.md に手順を残す

セットアップが完了したら、その手順を CLAUDE.md に残しておきます。

```bash
claude -p "
このプロジェクトの開発環境セットアップ手順を
CLAUDE.md の ## 環境構築 セクションに追記してください。

新しいメンバーが clone してから dev サーバーを起動するまでの
全手順をステップバイステップで書いてください。
"
```

次に新しいメンバーが参加したとき、Claude Code が CLAUDE.md を読んで自動でセットアップしてくれます。

## ハマりどころ

### Node.js のバージョン管理

プロジェクトごとに Node.js バージョンが違う場合、`.nvmrc` や `.node-version` を作っておくと Claude Code が自動で認識します。

### Docker の権限問題

Linux / WSL で Docker を使う場合、権限エラーが出ることがあります。Claude Code に「この Docker エラーを解決して」と言えば、`sudo` 追加やグループ設定を提案してくれます。

### M1/M2 Mac と Intel の差異

Apple Silicon と Intel でビルド挙動が違うことがあります。CLAUDE.md に「Apple Silicon の場合は〜」と書いておくと Claude Code がアーキテクチャを考慮してくれます。

## まとめ

- 新規プロジェクトのスキャフォールドが5分で完了
- 既存プロジェクトの環境構築を自動化（エラー対処含む）
- Docker Compose、エディタ設定、CI/CD も一発生成
- 環境変数の管理も漏れなく自動化
- CLAUDE.md に手順を残してチームで共有

開発環境のセットアップは「手作業」の時代から「Claude Code に任せる」時代へ。

---

※この記事は [claudecode-lab.com](https://claudecode-lab.com/blog/claude-code-dev-environment-setup/) の一部を Zenn 向けに再構成したものです。
