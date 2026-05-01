---
title: "Claude Code × Amazon Bedrock 連携ガイド｜AWS上でClaudeを呼び出す実践実装"
emoji: "🪨"
type: "tech"
topics: ["claudecode", "bedrock", "aws", "anthropic", "typescript"]
published: true
---

## はじめに

Claude Code で開発を進めていると「本番サービスでも Claude API を使いたい。でも Anthropic の API キーをそのまま使うのはセキュリティ的に不安…」という場面が出てきます。

そんな時の解答が **Amazon Bedrock** です。Bedrock を使えば、**AWS の IAM 認証を使って Claude を呼び出せる**ので、API キーの管理が不要になります。Claude Code で開発し、本番では Bedrock でデプロイ——この組み合わせが現時点での最強の選択肢です。

## Amazon Bedrock とは

Amazon Bedrock は、AWS が提供する **マネージド AI モデルサービス**。Claude (Anthropic)、Llama (Meta)、Titan (Amazon) など複数のモデルを統一した API で使えます。

Anthropic の API と Bedrock の主な違い:

| 比較軸 | Anthropic API | Amazon Bedrock |
|--------|--------------|----------------|
| 認証 | API キー | AWS IAM ロール |
| 請求 | Anthropic に直接 | AWS 請求に統合 |
| データ保持 | Anthropic のポリシー | AWS のポリシー |
| VPC 対応 | なし | PrivateLink で完全閉域 |
| コスト | 同等 (Bedrock は少し高め) | AWS 利用状況に合算 |

**エンタープライズ環境での採用が増えている理由**: IAM で権限管理できる + AWS の SOC2/ISO27001 準拠 + VPC 内で完結できる。

## セットアップ

### モデルアクセスの有効化

```bash
# まず AWS コンソールで Claude のモデルアクセスを申請
# Amazon Bedrock → Model access → Claude 系を有効化

# 有効化されているモデルを確認
aws bedrock list-foundation-models \
  --by-provider anthropic \
  --region us-east-1 \
  --query 'modelSummaries[].modelId'
```

注意: **Claude on Bedrock はリージョンが限られる**。2026年4月時点では us-east-1 と us-west-2 が主要対応リージョン。

### SDK のインストール

```bash
npm install @aws-sdk/client-bedrock-runtime
```

## 基本的な呼び出し

### Messages API (推奨)

```typescript
import {
  BedrockRuntimeClient,
  InvokeModelCommand,
} from "@aws-sdk/client-bedrock-runtime";

const client = new BedrockRuntimeClient({ region: "us-east-1" });

async function invokeClaudeOnBedrock(prompt: string): Promise<string> {
  const payload = {
    anthropic_version: "bedrock-2023-05-31",
    max_tokens: 1024,
    messages: [
      { role: "user", content: prompt }
    ],
  };

  const response = await client.send(
    new InvokeModelCommand({
      modelId: "anthropic.claude-opus-4-5",  // Bedrock 用のモデルID
      body: JSON.stringify(payload),
      contentType: "application/json",
      accept: "application/json",
    })
  );

  const result = JSON.parse(new TextDecoder().decode(response.body));
  return result.content[0].text;
}

// 使用例
const answer = await invokeClaudeOnBedrock("TypeScriptのベストプラクティスを3つ教えて");
console.log(answer);
```

### ストリーミング対応 (長い回答向け)

```typescript
import {
  BedrockRuntimeClient,
  InvokeModelWithResponseStreamCommand,
} from "@aws-sdk/client-bedrock-runtime";

async function streamClaude(prompt: string): Promise<void> {
  const payload = {
    anthropic_version: "bedrock-2023-05-31",
    max_tokens: 4096,
    messages: [{ role: "user", content: prompt }],
  };

  const response = await client.send(
    new InvokeModelWithResponseStreamCommand({
      modelId: "anthropic.claude-opus-4-5",
      body: JSON.stringify(payload),
      contentType: "application/json",
      accept: "application/json",
    })
  );

  // チャンクを受け取りながら表示
  for await (const chunk of response.body!) {
    if (chunk.chunk?.bytes) {
      const decoded = JSON.parse(new TextDecoder().decode(chunk.chunk.bytes));
      if (decoded.type === "content_block_delta") {
        process.stdout.write(decoded.delta.text ?? "");
      }
    }
  }
}
```

## Anthropic SDK から Bedrock を呼ぶ (推奨方法)

Anthropic の公式 SDK には Bedrock 対応が組み込まれています。コードの互換性が高く、移行が簡単です。

```bash
npm install @anthropic-ai/sdk @aws-sdk/credential-providers
```

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { fromNodeProviderChain } from "@aws-sdk/credential-providers";

// Bedrock 用クライアント初期化
const client = new Anthropic.AnthropicBedrock({
  awsRegion: "us-east-1",
  // IAM ロールで実行中なら awsAccessKey 等は不要
});

