---
title: "Claude Code × Playwright E2Eテスト自動化|テスト設計から実行まで爆速で完成させる"
emoji: "🎭"
type: "tech"
topics: ["claudecode", "playwright", "e2e", "testing", "typescript"]
published: true
---

## はじめに

「E2Eテストを書きたいけど時間がない」——これは多くのエンジニアが抱える悩みです。Playwright はパワフルですが、セレクター設計・テストデータ管理・CI連携など、書くべきことが多い。

Claude Code と組み合わせると、**ゼロからE2Eテストスイートを1時間以内に整備**できます。実際に私がやっている方法を紹介します。

## プロジェクトセットアップ

```bash
# Playwright のインストール
npm init playwright@latest

# Claude Code に初期設定を任せる
claude -p "
playwright.config.ts を以下の設定で最適化して:
- テスト対象URL: http://localhost:3000
- ブラウザ: chromium のみ (CI高速化のため)
- タイムアウト: 30秒
- リトライ: CI環境では2回、ローカルは0回
- レポート: html + github
- スクリーンショット: 失敗時のみ
"
```

## テストの自動生成

### 既存画面からテストを生成

```bash
claude -p "
src/pages/Login.tsx を読んで、以下のケースをカバーする
Playwright テストを tests/login.spec.ts に作成:

1. 正常ログイン (メール+パスワード → ダッシュボードにリダイレクト)
2. パスワード間違い → エラーメッセージ表示
3. メールアドレス未入力 → バリデーションエラー
4. ログアウト → ログイン画面に戻る

page.getByRole() や page.getByLabel() などアクセシブルなセレクターを優先して
"
```

生成例:

```typescript
import { test, expect } from "@playwright/test";

test.describe("ログイン機能", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/login");
  });

  test("正常ログイン", async ({ page }) => {
    await page.getByLabel("メールアドレス").fill("test@example.com");
    await page.getByLabel("パスワード").fill("password123");
    await page.getByRole("button", { name: "ログイン" }).click();
    await expect(page).toHaveURL("/dashboard");
    await expect(page.getByRole("heading", { name: "ダッシュボード" })).toBeVisible();
  });

  test("パスワード間違い", async ({ page }) => {
    await page.getByLabel("メールアドレス").fill("test@example.com");
    await page.getByLabel("パスワード").fill("wrong-password");
    await page.getByRole("button", { name: "ログイン" }).click();
    await expect(page.getByRole("alert")).toContainText("メールアドレスまたはパスワードが間違っています");
  });

  test("メールアドレス未入力", async ({ page }) => {
    await page.getByRole("button", { name: "ログイン" }).click();
    await expect(page.getByText("メールアドレスを入力してください")).toBeVisible();
  });
});
```

### Playwright の Codegen を活用

```bash
# ブラウザで操作を録画してテストコードを生成
npx playwright codegen http://localhost:3000

# 録画したコードを Claude Code にリファクタさせる
claude -p "
以下の録画済みPlaywrightコードをリファクタして:
- ハードコードされた URL を baseURL 相対パスに
- 重複する操作を Page Object Model に抽出
- セレクターをアクセシブルな形式に変換
[ここに録画コードを貼る]
"
```

## Page Object Model の自動生成

画面単位でクラスを作り、テストから分離します。

```bash
claude -p "
tests/login.spec.ts を読んで、LoginPage の Page Object Model を
tests/pages/LoginPage.ts に生成して。
- コンストラクタで page を受け取る
- ログイン操作・フォーム入力・エラー取得などをメソッド化
- テストファイルも Page Object を使うように更新
"
```

```typescript
// tests/pages/LoginPage.ts
import { Page, Locator } from "@playwright/test";

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorAlert: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel("メールアドレス");
    this.passwordInput = page.getByLabel("パスワード");
    this.submitButton = page.getByRole("button", { name: "ログイン" });
    this.errorAlert = page.getByRole("alert");
  }

  async goto() {
    await this.page.goto("/login");
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async getErrorMessage() {
    return this.errorAlert.textContent();
  }
}
```

## テストデータ管理

