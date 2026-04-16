---
title: "Claude Code × MCP サーバー自作ガイド｜独自ツールを生やす実践手順"
emoji: "🔌"
type: "tech"
topics: ["claudecode", "mcp", "typescript", "anthropic", "ai"]
published: true
---

## はじめに

Claude Code を使い込んでいると必ず訪れるのが **「標準ツールだけじゃ足りない」** という壁。社内DBを叩きたい、Jiraチケットを直接操作したい、独自の社内API を呼びたい——そんな時の解答が **MCP (Model Context Protocol)** です。

MCPは Anthropic が 2024年末にオープンソース化した、**AIモデルと外部ツールを接続するための標準プロトコル**。Claude Code はこのMCPサーバーを起動時に読み込み、独自ツールを生やすことができます。

この記事では、TypeScriptで **自作MCPサーバーを作り、Claude Codeに組み込むまでの全工程** を、動くコードと実例付きで解説します。

## MCPとは何か

MCPは一言でいうと **「AIアシスタント用のプラグイン仕様」** です。VS Code拡張や Chrome Extension と同じ発想で、Claude Code に **あなた専用の能力** を追加できます。

### 仕組みの概要

```
┌──────────────┐      JSON-RPC       ┌─────────────────┐
│ Claude Code  │ ◄────────────────► │  MCP サーバー    │
│  (クライアント)│                     │  (あなたのコード) │
└──────────────┘                     └─────────────────┘
                                          │
                                          ├─ DB
                                          ├─ 社内API
                                          └─ 独自ロジック
```

- **プロトコル**: JSON-RPC 2.0 over stdio または HTTP
- **通信内容**: ツール一覧の公開、ツール実行、リソース提供
- **言語非依存**: TypeScript / Python / Go など何でも書ける

### なぜMCPが重要なのか

従来は「Claude に社内DBを叩かせたい」と思っても:

1. プロンプトで指示してコピペしてもらう (非効率)
2. Bash ツールで直接 `psql` を叩かせる (危険)
3. カスタムCLIを作ってBashから呼ぶ (毎回プロンプトで説明)

MCPなら **「社内DB検索ツール」として公式に登録** でき、Claude Code が自然に認識・呼び出してくれます。

## 最小構成のMCPサーバーを書いてみる

TypeScript + 公式SDKで作ります。題材は「**社内メンバー情報を検索するツール**」。

### セットアップ

```bash
mkdir my-mcp-server && cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node tsx
npx tsc --init
```

`tsconfig.json` に以下を追加:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true
  }
}
```

### サーバー実装

`src/index.ts`:

```typescript
#!/usr/bin/env node
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import { z } from "zod";

// --- ダミーの社内メンバーDB ---
const MEMBERS = [
  { id: "u001", name: "田中太郎", dept: "開発", skills: ["TypeScript", "React"] },
  { id: "u002", name: "佐藤花子", dept: "デザイン", skills: ["Figma", "UI"] },
  { id: "u003", name: "鈴木一郎", dept: "開発", skills: ["Go", "AWS"] },
];

// --- MCPサーバー初期化 ---
const server = new Server(
  { name: "member-search", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// --- ツール一覧を返す ---
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "search_member_by_skill",
      description: "指定スキルを持つ社内メンバーを検索",
      inputSchema: {
        type: "object",
        properties: {
          skill: { type: "string", description: "検索対象スキル名 (例: TypeScript)" },
        },
        required: ["skill"],
      },
    },
    {
      name: "get_member_detail",
      description: "メンバーIDから詳細情報を取得",
      inputSchema: {
        type: "object",
        properties: {
          id: { type: "string", description: "メンバーID (u001形式)" },
        },
        required: ["id"],
      },
    },
  ],
}));

// --- ツール実行を処理 ---
server.setRequestHandler(CallToolRequestSchema, async (req) => {
  const { name, arguments: args } = req.params;

  if (name === "search_member_by_skill") {
    const schema = z.object({ skill: z.string() });
    const { skill } = schema.parse(args);
    const hits = MEMBERS.filter((m) =>
      m.skills.some((s) => s.toLowerCase().includes(skill.toLowerCase()))
    );
    return {
      content: [
        { type: "text", text: JSON.stringify(hits, null, 2) },
      ],
    };
  }

  if (name === "get_member_detail") {
    const schema = z.object({ id: z.string() });
    const { id } = schema.parse(args);
    const member = MEMBERS.find((m) => m.id === id);
    if (!member) {
      return {
        content: [{ type: "text", text: `ID '${id}' のメンバーは見つかりません` }],
        isError: true,
      };
    }
    return { content: [{ type: "text", text: JSON.stringify(member, null, 2) }] };
  }

  throw new Error(`Unknown tool: ${name}`);
});

