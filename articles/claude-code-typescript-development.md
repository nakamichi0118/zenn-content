---
title: "Claude Code × TypeScript 型安全開発ガイド — strict mode・型エラー自動修正・zodバリデーション実装パターン"
emoji: "🦺"
type: "tech"
topics: ["claudecode", "typescript", "型安全", "zod", "development"]
published: true
---

## はじめに

TypeScript を使っていても、`any` だらけのコードになっていませんか?

Claude Code と TypeScript の strict mode を組み合わせると、型エラーを**自動で検出・修正**しながら開発できます。この記事では、私が実際に運用しているパターンを、動くコードとともに紹介します。

4つのテーマを扱います:
1. **tsconfig.json の strict mode 最適化**
2. **Claude Code による型エラーの自動修正ワークフロー**
3. **zod を使ったランタイムバリデーション**
4. **実務でよく踏む落とし穴とその回避策**

---

## 1. tsconfig.json: strict mode の正しい設定

### ベースライン設定

Claude Code に型安全開発を依頼する前提として、tsconfig.json を正しく設定する必要があります。

```json
// tsconfig.json
{
  "compilerOptions": {
    // 基本設定
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],

    // 出力設定
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,

    // 型安全性 (strict モード一括有効化)
    "strict": true,

    // strict に含まれる個別オプション (明示的に記述推奨)
    "noImplicitAny": true,          // any の暗黙的な使用を禁止
    "strictNullChecks": true,       // null/undefined の厳密チェック
    "strictFunctionTypes": true,    // 関数型の共変・反変チェック
    "strictBindCallApply": true,    // bind/call/apply の型チェック
    "strictPropertyInitialization": true, // クラスプロパティの初期化チェック
    "noImplicitThis": true,         // this の暗黙的な any を禁止

    // さらに厳しくする (推奨)
    "noUncheckedIndexedAccess": true, // インデックスアクセスの undefined チェック
    "noImplicitReturns": true,      // 全コードパスで return を強制
    "noFallthroughCasesInSwitch": true, // switch の fall-through チェック
    "exactOptionalPropertyTypes": true, // オプショナルプロパティの厳密な型

    // import 関連
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,

    // パス設定
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### Claude Code への依頼パターン

```
このプロジェクトのtsconfig.jsonを確認して、型安全性が最大になるよう
strict modeを設定して。既存のコードが壊れないか確認してから変更して。
```

---

## 2. Claude Code × 型エラー自動修正ワークフロー

### Step 1: 型エラーの一括検出

```bash
# 型エラーを JSON 形式で出力するスクリプト
npx tsc --noEmit --pretty false 2>&1 | \
  grep "error TS" | \
  awk '{print $1, $2}' > /tmp/type-errors.txt

echo "型エラー数: $(wc -l < /tmp/type-errors.txt)"
```

このスクリプトの出力を Claude Code に渡すと、一括修正を依頼できます。

### Step 2: CLAUDE.md でルールを事前に教え込む

```markdown
<!-- CLAUDE.md -->
## TypeScript 型安全ルール

### 禁止事項
- `any` 型の使用は禁止。`unknown` を使い、型ガードで絞る
- Non-null assertion operator (`!`) の使用禁止。Optional chaining (`?.`) を使う
- `@ts-ignore` の使用禁止。`@ts-expect-error` + コメントで理由を明記

### 型エラー修正の優先順位
1. まず型定義を確認・修正
2. 型ガードを使って型を絞る
3. 最終手段のみ as でキャスト (理由をコメントに書く)

### 外部データの型安全
- API レスポンスは必ず zod でバリデーション
- JSON.parse の結果は unknown 型として扱う
- localStorage からのデータも zod を通す
```

### Step 3: 型エラーの修正依頼

```
src/api/user.ts に型エラーが3件あります:
- line 45: Argument of type 'string | undefined' is not assignable to parameter of type 'string'
- line 67: Object is possibly 'null'
- line 89: Property 'email' does not exist on type '{}'

CLAUDE.md のルールに従って修正して。any は使わないこと。
```

### 型ガードパターンの実装例

```typescript
// 悪い例: any を使った逃げ
function processUser(data: any) {
  return data.email.toLowerCase();
}

