---
title: "Claude Codeで大規模リファクタリングを安全に進める実践手順"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "ai", "リファクタリング", "テスト", "開発効率化"]
published: true
---

## はじめに

「このコード、書き直したい」と思っても、大規模リファクタリングは怖い作業です。機能が壊れる、レビューが通らない、影響範囲が読めない——失敗リスクが大きすぎて、結局先送りされがち。

この記事では、Claude Code を相棒にして **大規模リファクタリングを安全に進める7ステップ** を紹介します。筆者が運営する多言語技術メディア（2,400ページ超）でも、この手順でコアモジュールの書き換えを何度も完走しています。

## ステップ1: リファクタ対象のセーフティネットを確認

まずテストがあるかを確認。なければ書く。リファクタ前のテストは命綱です。

```bash
claude -p "
src/services/OrderProcessor.ts のテストカバレッジを確認:

1. 既存テストが何をカバーしているか
2. カバーされていない分岐
3. リファクタ前に追加すべき特性テスト（Characterization Test）
4. エッジケース

テストが不十分ならテストケースを書き足す提案を出して。
"
```

:::message alert
**テストなしでのリファクタは禁止**

テストがないまま書き換えると、振る舞いが変わったことに気付けません。必ず先にテストを整備。
:::

## ステップ2: 差分を最小化する単位に分割

一気に書き換えず、小さな変更に分割します。

```bash
claude -p "
OrderProcessor.ts の全面書き換え計画を立てて:

1. 独立して変更可能な部分の特定
2. 変更順序（依存関係の少ない順）
3. 各ステップの差分サイズ（目安50行以下）
4. 各ステップで通すテスト
5. ロールバック戦略

レビュアブルなPR分割案として docs/refactor-plan.md に出力。
"
```

**1 PR = 1 ステップ** の原則を守ることで、レビュー負荷とリスクの両方を下げられます。

## ステップ3: Parallel Change パターンで進める

新旧コードを一時的に共存させ、段階的に切り替えます。

```bash
claude -p "
OrderProcessor の rewrite を Parallel Change で実装:

1. 新実装 OrderProcessorV2 を別ファイルに作成
2. 呼び出し元を Feature Flag で切り替え可能に
3. 段階的にトラフィックを新実装に流す
4. 旧実装の削除

各フェーズのコード例を示して。
"
```

これは Martin Fowler が [Parallel Change](https://martinfowler.com/bliki/ParallelChange.html) として体系化したパターンで、ロールバックが常に可能です。

## ステップ4: 振る舞いの同値性をテストで証明

新旧実装が同じ結果を返すことを自動テストで担保します。

```bash
claude -p "
OrderProcessor と OrderProcessorV2 の同値性テストを書いて:

1. 両方に同じ入力を渡して出力を比較
2. テストデータは既存テストから流用
3. 差分があれば fail する assertion
4. プロパティベーステスト（ランダム入力）も追加

test/equivalence/OrderProcessor.test.ts に出力。
"
```

これで「書き換えたら壊れた」が起きた瞬間に気付けます。

## ステップ5: 影響範囲を自動で調査

呼び出し元の漏れを防ぐ。

```bash
claude -p "
OrderProcessor を呼び出している箇所を全リストアップ:

1. grep/ast-grep で静的に呼び出し検出
2. 動的呼び出し（文字列経由・リフレクション）も注意
3. テスト内の使用箇所
4. 依存関係グラフを Mermaid で描画

docs/refactor-impact.md にまとめて。
"
```

人力で `grep` するより網羅性が高く、**見落としによる本番事故** を防げます。

## ステップ6: 小さなPRを連続して出す

レビュアーが1回で読める量に制限。

```bash
claude -p "
refactor-plan.md のステップ1について:

1. ブランチを切る: refactor/order-processor-step-1
2. 該当変更だけ実装
3. 既存テストが全て通ることを確認
4. 同値性テストも通す
5. PR を作成（タイトルに [Refactor 1/7] のプレフィックス）

本体ロジック変更なし、リネーム・分割のみに限定して。
"
```

`[Refactor X/Y]` プレフィックスでレビュアーが進捗を把握できます。

## ステップ7: ロールバック手順を事前に書く

何かあっても即復旧できるようにする。

```bash
claude -p "
今回のリファクタのロールバック手順を docs/rollback.md に:

1. Feature Flag を旧実装に戻す方法
2. デプロイロールバックの手順
3. DB スキーマ変更があればマイグレーション復元手順
4. 監視で見るべきメトリクス
5. 判断基準（いつロールバックすべきか）

オンコール担当者が夜中でも実行できる粒度で。
"
```

:::message
**ロールバックは書いて初めて使える**

手順がなければパニックで失敗します。必ず事前に文書化してドライラン。
:::

## やってはいけないこと

### ❌ 一気に全部書き換える
巨大PRはレビュー不可能、ロールバックも困難。**必ず分割**。

### ❌ テストなしで着手
リファクタの安全装置。既存テストが薄ければ特性テストを先に書く。

### ❌ Parallel Change をスキップ
新実装の切り替えは Feature Flag で段階的に。一発切り替えは事故の元。

### ❌ レビュアーを信頼しすぎる
レビュアーも人間。**セルフレビューで Claude Code に叩き出させる** のが大事。

## まとめ

- テストでセーフティネットを確保
- Parallel Change で段階移行
- 同値性テストで振る舞い保証
- 影響範囲を網羅調査
- 小さなPRで連続進行
- ロールバック手順を事前文書化
- 1ステップごとにデプロイして観測

大規模リファクタリングは「勇気」ではなく「手順」で進めます。Claude Code は、その手順を忠実に回す補助輪として強力です。

## 関連リソース

- [Martin Fowler: Parallel Change](https://martinfowler.com/bliki/ParallelChange.html)
- [Anthropic Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)

---

この記事は [ClaudeCodeLab](https://claudecode-lab.com/blog/claude-code-large-refactoring/) からの転載です（10言語対応のClaude Code専門メディア）。

Claude Code での業務自動化・研修・コンサルティングのご相談は [claudecode-lab.com](https://claudecode-lab.com) まで。X (Twitter): [@eru91115019](https://x.com/eru91115019)