```bash
claude -p "
tests/fixtures/ に以下のテストデータ fixture を作成:
- users.json: テスト用ユーザーアカウント (管理者・一般・停止アカウント)
- Playwright の test.extend で fixture として使える形式にする
- 実行前に DB をリセット、実行後にクリーンアップするフックも追加
"
```

```typescript
// tests/fixtures/index.ts
import { test as base } from "@playwright/test";
import { LoginPage } from "./pages/LoginPage";

type Fixtures = {
  loginPage: LoginPage;
  authenticatedPage: { page: LoginPage; userId: string };
};

export const test = base.extend<Fixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await use(loginPage);
  },

  authenticatedPage: async ({ page }, use) => {
    // テスト用ユーザーでログイン済みの状態を提供
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login("test@example.com", "password123");
    await use({ page: loginPage, userId: "test-user-id" });
  },
});
```

## CI/CD への組み込み

```bash
claude -p "
.github/workflows/e2e.yml を作成して:
- push / PR時に実行
- Node.js 20 セットアップ
- Playwright のインストールとキャッシュ
- 開発サーバーをバックグラウンドで起動してからテスト実行
- テスト失敗時にスクリーンショット・動画を artifact として保存
- 並列実行 (4 workers)
"
```

```yaml
name: E2E Tests
on: [push, pull_request]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20" }

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Start dev server
        run: npm run dev &
        env:
          NODE_ENV: test

      - name: Wait for server
        run: npx wait-on http://localhost:3000 --timeout 60000

      - name: Run E2E tests
        run: npx playwright test --workers=4

      - name: Upload test artifacts
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```

## 失敗テストのデバッグを Claude Code に任せる

```bash
claude -p "
以下の Playwright テストが失敗している:
$(cat test-results/failed-test.txt)

スクリーンショット: test-results/screenshot.png (読んで)
エラーメッセージ: TimeoutError: locator.click: Timeout 30000ms exceeded

原因を分析して、修正方法を提案して
"
```

Claude Code がスクリーンショットを読んで「このボタンはモーダルの後ろに隠れているから `waitForLoadState` が必要」などと教えてくれます。

## よくある落とし穴

**1. フレーキーテスト (不安定なテスト) を量産する**

```typescript
// ❌ 固定時間のwaitは不安定
await page.waitForTimeout(3000);

// ✅ 状態を待つ
await page.waitForLoadState("networkidle");
await expect(page.getByRole("button")).toBeEnabled();
```

Claude Code に「waitForTimeout を使わないで書き直して」と頼むだけで修正してくれます。

**2. セレクターが壊れやすい**

```typescript
// ❌ CSSクラスは変わりやすい
page.locator(".btn-primary-v2")

// ✅ 意味のある属性を使う
page.getByRole("button", { name: "送信" })
page.getByTestId("submit-button")  // data-testid を使う場合
```

**3. テストが順序依存になる**

各テストは独立して実行できるようにします。

```typescript
// ❌ 前のテストのデータに依存
test("ユーザー削除", async ({ page }) => {
  // 前のテストで作ったユーザーを削除しようとするが...
});

// ✅ テスト内で必要なデータを作成・削除
test("ユーザー削除", async ({ page }) => {
  const userId = await createTestUser(); // fixture で作る
  // ... テスト処理 ...
  await deleteTestUser(userId);         // テスト後にクリーンアップ
});
```

## まとめ

| フェーズ | Claude Code の貢献 |
|---------|------------------|
| セットアップ | `playwright.config.ts` 最適設定を生成 |
| テスト生成 | コンポーネントを読んでテストケースを網羅 |
| リファクタ | Page Object Model への整理 |
| CI連携 | GitHub Actions ワークフロー生成 |
| デバッグ | 失敗スクリーンショットを読んで原因分析 |

「E2Eテストを書く時間がない」は言い訳にならない時代になりました。Claude Code に「このページのテストを書いて」と頼むだけで、品質の高いE2Eテストが数分で完成します。

---

関連記事: [Claude Code × テスト自動化 Hooks ガイド](https://zenn.dev/masa_claudecodelab/articles/claude-code-hooks-auto-test)
