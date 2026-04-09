---
title: "CLAUDE.mdの書き方完全ガイド：プロジェクト設定のベストプラクティス"
emoji: "📝"
type: "tech"
topics: ["claudecode", "ai", "設定", "ベストプラクティス", "開発効率化"]
published: true
---

## CLAUDE.md とは

CLAUDE.mdは、Claude Codeがプロジェクトを理解するためのコンテキストファイルです。プロジェクトのルートに配置すると、Claude Codeがセッション開始時に自動的に読み込みます。適切に書かれたCLAUDE.mdは、Claude Codeの応答品質を劇的に向上させます。

## CLAUDE.md の配置場所と優先順位

CLAUDE.mdは複数の場所に配置でき、すべて読み込まれます。

```
~/.claude/CLAUDE.md          # グローバル設定（全プロジェクト共通）
./CLAUDE.md                   # プロジェクトルート（チーム共有）
./CLAUDE.local.md             # ローカル設定（.gitignore推奨）
./src/CLAUDE.md               # サブディレクトリ単位の設定
```

- **グローバル**：自分の好みのコーディングスタイルなど
- **プロジェクトルート**：チーム共有のルールや技術スタック
- **ローカル**：個人的な設定（Gitにコミットしない）
- **サブディレクトリ**：特定のモジュールに関する追加情報

## 基本テンプレート

以下は実践的なCLAUDE.mdのテンプレートです。

```markdown
# プロジェクト概要

ECサイトのバックエンドAPI。注文管理・在庫管理・ユーザー認証を提供。

## 技術スタック

- 言語: TypeScript 5.x
- ランタイム: Node.js 22
- フレームワーク: Fastify
- DB: PostgreSQL 16 + Prisma ORM
- テスト: Vitest
- CI: GitHub Actions

## ディレクトリ構造

src/
  routes/       # APIエンドポイント定義
  services/     # ビジネスロジック
  repositories/ # データアクセス層
  middleware/   # 認証・ログ・エラーハンドリング
  utils/        # ユーティリティ関数
  types/        # 型定義

## コーディング規約

- 関数はアロー関数で定義する
- エラーハンドリングは必ず try-catch で行い、カスタムエラークラスを使用
- 命名規則: camelCase（変数・関数）、PascalCase（型・クラス）
- ファイル名: kebab-case
- インポートは相対パスではなく、パスエイリアス `@/` を使用

## よく使うコマンド

- テスト実行: `npm test`
- 型チェック: `npx tsc --noEmit`
- リント: `npm run lint`
- DB マイグレーション: `npx prisma migrate dev`
- 開発サーバー: `npm run dev`

## 重要なルール

- prisma/schema.prisma を変更したら必ずマイグレーションを作成すること
- APIエンドポイントには必ず Zod バリデーションスキーマを定義すること
- 新しいルートを追加したら routes/index.ts に登録すること
```

## 効果的な書き方のポイント

### 1. 簡潔に書く

CLAUDE.mdはコンテキストウィンドウを消費します。冗長な説明は避け、要点を箇条書きにしましょう。

```markdown
# 悪い例
このプロジェクトはTypeScriptを使用しています。TypeScriptは
Microsoftが開発した言語で、JavaScriptに型を追加したもので...

# 良い例
- 言語: TypeScript 5.x（strict mode）
```

### 2. 「やってほしくないこと」を明記する

Claude Codeに避けてほしいパターンを書いておくと効果的です。

```markdown
## 禁止事項

- any 型の使用禁止
- default export 禁止（named export のみ）
- console.log でのデバッグ禁止（logger を使用）
- 既存テストの削除・スキップ禁止
```

### 3. ワークフローを定義する

タスクの進め方をガイドします。

```markdown
## 新機能追加の手順

1. `src/types/` に型定義を作成
2. `src/repositories/` にデータアクセス層を実装
3. `src/services/` にビジネスロジックを実装
4. `src/routes/` にエンドポイントを定義
5. 各層のユニットテストを作成
6. `npm test` で全テスト通過を確認
```

### 4. プロジェクト固有の知識を記述する

ドキュメント化されていないけど重要な情報を書きます。

```markdown
## プロジェクト固有の注意点

- `user_id` は UUID v7 を使用（時系列ソート可能にするため）
- 価格計算は必ず `Decimal.js` を使用（浮動小数点誤差防止）
- 環境変数は `src/config.ts` で一元管理、直接 process.env を参照しない
```

## チーム開発での運用

### .gitignore の設定

個人設定はコミットしないようにしましょう。

```gitignore
# .gitignore
CLAUDE.local.md
```

### レビュー対象に含める

CLAUDE.mdはプロジェクトの重要なドキュメントです。PRレビューの対象に含め、チームで内容を最新に保ちましょう。

## CLAUDE.md の更新タイミング

- 新しいライブラリを導入したとき
- コーディング規約を変更したとき
- ディレクトリ構造を変更したとき
- 頻繁にClaude Codeが間違えるパターンを発見したとき

## まとめ

CLAUDE.mdはClaude Codeを「自分のプロジェクトの専門家」にするための設定ファイルです。最初は `/init` で雛形を生成し、使いながら育てていくのが効果的です。チームで共有すれば、メンバー全員がClaude Codeの恩恵を最大限に受けられます。

設定の詳細は[Anthropic公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)も参照してください。

---
*この記事は [claudecode-lab.com](https://claudecode-lab.com/blog/claude-md-best-practices/) からの転載です。*