// 良い例: 型ガードで安全に絞る
interface User {
  id: string;
  email: string;
  name: string;
}

// ユーザー定義型ガード
function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "email" in value &&
    "name" in value &&
    typeof (value as User).id === "string" &&
    typeof (value as User).email === "string" &&
    typeof (value as User).name === "string"
  );
}

// 安全に使う
function processUser(data: unknown): string {
  if (!isUser(data)) {
    throw new Error("Invalid user data");
  }
  // ここでは data が User 型として扱われる
  return data.email.toLowerCase();
}
```

---

## 3. zod によるランタイムバリデーション

TypeScript の型チェックはコンパイル時にしか機能しません。**実行時に外部から来るデータ**（APIレスポンス、フォーム入力、localStorage）は必ず zod でバリデーションが必要です。

### 基本パターン

```typescript
import { z } from "zod";

// スキーマ定義
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(["admin", "user", "guest"]),
  createdAt: z.string().datetime(),
  metadata: z
    .object({
      avatar: z.string().url().optional(),
      bio: z.string().max(500).optional(),
    })
    .optional(),
});

// スキーマから TypeScript 型を自動生成 (二重定義を避ける!)
type User = z.infer<typeof UserSchema>;

// API レスポンスのバリデーション
async function fetchUser(userId: string): Promise<User> {
  const response = await fetch(`/api/users/${userId}`);

  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }

  const rawData: unknown = await response.json();

  // ここで型安全を保証
  const result = UserSchema.safeParse(rawData);

  if (!result.success) {
    // zod がエラーの詳細を提供
    console.error("Validation failed:", result.error.format());
    throw new Error("Invalid API response");
  }

  return result.data; // User 型が保証される
}
```

### ネストした複雑なスキーマ

```typescript
import { z } from "zod";

// 再利用可能なパーツを定義
const PaginationSchema = z.object({
  page: z.number().int().positive(),
  perPage: z.number().int().min(1).max(100),
  total: z.number().int().nonnegative(),
});

const AddressSchema = z.object({
  street: z.string(),
  city: z.string(),
  country: z.string().length(2), // ISO 3166-1 alpha-2
  postalCode: z.string().regex(/^\d{5}(-\d{4})?$/, "Invalid postal code"),
});

// 複合スキーマ
const OrderSchema = z.object({
  id: z.string().uuid(),
  userId: z.string().uuid(),
  items: z
    .array(
      z.object({
        productId: z.string().uuid(),
        quantity: z.number().int().positive(),
        // 価格は小数点を含む可能性があるため number を使用
        unitPrice: z.number().positive().multipleOf(0.01),
      })
    )
    .min(1), // 少なくとも1件のアイテムが必要
  shippingAddress: AddressSchema,
  status: z.enum(["pending", "confirmed", "shipped", "delivered", "cancelled"]),
  // 合計金額はランタイムで計算 (transform を使用)
  total: z.number().positive(),
  createdAt: z.coerce.date(), // 文字列を Date に自動変換
});

type Order = z.infer<typeof OrderSchema>;

// リストレスポンス
const OrderListSchema = z.object({
  data: z.array(OrderSchema),
  pagination: PaginationSchema,
});

type OrderList = z.infer<typeof OrderListSchema>;

// 使用例
async function fetchOrders(page: number = 1): Promise<OrderList> {
  const response = await fetch(`/api/orders?page=${page}`);
  const rawData: unknown = await response.json();

  return OrderListSchema.parse(rawData); // 失敗時は ZodError をスロー
}
```

### フォームバリデーションへの応用

```typescript
import { z } from "zod";

// サインアップフォームのスキーマ
const SignUpSchema = z
  .object({
    email: z.string().email("有効なメールアドレスを入力してください"),
    password: z
      .string()
      .min(8, "パスワードは8文字以上必要です")
      .regex(/[A-Z]/, "大文字を1文字以上含めてください")
      .regex(/[0-9]/, "数字を1文字以上含めてください"),
    confirmPassword: z.string(),
    agreeToTerms: z.literal(true, {
      errorMap: () => ({ message: "利用規約に同意してください" }),
    }),
  })
  .refine((data) => data.password === data.confirmPassword, {
    // クロスフィールドバリデーション
    message: "パスワードが一致しません",
    path: ["confirmPassword"],
  });