// --- stdio で起動 ---
const transport = new StdioServerTransport();
await server.connect(transport);
console.error("MCP server 'member-search' running on stdio");
```

### ビルド

```bash
npx tsc
chmod +x dist/index.js
```

これでMCPサーバーの完成です。

## Claude Code に組み込む

`~/.claude.json` (またはプロジェクト毎の `.claude/settings.json`) に以下を追加:

```json
{
  "mcpServers": {
    "member-search": {
      "command": "node",
      "args": ["/absolute/path/to/my-mcp-server/dist/index.js"]
    }
  }
}
```

Claude Code を再起動すると、ツール一覧に `mcp__member-search__search_member_by_skill` などが登場します。

### 動作確認

```
> TypeScript が書けるメンバーを教えて

[Claude Code が自動的に search_member_by_skill({skill: "TypeScript"}) を呼ぶ]

田中太郎さん (u001) が TypeScript と React を使えます。
開発部所属です。
```

**ポイント**: プロンプトで「ツール使って」と指示する必要はありません。Claude Code はツールの `description` を読んで適切なタイミングで呼び出します。

## 実用的なMCPサーバー例 3選

### 例1: Linear チケット操作

```typescript
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "create_linear_issue",
      description: "Linear にチケットを作成",
      inputSchema: {
        type: "object",
        properties: {
          title: { type: "string" },
          description: { type: "string" },
          team: { type: "string", description: "チームキー (例: ENG)" },
        },
        required: ["title", "team"],
      },
    },
  ],
}));

// 実装で LINEAR_API_KEY を使って fetch するだけ
```

Claude Code で「この設計レビューで出た指摘をLinearにチケット化して」と言えば、議論をまとめてまとめて起票してくれます。

### 例2: 社内Wiki検索

```typescript
{
  name: "search_internal_wiki",
  description: "社内 Confluence / Notion を全文検索",
  inputSchema: {
    type: "object",
    properties: { query: { type: "string" } },
    required: ["query"],
  },
}
```

「このサービスのデプロイ手順知ってる?」と聞くだけで、Claude Code が社内Wikiを検索して答えます。**社外に情報を出さずに** 社内知識を活用できる強みがあります。

### 例3: カスタムLinter

```typescript
{
  name: "check_naming_convention",
  description: "社内命名規約チェッカー (camelCase/kebab-case/ディレクトリ命名)",
  inputSchema: {
    type: "object",
    properties: { file_path: { type: "string" } },
    required: ["file_path"],
  },
}
```

汎用Linterでは捕まえられない社内独自ルールをMCPで実装。コード生成時に自動で規約準拠になります。

## 落とし穴5選

**1. stdio通信のログ出力に注意**
stdio transportは **stdoutを JSON-RPCに独占** します。デバッグ出力は必ず `console.error()` で stderr に出すこと。`console.log()` を使うとプロトコルが壊れます。

**2. 認証情報の扱い**
API キーなどは環境変数経由で。設定ファイルに直書きするとリポジトリに漏れます。

```json
{
  "mcpServers": {
    "linear": {
      "command": "node",
      "args": ["dist/index.js"],
      "env": { "LINEAR_API_KEY": "${LINEAR_API_KEY}" }
    }
  }
}
```

**3. ツール数を増やしすぎない**
1サーバー内のツールは **5-10個** に絞る。20個以上あるとモデルが選択に迷います。機能が多い場合はサーバー自体を分割 (`member-search`, `ticket-manager` など) する方が良いです。

**4. エラーメッセージはLLMが読める形式に**
`throw new Error("Failed")` ではなく、**何が悪くてどうすれば直るか** まで書くと Claude Code が自己修復します。

```typescript
throw new Error(
  `Linear API returned 401. LINEAR_API_KEY 環境変数を確認してください。` +
  `設定方法: .claude/settings.json の env に追加`
);
```

**5. 変更したら Claude Code の再起動が必要**
ツール定義を変えたら、Claude Code を一度終了して再起動しないと反映されません。開発中は特にハマりポイント。

## デバッグ Tips

Claude Code は MCP サーバーのログを以下で確認できます:

```bash
tail -f ~/.claude/logs/mcp-*.log
```

起動失敗時は大抵 stderr に原因が出ています。`console.error("debug info")` を活用しましょう。

## まとめ

MCP を使いこなせば、Claude Code は **「汎用AIアシスタント」** から **「あなたの業務に最適化された AIエージェント」** に進化します。

ステップをまとめると:

1. `@modelcontextprotocol/sdk` で TypeScript サーバーを書く
2. `ListTools` と `CallTool` のハンドラを実装
3. `dist/index.js` をビルドして `~/.claude.json` に登録
4. Claude Code 再起動で使えるようになる

最初の1本を作れば、2本目以降はコピペで15分で作れるようになります。社内DB、社内API、社内Wiki…あなたの業務特有の「あと一歩」を埋める決定打として、ぜひ活用してみてください。

## 参考資料

- [Model Context Protocol 公式ドキュメント](https://modelcontextprotocol.io/)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Claude Code 公式ガイド](https://docs.anthropic.com/claude/docs/claude-code)
