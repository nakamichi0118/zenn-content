---
title: "Claude Code のAPIコストを90%削減する実践テクニック｜プロンプトキャッシュ・モデル使い分け・バッチ最適化"
emoji: "💰"
type: "tech"
topics: ["claudecode", "anthropic", "cost", "optimization", "llm"]
published: true
---

## はじめに

「Claude Code を使い込んでいたら月のAPI料金が想定の10倍になった」——この経験をしたエンジニアは少なくありません。Claude Code は強力な反面、**使い方によってコストが10倍以上変わります**。

この記事では、**APIコストを最大90%削減できる実践テクニック**を解説します。プロンプトキャッシュの仕組み、モデルの使い分け、バッチ処理最適化まで、数値根拠付きで紹介します。

## まずコストの仕組みを理解する

Claude API の料金はシンプルです。

```
コスト = (入力トークン × 入力単価) + (出力トークン × 出力単価)
```

### Claude Opus 4.6 の料金 (2026年4月時点)

| 種別 | 単価 |
|------|------|
| 入力 (通常) | $15 / 1M tokens |
| 入力 (キャッシュ読み込み) | $1.50 / 1M tokens |
| 入力 (キャッシュ書き込み) | $18.75 / 1M tokens |
| 出力 | $75 / 1M tokens |

**キャッシュ読み込みは通常入力の1/10のコスト**。この差を活かすのが最大のコスト削減手段です。

## テクニック1: プロンプトキャッシュを使い倒す

プロンプトキャッシュ (Prompt Caching) は、**同じ内容を繰り返し送らないようにする仕組み**です。TTL は5分。5分以内に同じ内容が来たらキャッシュから返し、入力コストが1/10になります。

### 使い方

```typescript
import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic();

// ❌ キャッシュなし: 毎回全文送信 (高コスト)
const response = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 1024,
  system: longSystemPrompt,  // 毎回フル課金
  messages: [{ role: "user", content: userMessage }],
});

// ✅ キャッシュあり: 初回のみ書き込み、2回目以降は1/10コスト
const response = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 1024,
  system: [
    {
      type: "text",
      text: longSystemPrompt,
      cache_control: { type: "ephemeral" },  // ← これだけ追加
    },
  ],
  messages: [{ role: "user", content: userMessage }],
});
```

### キャッシュが効くケース・効かないケース

```
✅ 効く:
- CLAUDE.md の内容 (毎回同じ)
- コードベースの要約 (セッション中は同じ)
- システムプロンプト (役割・ルールの定義)
- 長いドキュメントの参照

❌ 効かない:
- ユーザーの入力部分 (毎回変わる)
- 現在時刻やランダム値を含む内容
- セッションをまたいだ場合 (TTL 5分)
```

### コスト試算: 1000回呼び出しの場合

```
system prompt: 10,000 tokens

【キャッシュなし】
1,000回 × 10,000tokens × $15/1M = $150

【キャッシュあり】
初回書き込み: 10,000tokens × $18.75/1M = $0.19
999回読み込み: 999 × 10,000tokens × $1.50/1M = $14.99
合計: $15.18

削減額: $134 (89%削減) 🎯
```

## テクニック2: モデルを用途で使い分ける

全ての処理に Opus を使うのは過剰です。**タスクの複雑さに応じてモデルを切り替える**だけで大幅削減できます。

### モデル比較

| モデル | 入力単価 | 向いているタスク |
|--------|---------|----------------|
| claude-opus-4-6 | $15/1M | 複雑な設計・コードレビュー・難しいデバッグ |
| claude-sonnet-4-6 | $3/1M | 一般的な実装・リファクタリング・ドキュメント生成 |
| claude-haiku-4-5 | $0.80/1M | 単純な変換・フォーマット・分類・翻訳 |

**Opus と Haiku の価格差は約19倍**。翻訳や単純なコード変換に Opus を使うのは大きなムダです。

### 使い分けの実装例

```typescript
function selectModel(taskType: "design" | "implement" | "translate" | "format") {
  const modelMap = {
    design: "claude-opus-4-6",      // 設計・複雑な問題解決
    implement: "claude-sonnet-4-6", // 一般的な実装
    translate: "claude-haiku-4-5-20251001",  // 翻訳・変換
    format: "claude-haiku-4-5-20251001",     // フォーマット・整形
  };
  return modelMap[taskType];
}

// 使用例: 10言語翻訳はHaikuで十分
const translations = await Promise.all(
  languages.map(lang =>
    client.messages.create({
      model: selectModel("translate"),  // Haiku = 1/19のコスト
      max_tokens: 4096,
      messages: [{ role: "user", content: `Translate to ${lang}: ${text}` }],
    })
  )
);
```

### 実際の削減効果

```
翻訳タスク 1000回 (各2000 tokens 入力):
Opus:  1000 × 2000 × $15/1M   = $30
Haiku: 1000 × 2000 × $0.80/1M = $1.60

削減額: $28.40 (94.7%削減) 🎯
```

