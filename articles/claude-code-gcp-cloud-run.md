---
title: "Claude Code × GCP Cloud Run 完全デプロイガイド｜コンテナをサーバーレスで運用"
emoji: "☁️"
type: "tech"
topics: ["claudecode", "gcp", "cloudrun", "docker", "typescript"]
published: true
---

## はじめに: なぜ Cloud Run なのか

claudecode-lab.com を運営している Masa です。このブログのバックエンド処理（画像リサイズ API・メール配信サービス・検索インデックス更新）を、最初は AWS Lambda で運用していました。ところが GCP に移行してみると、**Cloud Run の方が「コンテナをそのまま動かせる」という点で Claude Code との相性が抜群に良かった**のです。

Lambda は ZIP ファイルやレイヤーの管理があり、ランタイムのバージョン縛りもあります。Cloud Run は「Dockerfile さえあれば動く」。Claude Code が生成した Dockerfile をそのまま `gcloud run deploy` で動かせるシンプルさが、開発サイクルを大幅に短縮しました。

この記事では、**Claude Code × Cloud Run の実践パターン**を、デプロイから自動スケーリング・Secret Manager・CI/CD・カスタムドメインまで、実際に動くコードとともに解説します。

---

## Cloud Run の基本概念

Cloud Run は Google Cloud のサーバーレスコンテナ実行環境です。Kubernetes の複雑さなしに、コンテナをスケーラブルに動かせます。

**特徴**:
- トラフィックがないとき **0 インスタンスにスケールダウン** → コスト ₀
- リクエストが来たら自動でスケールアップ（コールドスタートは数百ms）
- Dockerfile さえあれば何でも動く（Node.js / Python / Go / Java など）
- 最大 **60分** のリクエストタイムアウト (Lambda の 15分 制限より長い)
- リクエスト単位の課金

---

## Step 1: Cloud Run 用 TypeScript API の Dockerfile

まず Claude Code に Dockerfile を生成させます。

### Claude Code へのプロンプト

```
以下の条件で Cloud Run にデプロイするための本番用 Dockerfile を生成して。

- Node.js 20 + TypeScript
- Express.js API サーバー
- マルチステージビルド (builder → runner)
- 非 root ユーザーで実行
- ヘルスチェックエンドポイント GET /health
- ポートは環境変数 PORT から取得 (Cloud Run はデフォルト 8080)
- .dockerignore も作成
```

### 生成される Dockerfile

```dockerfile
# --- Builder ステージ ---
FROM node:20-alpine AS builder
WORKDIR /app

# 依存関係を先にコピー (キャッシュ活用)
COPY package*.json ./
RUN npm ci --frozen-lockfile

# ソースをコピーしてビルド
COPY . .
RUN npm run build

# 本番依存のみに絞る
RUN npm prune --production

# --- Runner ステージ ---
FROM node:20-alpine AS runner
WORKDIR /app

# セキュリティ: 非 root ユーザー
RUN addgroup --system --gid 1001 nodejs \
  && adduser --system --uid 1001 --ingroup nodejs appuser

COPY --from=builder --chown=appuser:nodejs /app/dist ./dist
COPY --from=builder --chown=appuser:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:nodejs /app/package.json ./

USER appuser

# Cloud Run は PORT 環境変数で指定されたポートをリッスンする必要がある
ENV PORT=8080
EXPOSE 8080

# ヘルスチェック (Cloud Run の startup probe と連携)
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD wget -qO- http://localhost:${PORT}/health || exit 1

CMD ["node", "dist/index.js"]
```

```text
# .dockerignore
node_modules
dist
.git
*.md
.env*
coverage
.nyc_output
**/*.test.ts
**/*.spec.ts
```

### Express サーバーの実装

```typescript
// src/index.ts
import express from "express";

const app = express();
const PORT = parseInt(process.env.PORT ?? "8080", 10);

app.use(express.json());

// Cloud Run の health check エンドポイント
// startup probe / liveness probe がここをチェックする
app.get("/health", (_req, res) => {
  res.json({
    status: "healthy",
    timestamp: new Date().toISOString(),
    version: process.env.K_REVISION ?? "local", // Cloud Run はリビジョン名を注入
  });
});

// サンプル API エンドポイント
app.get("/api/hello", (_req, res) => {
  res.json({ message: "Hello from Cloud Run!" });
});

// グレースフルシャットダウン (Cloud Run はSIGTERMを送る)
process.on("SIGTERM", () => {
  console.log("SIGTERM received, shutting down gracefully");
  server.close(() => {
    console.log("Server closed");
    process.exit(0);
  });
});

const server = app.listen(PORT, () => {
  console.log(`Server listening on port ${PORT}`);
});
```