type SignUpFormData = z.infer<typeof SignUpSchema>;

// React Hook Form との組み合わせ例
function validateSignUp(formData: unknown): {
  success: boolean;
  data?: SignUpFormData;
  errors?: Record<string, string>;
} {
  const result = SignUpSchema.safeParse(formData);

  if (!result.success) {
    const errors: Record<string, string> = {};
    result.error.errors.forEach((err) => {
      const path = err.path.join(".");
      errors[path] = err.message;
    });
    return { success: false, errors };
  }

  return { success: true, data: result.data };
}
```

---

## 4. Claude Code に依頼する型安全実装パターン

### 依頼テンプレート

効果的な依頼の書き方をいくつか紹介します。

**API クライアント生成:**
```
OpenAPI spec (openapi.yaml) から zod スキーマと TypeScript 型を生成して。
- レスポンスはすべて safeParse でバリデーション
- エラーハンドリングは Result 型パターン (never throw)
- 生成したコードは src/api/ に配置
```

**既存コードのリファクタリング:**
```
src/utils/ 以下の any を全部型安全に修正して。
条件:
- any → unknown + 型ガードに置き換え
- 外部データは zod でバリデーション追加
- 変更後に tsc --noEmit で型エラーがないことを確認
修正後の diff を見せて。
```

**型定義の生成:**
```
以下の JSON レスポンスから zod スキーマと TypeScript 型を生成して。
型名は "ProductResponse" にして。

{
  "id": "prod_123",
  "name": "商品A",
  "price": 1980,
  "stock": 50,
  "tags": ["new", "sale"],
  "images": [
    {"url": "https://example.com/img.jpg", "alt": "商品画像"}
  ]
}
```

---

## 5. 実務でよく踏む落とし穴と回避策

### 落とし穴 1: `noUncheckedIndexedAccess` で既存コードが壊れる

`noUncheckedIndexedAccess` を有効にすると、配列のインデックスアクセスが `T | undefined` になります。

```typescript
// tsconfig に noUncheckedIndexedAccess: true を追加後...

const users = ["Alice", "Bob", "Carol"];

// エラー! string | undefined を string に代入できない
const first: string = users[0]; // NG

// 修正パターン1: 型アサーション (証拠がある場合のみ)
const first = users[0]!; // Non-null assertion

// 修正パターン2: 条件チェック (推奨)
const first = users[0];
if (first === undefined) {
  throw new Error("配列が空です");
}
// ここで first は string 型

// 修正パターン3: at() メソッド (意図を明示)
const last = users.at(-1); // string | undefined を明示
if (last !== undefined) {
  console.log(last.toUpperCase());
}
```

**Claude Code への依頼:**
```
noUncheckedIndexedAccess を有効化した後の型エラーを修正して。
Non-null assertion (!) の乱用は避けて、適切な undefined チェックを追加して。
```

### 落とし穴 2: zod の `parse` vs `safeParse` の使い分け

```typescript
import { z } from "zod";

const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
});

// parse: 失敗時に ZodError をスロー
// → エラーをキャッチしないと runtime crash
try {
  const user = UserSchema.parse(untrustedData);
} catch (error) {
  if (error instanceof z.ZodError) {
    console.error(error.issues); // 詳細なエラー情報
  }
}

// safeParse: 失敗時に { success: false, error } を返す (推奨)
// → エラーハンドリングを強制できる
const result = UserSchema.safeParse(untrustedData);
if (!result.success) {
  // result.error は ZodError 型が保証される
  const formatted = result.error.format();
  return { error: "バリデーションエラー", details: formatted };
}
// result.data は UserSchema の型が保証される
const user = result.data;

// parseAsync / safeParseAsync: 非同期バリデーション
const asyncResult = await UserSchema.safeParseAsync(untrustedData);
```

**ベストプラクティス**: API境界では `safeParse`、内部ロジックで型が保証されている場合は `parse` を使います。

### 落とし穴 3: `exactOptionalPropertyTypes` で意図しないエラーが出る

```typescript
// tsconfig に exactOptionalPropertyTypes: true の場合