## テクニック3: 出力トークンを絞る

**出力単価は入力の5倍**。不必要な長文出力を制限するだけでコストが激変します。

```typescript
// ❌ 出力を絞らない
const response = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 4096,  // 最大4096tokens出力される可能性
  messages: [{ role: "user", content: "このコードの問題を教えて" }],
});

// ✅ 用途に応じて上限を設定
const response = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 512,   // 問題指摘だけなら512で十分
  messages: [{
    role: "user",
    content: "このコードの問題を3行以内で教えて",  // 出力長を指示
  }],
});
```

### プロンプトで出力を制御する

```
❌ 「このコードをレビューして」(無制限に長くなる)
✅ 「このコードの問題点を箇条書き5項目以内で列挙」
✅ 「修正が必要な箇所だけ diff 形式で出力」
✅ 「Yes/No で回答し、理由は1行で」
```

## テクニック4: バッチ処理で並列実行

Claude API は **同時に複数リクエストを投げても料金は変わりません**。逐次実行より並列実行でスループットを上げ、課金時間を短縮しましょう。

```typescript
// ❌ 逐次実行: 10件 × 2秒 = 20秒、待機時間も課金対象の処理が増える
for (const item of items) {
  await processWithClaude(item);
}

// ✅ 並列実行: 全件同時 = 2秒
const results = await Promise.all(items.map(processWithClaude));

// ✅ 並列数を制御したい場合
import pLimit from "p-limit";
const limit = pLimit(5);  // 最大5並列
const results = await Promise.all(
  items.map(item => limit(() => processWithClaude(item)))
);
```

## テクニック5: コンテキストを圧縮する

長い会話になるほど入力トークンが膨らみます。**定期的に会話を要約**してコンテキストを圧縮しましょう。

```typescript
async function compressConversation(
  messages: Message[],
  threshold = 50000  // 50kトークン超えたら圧縮
): Promise<Message[]> {
  const tokenCount = estimateTokens(messages);
  if (tokenCount < threshold) return messages;

  // 古い会話を要約
  const summary = await client.messages.create({
    model: "claude-haiku-4-5-20251001",  // 要約はHaikuで十分
    max_tokens: 500,
    messages: [{
      role: "user",
      content: `以下の会話を500文字以内で要約:\n${JSON.stringify(messages.slice(0, -4))}`,
    }],
  });

  // 要約 + 直近4件のメッセージを保持
  return [
    { role: "user", content: `[過去の会話要約]\n${summary.content[0].text}` },
    { role: "assistant", content: "了解しました。" },
    ...messages.slice(-4),
  ];
}
```

Claude Code の `/compact` コマンドはこれと同等の処理を自動で行います。

## 月間コスト試算: 最適化前後の比較

```
【典型的な使い方: 1日100回の対話、system prompt 5000tokens、平均応答 1000tokens】

最適化前:
入力: 100回 × 5000tokens × $15/1M = $7.50/日 → $225/月
出力: 100回 × 1000tokens × $75/1M = $7.50/日 → $225/月
合計: $450/月

最適化後:
入力 (キャッシュ): 100回 × 5000tokens × $1.50/1M = $0.75/日 → $22.5/月
出力 (max_tokens絞る): 100回 × 300tokens × $75/1M = $2.25/日 → $67.5/月
合計: $90/月

削減額: $360/月 (80%削減) 🎯
```

## すぐに実装できるチェックリスト

```
コスト削減チェックリスト

今日やること (30分):
□ system prompt に cache_control: { type: "ephemeral" } を追加
□ 翻訳・変換タスクを Haiku に切り替え
□ max_tokens を用途に応じて設定 (デフォルト4096は過剰)

今週やること:
□ 複雑なタスクと単純なタスクでモデルを分岐
□ 逐次処理を Promise.all() で並列化
□ 長い会話に /compact を定期実行

今月やること:
□ Anthropic コンソールで Usage を確認し、コストの大きいリクエストを特定
□ コスト上限 (Budget Alert) を設定してアラート受信
□ 月間コストレポートを自動生成
```

## まとめ

| テクニック | 削減効果 | 実装コスト |
|-----------|---------|----------|
| プロンプトキャッシュ | **最大89%** | 1行追加 |
| モデル使い分け | **最大95%** | 数行 |
| 出力トークン制限 | **30-60%** | プロンプト改善 |
| 並列実行 | スループット向上 | Promise.all化 |
| コンテキスト圧縮 | **20-40%** | 関数実装 |

一番コスパが良いのは**プロンプトキャッシュの追加 (1行で最大89%削減)**。今日中に実装できます。まずここから始めて、余裕が出たらモデル使い分けを導入するのが現実的な順番です。

## 参考資料

- [Anthropic Pricing 公式](https://www.anthropic.com/pricing)
- [Prompt Caching 公式ドキュメント](https://docs.anthropic.com/claude/docs/prompt-caching)
- [Claude Code 公式ガイド](https://docs.anthropic.com/claude/docs/claude-code)