---

## Step 2: Cloud Run への初回デプロイ

### 前提条件

```bash
# gcloud CLI のインストール・認証
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

# Artifact Registry のリポジトリ作成 (初回のみ)
gcloud artifacts repositories create app-images \
  --repository-format=docker \
  --location=asia-northeast1 \
  --description="Docker images for Cloud Run"

# Docker の認証設定
gcloud auth configure-docker asia-northeast1-docker.pkg.dev
```

### ビルド & デプロイコマンド

```bash
# 環境変数
PROJECT_ID=$(gcloud config get-value project)
REGION="asia-northeast1"
SERVICE_NAME="my-api"
IMAGE="asia-northeast1-docker.pkg.dev/${PROJECT_ID}/app-images/${SERVICE_NAME}"

# 1. Docker イメージのビルド
docker build -t "${IMAGE}:latest" .

# 2. Artifact Registry にプッシュ
docker push "${IMAGE}:latest"

# 3. Cloud Run へデプロイ
gcloud run deploy ${SERVICE_NAME} \
  --image "${IMAGE}:latest" \
  --region ${REGION} \
  --platform managed \
  --allow-unauthenticated \      # 公開APIの場合
  --port 8080 \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 0 \            # コスト削減: ゼロスケール
  --max-instances 10 \           # 最大インスタンス数
  --timeout 60 \                 # リクエストタイムアウト (秒)
  --set-env-vars "NODE_ENV=production"
```

デプロイ後、Cloud Run が自動でカスタムドメインなしの URL (`https://my-api-xxxx-an.a.run.app`) を発行します。

---

## Step 3: 自動スケーリング設定

Cloud Run のスケーリングは細かく制御できます。

### Claude Code へのプロンプト

```
Cloud Run の自動スケーリングで以下を実現する gcloud コマンドを教えて:
- 通常時: 1〜3インスタンス (コールドスタートを防ぐ)
- バースト時: 最大20インスタンム
- CPU: 1vCPU、同時リクエスト数: 80 (デフォルト80)
- メモリ: 1Gi
```

### スケーリング設定コマンド

```bash
gcloud run services update my-api \
  --region asia-northeast1 \
  --min-instances 1 \            # 常時1インスタンス確保 (コールドスタートなし)
  --max-instances 20 \
  --concurrency 80 \             # 1インスタンスが同時に処理するリクエスト数
  --cpu 1 \
  --memory 1Gi \
  --cpu-boost                    # 起動時にCPUを一時的に倍増 (コールドスタート短縮)
```

### スケーリングの考え方

| 用途 | min-instances | max-instances | concurrency |
|------|--------------|---------------|-------------|
| 開発・低トラフィック | 0 | 5 | 80 |
| 本番 API (レイテンシ重視) | 1〜3 | 50 | 80 |
| バッチ処理 | 0 | 100 | 1 (並列処理なし) |
| WebSocket | 1 | 20 | 1000 |

**ポイント**: `--concurrency 1` にすると、1リクエスト = 1インスタンスになります。CPU 集中型の処理はここを下げると安全です。

---

## Step 4: Secret Manager 連携

本番環境の API キー・DB パスワードは **Secret Manager** で管理します。環境変数に直接書かないことが原則です。

### シークレットの作成

```bash
# データベース URL をシークレットに登録
echo -n "postgresql://user:password@host:5432/dbname" | \
  gcloud secrets create DATABASE_URL \
    --data-file=- \
    --replication-policy="automatic"

# API キーを登録
echo -n "sk-your-api-key-here" | \
  gcloud secrets create OPENAI_API_KEY \
    --data-file=-
```

### Cloud Run サービスにシークレットをマウント

```bash
gcloud run services update my-api \
  --region asia-northeast1 \
  --set-secrets "DATABASE_URL=DATABASE_URL:latest,OPENAI_API_KEY=OPENAI_API_KEY:latest"
```

これで Cloud Run コンテナの環境変数 `DATABASE_URL` と `OPENAI_API_KEY` に自動的にシークレットの値が注入されます。

### TypeScript でのアクセス

```typescript
// 環境変数から取得するだけ (Secret Manager SDK 不要!)
const dbUrl = process.env.DATABASE_URL;
if (!dbUrl) {
  throw new Error("DATABASE_URL is required");
}
```

### IAM 権限の付与 (CDK で管理する場合)

```typescript
// Cloud Run サービスアカウントに Secret Manager アクセスを許可
// Claude Code に "Cloud Run が Secret Manager のシークレットを読める IAM 設定を CDK で書いて" と依頼

import * as gcp from "@cdktf/provider-google"; // Terraform CDK for GCP

const serviceAccount = new gcp.serviceAccount.ServiceAccount(this, "CloudRunSA", {
  accountId: "cloud-run-sa",
  displayName: "Cloud Run Service Account",
});

// シークレットアクセス権限
new gcp.projectIamMember.ProjectIamMember(this, "SecretAccessor", {
  project: projectId,
  role: "roles/secretmanager.secretAccessor",
  member: `serviceAccount:${serviceAccount.email}`,
});
```

---

## Step 5: CI/CD パイプライン (GitHub Actions)

### Claude Code へのプロンプト

```
GitHub Actions で Cloud Run への CI/CD パイプラインを構築して。
- トリガー: main ブランチへの push
- テスト実行 → Docker ビルド → Artifact Registry プッシュ → Cloud Run デプロイ
- Workload Identity Federation を使った認証 (サービスアカウントキーなし)
- ビルドキャッシュの活用
```

### 生成される GitHub Actions ワークフロー

```yaml
# .github/workflows/deploy.yml
name: Deploy to Cloud Run

on:
  push:
    branches: [main]

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  REGION: asia-northeast1
  SERVICE_NAME: my-api
  IMAGE: asia-northeast1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/app-images/my-api

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci
      - run: npm test
      - run: npm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write # Workload Identity Federation に必要

    steps:
      - uses: actions/checkout@v4

      # Workload Identity Federation で認証 (キーファイル不要)
      - id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
          service_account: ${{ secrets.WIF_SERVICE_ACCOUNT }}

      - uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker
        run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev --quiet

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE }}:latest
            ${{ env.IMAGE }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy ${{ env.SERVICE_NAME }} \
            --image "${{ env.IMAGE }}:${{ github.sha }}" \
            --region ${{ env.REGION }} \
            --platform managed \
            --quiet

      - name: Get Service URL
        run: |
          URL=$(gcloud run services describe ${{ env.SERVICE_NAME }} \
            --region ${{ env.REGION }} \
            --format 'value(status.url)')
          echo "Deployed to: $URL"
          echo "SERVICE_URL=$URL" >> $GITHUB_SUMMARY
```

### Workload Identity Federation の設定 (初回のみ)

```bash
# サービスアカウント作成
gcloud iam service-accounts create github-actions \
  --display-name="GitHub Actions"

# 必要な権限を付与
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:github-actions@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:github-actions@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

# Workload Identity Pool の作成
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --display-name="GitHub Actions Pool"

# GitHub の OIDC プロバイダーを追加
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# サービスアカウントへのバインディング
REPO="your-org/your-repo"  # GitHub リポジトリ名
gcloud iam service-accounts add-iam-policy-binding \
  github-actions@${PROJECT_ID}.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-pool/attribute.repository/${REPO}"
```

---

## Step 6: カスタムドメイン設定

### ドメインマッピングの設定

```bash
# ドメインの所有確認が済んでいる前提
gcloud run domain-mappings create \
  --service my-api \
  --domain api.example.com \
  --region asia-northeast1
```

Cloud Run が表示する DNS レコード (`CNAME: ghs.googlehosted.com.`) を DNS プロバイダーに設定すると、SSL 証明書が自動でプロビジョニングされます。

### Cloud Load Balancing を使った高度な設定

カスタムドメインで複数の Cloud Run サービスをルーティングしたい場合:

```yaml
# Claude Code に "Cloud Load Balancer でパスベースルーティングを設定して" と依頼
# 生成されるサーバーレス NEG の Terraform 設定 (抜粋)

resource "google_compute_region_network_endpoint_group" "api_neg" {
  name                  = "api-neg"
  network_endpoint_type = "SERVERLESS"
  region                = "asia-northeast1"

  cloud_run {
    service = google_cloud_run_service.my_api.name
  }
}

resource "google_compute_backend_service" "api_backend" {
  name        = "api-backend"
  protocol    = "HTTP"
  timeout_sec = 60

  backend {
    group = google_compute_region_network_endpoint_group.api_neg.id
  }
}
```

---

## 落とし穴3選

### 落とし穴1: コールドスタートで最初のリクエストが遅い

**症状**: デプロイ後、最初のリクエストだけ 2〜5秒かかる。

**原因**: ゼロスケールの場合、インスタンスの起動時間がかかります。

**解決策**:

```bash
# 方法1: min-instances を 1 以上に設定 (最もシンプル、少しコストがかかる)
gcloud run services update my-api --min-instances 1

# 方法2: --cpu-boost で起動時 CPU を倍増
gcloud run services update my-api --cpu-boost

# 方法3: startup probe の設定 (コンテナ起動後すぐにトラフィックを受け付けない)
# Cloud Run コンソールで startup probe のパスと設定を行う
```

**コスト比較 (東京リージョン、512Mi メモリ)**:
- `min-instances 0`: ゼロコスト (非アクティブ時)
- `min-instances 1`: ~$4-5/月 (常時1インスタンス)

開発環境は 0、本番はレイテンシ要件に応じて 1 以上を推奨します。

### 落とし穴2: 環境変数に機密情報を直接書いてしまう

**症状**: デプロイコマンドに `--set-env-vars "DATABASE_PASSWORD=xxxx"` と書いてしまう。

**問題**: Cloud Run コンソールのログやデプロイ履歴に平文で残ります。

```bash
# ❌ 絶対にやってはいけない
gcloud run deploy my-api --set-env-vars "DB_PASSWORD=supersecret123"

# ✅ Secret Manager を使う
gcloud run deploy my-api --set-secrets "DB_PASSWORD=db-password:latest"
```

**Claude Code への確認プロンプト**: 「このデプロイコマンドにセキュリティ上の問題はあるか確認して」

### 落とし穴3: SIGTERM を無視したコンテナがデプロイ時にリクエストを落とす

**症状**: デプロイ中に一部のリクエストが `503` になる。

**原因**: Cloud Run は新旧インスタンスを切り替える際に SIGTERM を送りますが、コンテナがこれを無視すると強制終了されます。

```typescript
// ✅ グレースフルシャットダウンを実装する
const server = app.listen(PORT);

// Cloud Run が送る SIGTERM を受け取ったら、
// 処理中のリクエストが完了するのを待ってからシャットダウン
process.on("SIGTERM", () => {
  console.log("SIGTERM received");
  // 新しいリクエストの受付を停止
  server.close(() => {
    // 接続が全てクローズされたら終了
    console.log("All connections closed, exiting");
    process.exit(0);
  });

  // タイムアウト (30秒以内にシャットダウンできなければ強制終了)
  setTimeout(() => {
    console.error("Forceful shutdown after timeout");
    process.exit(1);
  }, 30000);
});
```

---

## まとめ

| 機能 | 設定方法 | Claude Code の活用 |
|------|---------|-------------------|
| 初回デプロイ | `gcloud run deploy` | Dockerfile + gcloudコマンド一式を生成 |
| 自動スケーリング | `--min/max-instances` | ユースケースに合わせたパラメータを提案 |
| シークレット管理 | Secret Manager + `--set-secrets` | IAM設定から利用コードまで生成 |
| CI/CD | GitHub Actions | WIF認証付きワークフローを生成 |
| カスタムドメイン | Domain Mapping / LB | DNS設定手順も含めて説明 |

Cloud Run の最大の魅力は「インフラを意識せずにコンテナを動かせること」です。Claude Code が生成した Dockerfile をほぼそのままデプロイできる体験は、AWS Lambda + ZIP デプロイとは別次元の快適さがあります。

**まず試すなら**: ローカルで動いている Docker コンテナを、そのまま `gcloud run deploy --source .` でデプロイするところから始めてみてください。ソースコードを渡すだけで Cloud Build が自動でビルド・デプロイしてくれます。

---

*この記事の内容を claudecode-lab.com のバックエンド処理に適用した結果: Lambda から Cloud Run に移行後、デプロイ頻度が週1回 → 毎日に上がりました。コンテナ = 再現性の高い環境という安心感が、デプロイへの心理的ハードルを下げてくれています。*

---

関連記事: [Claude Code × Docker 開発環境](https://zenn.dev/masa_claudecodelab/articles/claude-code-docker-guide) | [Claude Code × CI/CD 自動化](https://zenn.dev/masa_claudecodelab/articles/claude-code-ci-cd-automation)