// 通常の Anthropic SDK と同じ書き方で使える
const message = await client.messages.create({
  model: "anthropic.claude-opus-4-5",  // Bedrock 用モデルID
  max_tokens: 1024,
  messages: [{ role: "user", content: "こんにちは" }],
});

console.log(message.content[0].text);
```

**重要**: Anthropic SDK を使う場合、モデルIDのプレフィックスが異なります。

```
Anthropic API: claude-opus-4-5
Bedrock:       anthropic.claude-opus-4-5
```

## Lambda + Bedrock のパターン

本番でよく使われる構成: Lambda から Bedrock を呼ぶ。

```typescript
// src/lambda/ai-handler.ts
import { Handler } from "aws-lambda";
import Anthropic from "@anthropic-ai/sdk";

// Lambda 実行時はコンテナ外で初期化 (コールドスタート高速化)
const bedrock = new Anthropic.AnthropicBedrock({
  awsRegion: process.env.AWS_REGION ?? "us-east-1",
});

export const handler: Handler = async (event) => {
  const { prompt, maxTokens = 512 } = event;

  try {
    const response = await bedrock.messages.create({
      model: "anthropic.claude-opus-4-5",
      max_tokens: maxTokens,
      messages: [{ role: "user", content: prompt }],
    });

    return {
      statusCode: 200,
      body: JSON.stringify({
        text: response.content[0].text,
        usage: response.usage,
      }),
    };
  } catch (error) {
    console.error("Bedrock error:", error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: "AI generation failed" }),
    };
  }
};
```

Lambda の IAM ロールに必要な権限:

```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModel",
    "bedrock:InvokeModelWithResponseStream"
  ],
  "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-*"
}
```

## Claude Code で Bedrock 実装を開発する

Claude Code でこの実装を開発する時の便利なプロンプト:

```bash
claude -p "
src/lib/bedrock.ts に以下を実装して:

1. AnthropicBedrock クライアントの初期化 (リージョンは環境変数から)
2. 単発呼び出し関数: generateText(prompt, maxTokens)
3. ストリーミング関数: streamText(prompt, onChunk)
4. エラーハンドリング:
   - ValidationException: プロンプトの問題
   - ThrottlingException: レート制限、リトライ
   - ModelTimeoutException: タイムアウト

テストファイルも src/lib/bedrock.test.ts に作成して
"
```

## コスト最適化

```typescript
// モデルによるコスト差
const MODELS = {
  // Bedrock 料金 (2026年4月時点、us-east-1)
  "anthropic.claude-haiku-4-5-20251001": {
    inputPer1M: 0.8,   // $0.80/1M tokens
    outputPer1M: 4.0,
  },
  "anthropic.claude-sonnet-4-6": {
    inputPer1M: 3.0,
    outputPer1M: 15.0,
  },
  "anthropic.claude-opus-4-5": {
    inputPer1M: 15.0,
    outputPer1M: 75.0,
  },
};

// タスクに応じてモデルを選択
function selectModel(task: "classify" | "summarize" | "complex"): string {
  if (task === "classify") return "anthropic.claude-haiku-4-5-20251001";
  if (task === "summarize") return "anthropic.claude-sonnet-4-6";
  return "anthropic.claude-opus-4-5";
}
```

## よくある落とし穴

**1. リージョンが対応していない**

Claude on Bedrock は全リージョンで使えません。`us-east-1` か `us-west-2` を使いましょう。Cross-region inference を使えば東京リージョン経由でも呼び出せます。

**2. モデルアクセスの申請を忘れる**

Bedrock はモデルごとに「アクセス申請」が必要。申請しないと `AccessDeniedException` が出ます。AWS コンソールの「Model access」で事前に有効化しておきましょう。

**3. Lambda のタイムアウト設定**

Claude の応答には数秒〜数十秒かかることがあります。Lambda のデフォルトタイムアウト (3秒) では足りないケースが多い。最低30秒、長い生成なら300秒に設定しましょう。

## まとめ

| ユースケース | おすすめ |
|------------|---------|
| プロトタイプ・個人開発 | Anthropic API (シンプル) |
| エンタープライズ・AWS 環境 | Amazon Bedrock (IAM認証・VPC対応) |
| コスト最適化 | Haiku on Bedrock |
| 高品質な生成 | Opus on Bedrock |

Claude Code で開発して Bedrock で本番運用——この組み合わせは、セキュリティと開発効率を両立する現時点のベストプラクティスです。

---

関連: [Claude Code × AWS Lambda ガイド](https://claudecode-lab.com/blog/claude-code-aws-lambda-complete-guide/) | [Claude Code × AWS IAM ガイド](https://claudecode-lab.com/blog/claude-code-aws-iam-guide/)
