---
title: "Claude CodeでCI/CDパイプラインを自動構築する実践ガイド"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "ai", "cicd", "githubactions", "開発効率化"]
published: true
---

Claude Code はコードを書くだけではなく、**CI/CDパイプライン全体の構築と運用**も任せられます。GitHub Actions の設定ファイル作成、自動テスト、デプロイまで、AIと対話しながら一気通貫で構築する方法を紹介します。

## 1. GitHub Actions ワークフロー自動生成

Claude Code に既存コードベースを読ませて、適切な CI 設定を自動生成させるのが第一歩です。

```bash
claude -p "
このプロジェクトの構成を調べて、以下の要件でGitHub Actionsワークフローを作成してください:

1. push時: lint → test → build の順で実行
2. PRオープン時: 上記 + typecheck
3. main ブランチへのpush時: 上記 + デプロイ

使用技術: Node.js 20, pnpm, Vitest, TypeScript
出力先: .github/workflows/ci.yml
"
```

すると実際にファイルが作成されて、すぐに動く YAML が手に入ります。ポイントは **既存の package.json スクリプトを Claude Code が勝手に読んで合わせてくれる**ことです。

## 2. 自動レビューをパイプラインに組み込む

PR オープン時に Claude Code 自身がコードレビューするワークフローを入れると、人間レビュアーの負担が大幅に減ります。

```yaml
name: Claude Auto Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run auto review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh pr diff ${{ github.event.pull_request.number }} | \
            claude -p "
          このPRを以下の観点でレビューし、問題があれば重要度付きで報告してください:
          1. バグの可能性
          2. セキュリティリスク
          3. パフォーマンスの懸念
          4. テストカバレッジの不足
          5. 命名・可読性
          "
```

`ANTHROPIC_API_KEY` は GitHub Secrets にセットしておきます。

## 3. 自動マージ条件を組む

テストが通ってClaude Codeのレビューも問題なければ、自動マージするワークフローも作れます。

```yaml
name: Auto Merge
on:
  pull_request_review:
    types: [submitted]

jobs:
  auto-merge:
    if: github.event.review.state == 'approved'
    runs-on: ubuntu-latest
    steps:
      - name: Check CI status
        run: |
          gh pr checks ${{ github.event.pull_request.number }} --required

      - name: Enable auto-merge
        if: success()
        run: |
          gh pr merge ${{ github.event.pull_request.number }} \
            --auto --squash
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

人間の承認 → CIグリーン → 自動マージ、という流れが完成します。

## 4. デプロイも Claude Code で

デプロイスクリプト自体を Claude Code に書かせるのも有効です。

```bash
claude -p "
Cloudflare Pages に Astro サイトをデプロイするワークフローを書いてください。
要件:
- main ブランチへの push で起動
- pnpm install && pnpm build
- wrangler pages deploy ./dist --project-name my-site
- 環境変数 CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID を使う
"
```

数秒で `.github/workflows/deploy.yml` が出来上がります。

## 5. 失敗時の自動リカバリ

ビルドが失敗したらClaude Codeに原因を調べさせるフローも組めます。

```yaml
- name: Diagnose on failure
  if: failure()
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
  run: |
    LOG=$(cat build.log 2>/dev/null || echo "")
    echo "$LOG" | claude -p "
    ビルドログを分析して、失敗の原因と修正方法を示してください。
    可能なら修正コードも提示してください。
    "
```

失敗時に自動でIssueを立てて原因分析を添付する、という運用にすると品質が上がります。

## 6. モノレポとの組み合わせ

Turborepo や Nx のモノレポでも、Claude Code は全体を俯瞰して適切なワークフローを組んでくれます。

```bash
claude -p "
このTurborepoの構成を調べて、変更があったパッケージのみテスト・ビルドする
GitHub Actions ワークフローを作成してください。

要件:
- turbo run test --filter で差分ビルド
- キャッシュを使う（actions/cache）
- 並列実行
"
```

パッケージが増えても同じワークフローが使い回せます。

## 7. シークレット管理の自動化

`.env.example` と既存の `.env` を比較して、足りないシークレットを検出するジョブも入れられます。

```bash
claude -p "
.env.example にあって GitHub Secrets に登録されていない
環境変数を列挙してください。gh secret list の結果と比較してください。
"
```

セットアップ漏れを事前に検出できます。

## ハマりどころ

### 1. API Key の取り扱い
`ANTHROPIC_API_KEY` は絶対にワークフローのログに出さないこと。GitHub Secrets に入れて `env:` 経由で渡します。

### 2. Claude Code のレート制限
同時実行が多いとレート制限に引っかかります。matrix ビルドで大量のジョブを回す場合は、`concurrency:` で制限をかけましょう。

### 3. コスト管理
自動レビューを PR ごとに走らせるとAPI利用料が増えます。`paths-ignore:` でドキュメントのみの変更は除外すると節約できます。

```yaml
on:
  pull_request:
    paths-ignore:
      - 'docs/**'
      - '*.md'
```

## まとめ

- ワークフロー自動生成で初期設定が一瞬で終わる
- PR 自動レビューで人間の負担が激減
- デプロイから障害対応まで AI がサポート
- モノレポ・シークレット管理・コスト最適化まで対応可能

Claude Code は単なる「AIコーディングアシスタント」ではなく、**DevOps全体のコパイロット**として使える時代になりました。

---

※この記事は [claudecode-lab.com](https://claudecode-lab.com/blog/claude-code-ci-cd-setup/) の一部を Zenn 向けに再構成したものです。より詳しい実装例やCIテンプレート集は本サイトをご覧ください。