interface Config {
  timeout?: number; // number | undefined ではなく number のみ
}

// NG: undefined を明示的に設定できない
const config: Config = {
  timeout: undefined, // エラー!
};

// OK: プロパティ自体を省略する
const config: Config = {}; // timeout プロパティなし

// NG: 条件分岐でも注意
function createConfig(timeout?: number): Config {
  return {
    // timeout が undefined の場合にエラー
    timeout: timeout, // NG
  };
}

// OK: 条件的にプロパティを設定
function createConfig(timeout?: number): Config {
  return {
    ...(timeout !== undefined && { timeout }), // スプレッド構文
  };
}
```

### 落とし穴 4: zod スキーマと TypeScript 型の二重定義

```typescript
// NG: 二重定義は同期ズレのリスク
interface User {
  id: string;
  email: string;
}

const UserSchema = z.object({
  id: z.string(),
  email: z.string(),
  // 型を追加したのにスキーマを忘れた! → ランタイムでバリデーション漏れ
});

// OK: z.infer で型を自動生成
const UserSchema = z.object({
  id: z.string(),
  email: z.string(),
});

type User = z.infer<typeof UserSchema>; // スキーマから自動生成
// → スキーマを変更すれば型も自動的に変わる
```

### 落とし穴 5: `as` キャストの乱用

```typescript
// NG: 型チェックをバイパスする危険な使い方
const user = JSON.parse(response) as User; // ランタイムで壊れる

// NG: 危険なダウンキャスト
function getElement(id: string) {
  return document.getElementById(id) as HTMLInputElement; // null の場合にクラッシュ
}

// OK: 型ガードで安全に絞る
function getInputElement(id: string): HTMLInputElement {
  const element = document.getElementById(id);
  if (!(element instanceof HTMLInputElement)) {
    throw new Error(`Element #${id} is not an input element`);
  }
  return element; // 型が保証される
}

// OK: 外部データは zod で
const user = UserSchema.parse(JSON.parse(response)); // 型安全
```

---

## 6. CI/CD への組み込み

型チェックを CI に組み込むことで、型エラーを本番に持ち込まないようにします。

```yaml
# .github/workflows/type-check.yml
name: Type Check

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  typecheck:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: TypeScript type check
        run: npx tsc --noEmit

      - name: Run tests with type checking
        run: npm test
```

### Claude Code による CI 修正

型チェックが CI で失敗した場合の依頼:
```
GitHub Actions の型チェックが失敗しました。
エラーログ: [ログをペースト]

ローカルで `npx tsc --noEmit` を実行して型エラーを確認し、
CLAUDE.md のルールに従って修正してください。
修正後、再度 tsc で確認してから報告してください。
```

---

## まとめ

Claude Code × TypeScript 型安全開発のポイントを整理します。

| テーマ | 重要度 | アクション |
|--------|--------|-----------|
| strict mode 設定 | ★★★ | tsconfig.json に全オプション明示 |
| 型エラー一括修正 | ★★★ | tsc エラーを Claude Code に渡す |
| zod バリデーション | ★★★ | 外部データは必ず safeParse |
| 型定義の二重管理禁止 | ★★☆ | z.infer で型を自動生成 |
| as キャスト最小化 | ★★☆ | 型ガード + 例外スローで代替 |
| CI への組み込み | ★★☆ | tsc --noEmit を必須ステップに |

Claude Code に型安全のルールを CLAUDE.md で事前に教え込んでおくことが最大のポイントです。ルールなしに依頼すると `any` で逃げられることがあります。厳格なルールを与えれば、Claude Code は驚くほど高品質な型安全コードを生成してくれます。

## 関連記事

- [CLAUDE.md ベストプラクティス完全ガイド](https://zenn.dev/nakamichi0118/articles/claude-md-best-practices)
- [Claude Code セキュリティ自動化入門](https://zenn.dev/nakamichi0118/articles/claude-code-security-automation)
- [Claude Code 大規模リファクタリング実践](https://zenn.dev/nakamichi0118/articles/claude-code-large-refactoring)
