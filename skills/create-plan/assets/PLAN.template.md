# PLAN

## 目的
- (1〜2 行で達成内容)

## 前提
- Repo: Fastify API + GCP インフラ（Terraform は infra/ 配下）
- 環境:
  - staging:
  - production:
- 制約:
  - downtime:
  - data migration:
  - cost/security/compliance:

## 影響範囲
- Fastify:
  - (routes/plugins/auth/config/env)
- Prisma/DB:
  - (schema, migrations, seed)
- TypeSpec/OpenAPI:
  - (src/typespec/, src/swagger/)
- Terraform:
  - (infra/modules, infra/environments, state/backend)
- GCP services:
  - (Cloud Run / Cloud SQL / VPC / IAM / Secret Manager / LB / Scheduler / PubSub / etc.)
- CI/CD:
  - (GitHub Actions / Cloud Build / deploy steps)

## 前提条件 / 未確定事項
- (仮定を列挙; 必要な場合のみ質問を追加)

## 計画（手順）
### 0) 事前チェック（安全）
1. 対象環境とプロジェクト ID を確認する。
2. 現在の状態を把握する:
   - Terraform の backend/state と workspace
   - 現在のデプロイ済みリビジョン（Cloud Run）/ DB バージョン（必要なら）
3. シークレットと環境変数の管理元を確認する（Secret Manager、CI 変数）。

### 1) アプリ + データ変更（ローカル）
1. Fastify の変更を実装する:
   - (routes/handlers/plugins/middleware/config)
2. env schema を更新する:
   - (新規 env 変数、デフォルト、バリデーション)
3. API 変更がある場合は TypeSpec/OpenAPI を更新:
   - `src/typespec/` を編集し `pnpm generate:openapi` を実行
4. DB 変更がある場合は Prisma を更新:
   - `pnpm migrate:generate --name=<topic>`
   - `pnpm migrate:deploy`
   - `pnpm seed:run`（seed/master data に影響がある場合）

### 2) 検証（ローカル）
1. Lint/format/typecheck:
   - `pnpm lint`
   - `pnpm format`（フォーマット変更が意図されている場合のみ）
   - `pnpm type:check`
2. テスト:
   - `pnpm test:unit`
   - `pnpm test:integration`
   - `pnpm test:all`（全体実行が必要な場合）
3. ビルド:
   - `pnpm build`

### 3) Terraform 変更（infra/）
1. モジュール/リソースを更新・追加する:
   - (変更内容)
2. 変数と環境の紐づけを更新する:
   - (tfvars / locals / outputs)
3. 実行:
   - `terraform fmt -recursive`
   - `terraform validate`
   - `terraform plan`（環境ごとのコマンドを明記）
4. plan の確認ポイント:
   - replacements/destroys
   - IAM の広範化
   - network / egress の変更
   - コストに影響するリソース
5. apply 方針:
   - staging から適用
   - 影響範囲が大きい場合は段階適用（理由を明記）

### 4) デプロイ + 連携（infra <-> app）
1. デプロイ方針:
   - staging デプロイ
   - smoke test
   - production ロールアウト（必要ならトラフィックスプリット）
2. アプリ権限の確認:
   - service account roles
   - Secret Manager アクセス
3. ネットワーク整合性の確認:
   - Cloud Run ingress/egress, VPC connector（必要なら）
   - Cloud SQL 接続（private IP / connector / auth proxy）
4. 監視・可観測性の確認:
   - logs/metrics/traces 設定
   - error reporting/Sentry（存在する場合）

## 検証
### Terraform
- `terraform fmt -recursive`
- `terraform validate`
- `terraform plan` (staging)
- `terraform apply` (staging)
- Re-run `terraform plan` to confirm no drift (optional)

### App (Fastify)
- Unit:
  - `pnpm test:unit`
- Integration:
  - `pnpm test:integration`
- Lint/Typecheck:
  - `pnpm lint`
  - `pnpm type:check`
- Build:
  - `pnpm build`
- Smoke:
  - Health check endpoint:
  - Critical API call:
  - Auth flow (if any):

### GCP 稼働時チェック
- Cloud Run:
  - revision healthy / error rate OK / latency OK
- IAM:
  - least privilege preserved
- Secrets:
  - mounted/resolved correctly
- DB:
  - connectivity OK / migrations OK (if any)

## ロールアウト
- staging 先行、次に production
- monitoring window:
- traffic strategy:
  - (all-at-once / gradual / canary)

## ロールバック
- Terraform:
  - revert commit + `terraform apply`
  - note: replacements がある場合は復旧手順を記載
- Cloud Run:
  - 直前のリビジョンに戻す
- DB:
  - (migrations がある場合: forward-only? down migration plan?)

## 補足
- Risks:
- Cost impact:
- Security considerations:
