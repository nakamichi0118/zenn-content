---
title: "Claude Code × Docker 開発環境完全ガイド｜Dockerfile・compose・DevContainer を自動生成"
emoji: "🐳"
type: "tech"
topics: ["claudecode", "docker", "devcontainer", "typescript", "infrastructure"]
published: true
---

## はじめに

「Docker の設定ってなんか難しい」——Dockerfile や docker-compose.yml を一から書くのは慣れが必要ですが、Claude Code に任せれば **開発環境のコンテナ化を30分以内に完成**できます。

この記事では、Dockerfile 生成から docker-compose 設計、VS Code DevContainer 設定まで、実際に使えるコードと一緒に解説します。

## Dockerfile の自動生成

### Node.js / TypeScript プロジェクト

```bash
claude -p "
以下の条件で本番用の Dockerfile を生成して。

- Node.js 20 (Alpine ベース、イメージ軽量化)
- TypeScript プロジェクト (npm run build でビルド)
- マルチステージビルド (builder → runner)
- 非 root ユーザーで実行 (セキュリティ)
- ヘルスチェックエンドポイント /health を設定
- .dockerignore も作成
"
```

生成される Dockerfile:

```dockerfile
# --- Builder ステージ ---
FROM node:20-alpine AS builder
WORKDIR /app

# 依存関係を先にコピー (レイヤーキャッシュ活用)
COPY package*.json ./
RUN npm ci --frozen-lockfile

# ソースをコピーしてビルド
COPY . .
RUN npm run build

# 開発依存を除去
RUN npm prune --production

# --- Runner ステージ ---
FROM node:20-alpine AS runner
WORKDIR /app

# セキュリティ: 非 root ユーザー
RUN addgroup --system --gid 1001 nodejs \
  && adduser --system --uid 1001 --ingroup nodejs appuser

# ビルド成果物のみコピー
COPY --from=builder --chown=appuser:nodejs /app/dist ./dist
COPY --from=builder --chown=appuser:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:nodejs /app/package.json ./

USER appuser
EXPOSE 3000

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD wget -qO- http://localhost:3000/health || exit 1

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
```

マルチステージビルドのポイントは **builder と runner を分離してイメージサイズを最小化**すること。TypeScript のコンパイラや開発依存関係を runner に含めないだけで、イメージが 1/3 以下になります。

## docker-compose の設計

### フルスタック開発環境

```bash
claude -p "
以下の構成で docker-compose.yml (開発用) を生成して。

- app: Node.js TypeScript API (ホットリロード対応)
- db: PostgreSQL 16
- cache: Redis 7
- adminer: DB管理UI
- localstack: AWS サービスのローカルエミュレート (S3, SQS)

ボリュームマウントでソースコードの変更を即反映。
各サービスの依存関係 (depends_on + healthcheck) を正しく設定。
.env から環境変数を読み込む設定も含める。
"
```

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build:
      context: .
      target: builder  # 開発用はビルダーステージを使用
    volumes:
      - .:/app
      - /app/node_modules  # node_modules はコンテナ内を優先
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:${DB_PASSWORD}@db:5432/myapp
      - REDIS_URL=redis://cache:6379
    env_file:
      - .env
    command: npm run dev  # ts-node-dev でホットリロード
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  adminer:
    image: adminer:4
    ports:
      - "8080:8080"
    depends_on:
      - db

  localstack:
    image: localstack/localstack:3
    ports:
      - "4566:4566"
    environment:
      - SERVICES=s3,sqs,sns
      - AWS_DEFAULT_REGION=ap-northeast-1
    volumes:
      - localstack_data:/var/lib/localstack

volumes:
  postgres_data:
  redis_data:
  localstack_data:
```

## VS Code DevContainer の設定

DevContainer を使うと「コンテナ内でそのまま開発」できます。チームメンバー全員が同じ環境で作業できるので、「自分の環境では動くのに」問題がなくなります。

```bash
claude -p "
.devcontainer/devcontainer.json を生成して。

- ベースイメージ: node:20
- 自動インストール: Docker in Docker、GitHub CLI、AWS CLI
- VS Code 拡張機能: ESLint、Prettier、GitLens、Docker、REST Client
- postCreateCommand で npm install を自動実行
- ポートフォワーディング: 3000、5432、8080
- コンテナ内のターミナルをzshにする
"
```

```json
{
  "name": "Node.js Dev Container",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:1-20",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/aws-cli:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "eamodio.gitlens",
        "ms-azuretools.vscode-docker",
        "humao.rest-client"
      ],
      "settings": {
        "editor.formatOnSave": true,
        "terminal.integrated.defaultProfile.linux": "zsh"
      }
    }
  },
  "forwardPorts": [3000, 5432, 8080],
  "postCreateCommand": "npm install",
  "remoteUser": "node"
}
```

## CI/CD での Docker ビルド最適化

```bash
claude -p "
GitHub Actions で Docker イメージをビルド・プッシュする workflow を作成して。

- トリガー: main ブランチへの push
- レジストリ: Amazon ECR
- キャッシュ: GitHub Actions cache で BuildKit キャッシュを活用
- マルチプラットフォーム: linux/amd64 と linux/arm64
- タグ: Git SHA とlatest
"
```

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/myapp:latest
            ${{ steps.login-ecr.outputs.registry }}/myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## よくある落とし穴

**1. node_modules をコンテナとホストで共有してしまう**

```yaml
# ❌ これをやると node_modules が Linux/Mac で混在してエラー
volumes:
  - .:/app

# ✅ node_modules だけコンテナ内を使う
volumes:
  - .:/app
  - /app/node_modules  # 匿名ボリュームで上書き
```

**2. ビルドキャッシュを無効化するCOPYの順番**

```dockerfile
# ❌ ソースを先にコピーすると毎回全キャッシュが無効化
COPY . .
RUN npm install

# ✅ package.json を先にコピーしてキャッシュを活用
COPY package*.json ./
RUN npm install
COPY . .
```

**3. .env を Docker イメージに含めてしまう**

`.dockerignore` に `.env` を追加し忘れると、シークレットがイメージに焼き込まれます。`docker history` コマンドで簡単に取り出せてしまうので注意。

## まとめ

| タスク | Claude Code でできること |
|--------|----------------------|
| Dockerfile | マルチステージ・非rootユーザー・ヘルスチェック込みで生成 |
| docker-compose | 複数サービスの依存関係・ヘルスチェック・ボリューム設計 |
| DevContainer | チーム統一の開発環境を一発設定 |
| CI/CD | ECR プッシュ・マルチプラットフォームビルドを自動化 |

「Dockerが難しい」のは書き方を知らないだけで、Claude Code に「こういう環境が欲しい」と伝えれば30分以内に動く環境が揃います。まず `docker-compose.yml` の生成から試してみてください。

---

関連記事: [Claude Code × CI/CD 自動化](https://zenn.dev/masa_claudecodelab/articles/claude-code-ci-cd-automation) | [Claude Code 開発環境セットアップ](https://zenn.dev/masa_claudecodelab/articles/claude-code-dev-env-setup)
