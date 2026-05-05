# 回答ファイル: 第4章 設定・環境ファイル確認

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第4章 設定・環境ファイル確認(3.1)
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| D-1-01 | Critical | 3.1 | `frontend/` | `.env.local.example` | プレースホルダ値が実値を含まないか。全キー名と値のパターン | 全文(10行)を読み取り。記載キーは2つ:<br>(a) `DATABASE_URL="postgresql://USER:PASSWORD@localhost:5432/arte_db?schema=public"` (プレースホルダ)<br>(b) `STAFF_PASSWORD="your-staff-password-here"` (プレースホルダ)<br>**実値は混入していない**(`USER:PASSWORD`, `your-staff-password-here` 共に明確なプレースホルダ文字列)。 | 部分NG | **判定根拠**: 実値混入はないため Critical の本質的問題はない(本来 OK)。ただし以下の **大きな乖離あり**:<br>- 現実装は SQLite (`file:./dev.db`、`prisma.config.ts` 経由)であり、`DATABASE_URL=postgresql://...` の例は**実装と完全に乖離**(古い情報の残存)。<br>- `STAFF_PASSWORD` は既存レポート §10.3 で「未使用」と指摘済みの古いキー(現在の実装は iron-session + User テーブル)。<br>- 現実装で必要な `ENCRYPTION_KEY` / `SESSION_SECRET` / `SEED_ADMIN_PASSWORD` 等の必須シークレットが **example に1つも記載されていない** → 新規開発者がローカル環境構築時の必須設定を例示できていない。<br>- 既存レポート §10.3 で LOW 指摘されており、本作業時点でも未修正。<br>- 引用元: [.env.local.example:1-10](frontend/.env.local.example) 全文。 |
| D-2-01 | High | 3.1 / 4.C.7 | `frontend/` | `next.config.ts` | `experimental.serverActions.allowedOrigins`、`bodySizeLimit`、`encryptionKey`、`experimental.ppr`、`images.remotePatterns`、`poweredByHeader` の設定値 | (a) `experimental.serverActions.allowedOrigins`: **未設定**(allowedOrigins キーなし)<br>(b) `bodySizeLimit`: `'10mb'` 設定済み [next.config.ts:60](frontend/next.config.ts#L60)<br>(c) `encryptionKey` (Server Actions encryption key): **未設定**(Next.js 16 のデフォルト挙動で生成されるが、ビルド毎に変わる可能性あり)<br>(d) `experimental.ppr`: **未設定**(PPR 無効)<br>(e) `images.remotePatterns`: **未設定**。`images` 配下は `formats`/`deviceSizes`/`imageSizes` のみ [next.config.ts:63-67](frontend/next.config.ts#L63-L67)<br>(f) `poweredByHeader: false` 設定済み [next.config.ts:57](frontend/next.config.ts#L57) | 部分NG | **判定根拠**:<br>- **OK**: `poweredByHeader: false` ✅、`bodySizeLimit: '10mb'` (画像アップロード用途)、`images.remotePatterns` 未設定=外部画像許可なし(安全側)、`experimental.ppr` 未設定=機能無効。<br>- **NG寄り**: `experimental.serverActions.allowedOrigins` **未設定** → CSRF対策をフレームワークデフォルトに依存(同一オリジン判定はホストヘッダ/Origin等から実施だが、リバースプロキシ越しでは設定明示が望ましい)。**B-9-01 と連動**(下記参照)。<br>- **NG寄り**: `encryptionKey` **未設定** → Next.js は内部で生成(ビルドごとに変わる/プロセス再起動で変わる可能性)。Server Actions の暗号化された ID にビルド間互換性が必要なら明示設定が必要。実害は限定的。<br>- 加えて、本ファイルの `headers()` 実装は CSP / HSTS / X-Frame-Options / X-Content-Type-Options / Referrer-Policy / Permissions-Policy が公開・staff・/uploads で適切に分岐 → 既存レポート §11/§12 で「修正済み」確認、現コードと一致。<br>- 関連: B-9-01 (allowedOrigins) / B-11-01 (CSP) / 既存レポート §6.4 (X-Powered-By) / §12 (HTTPセキュリティヘッダー)。 |
| D-3-01 | High | 3.1 | `docker/` | `docker-compose.prod.yml` | 環境変数注入(env_file/environment)、secrets、ネットワーク分離、cap_drop/security_opt、ports/expose、read_only、restart | 全文(13行)読み取り。設定内容:<br>- `image: arte-frontend-prod`<br>- `build: { context: ../frontend, dockerfile: docker/Dockerfile.prod }`<br>- `container_name: arte-frontend-prod`<br>- `ports: - "3000:3000"` (全インターフェース)<br>- `restart: unless-stopped`<br>- `environment: - NODE_ENV=production`<br><br>未設定項目:<br>- `env_file`: **未設定**<br>- `secrets`: **未設定**<br>- `networks`: **未設定**(default network)<br>- `cap_drop` / `security_opt`: **未設定**<br>- `read_only`: **未設定**(false)<br>- `expose`: 未使用<br>- ボリュームマウント: なし(DB永続化未定義) | NG | **判定根拠**: 本ファイルは **本番運用には大幅に不足** している:<br>- ENCRYPTION_KEY / SESSION_SECRET / SEED_ADMIN_PASSWORD 等 **シークレット注入経路が未定義** → 現状ではコンテナ起動時に環境変数が無く、起動失敗するか Next.js デフォルトで動作する状態。<br>- DB ファイル(SQLite)の永続化ボリューム未定義 → コンテナ削除でデータ消失。<br>- `ports: "3000:3000"` は全インターフェースバインド(既存レポート §19 LOW で指摘済み)。本番ではリバースプロキシ前提なら `127.0.0.1:3000:3000` が望ましい。<br>- `cap_drop: [ALL]` / `security_opt: [no-new-privileges:true]` / `read_only: true` 等の防御層が一切ない(既存レポート §16 MEDIUM で指摘済み)。<br>- リソース制限(`deploy.resources.limits`)なし(既存レポート §16 で指摘済み)。<br>- 引用元: [docker-compose.prod.yml:1-13](docker/docker-compose.prod.yml) 全文。<br>- 関連: 既存レポート §16 / §19、C-1-01 / C-2-01 (本番環境依存)、3章全般。 |
| D-3-02 | High | 3.1 | `frontend/docker/` | `Dockerfile.prod` | ベースイメージdigest固定、非rootユーザー(nextjs UID 1001)、multi-stage、NODE_OPTIONS、init、不要パッケージ | 全文(35行)読み取り:<br>(a) **ベースイメージ**: `node:20-alpine` (digest固定なし。タグのみ)<br>(b) **非rootユーザー**: ✅ 実装あり [Dockerfile.prod:23-30](frontend/docker/Dockerfile.prod#L23-L30)<br>```dockerfile<br>RUN addgroup --system --gid 1001 nodejs && \\<br>    adduser --system --uid 1001 nextjs<br>USER nextjs<br>```<br>(c) **multi-stage**: ✅ 3段階(deps / builder / runner) [Dockerfile.prod:1,8,17](frontend/docker/Dockerfile.prod)<br>(d) **NODE_OPTIONS**: 未設定<br>(e) **init**: 未設定(Dockerfile / compose とも)<br>(f) **不要パッケージ**: alpine + standalone build のため runner stage は最小構成。`pnpm install` は builder stage のみ、runner stage は `.next/standalone` と `.next/static` と `public` のみコピー。 | 部分NG | **判定根拠**:<br>- **OK**: 非rootユーザー (UID 1001 nextjs) ✅、multi-stage ✅、standalone build による runner 最小化 ✅、`NEXT_TELEMETRY_DISABLED=1` ✅。<br>- **NG寄り**: ベースイメージ **digest固定なし** → 同一タグでもイメージが変動する可能性(サプライチェーン)。`FROM node:20-alpine@sha256:...` 形式が望ましい。<br>- **NG寄り**: `init` 未設定 → コンテナ内 PID 1 のシグナル処理・ゾンビプロセス回収が node プロセスに集約。長期運用ではゾンビプロセスのリスク。<br>- 既存レポート §16 で「rootユーザーで実行」と指摘されているが、これは `docker-compose.dev.yml` 由来(volumes マウントで /app 全体が書き込み可能)。本番 Dockerfile は非rootユーザー化済みで現状一致。<br>- 関連: D-3-01 (compose 側の不足)、既存レポート §16。 |
| D-4-01 | High | 3.1 | `frontend/` | `.dockerignore` | `.env*`、node_modules、.git、`*.backup`、`dev.db*`(WAL/journal)の除外状況 | 全文(29行)読み取り。除外設定済み:<br>- `node_modules`、`.next`、`.git`、`.gitignore`、`README.md`、`Dockerfile*`、`.dockerignore`、`dist`、`build`、`out`、`coverage`、`.DS_Store`、`*.pem`、`*.md`、`.pnpm-store`、`.vscode`、`.idea`、`*.swp/*.swo`、`Thumbs.db`、`.nyc_output`、`.backups`、`tsconfig.tsbuildinfo`、`docker/`<br>- ログ系: `npm-debug.log*`、`yarn-debug.log*`、`yarn-error.log*`、`.pnpm-debug.log*`<br>- `.env*.local` (除外: ✅)<br><br>**除外漏れ(NG)**:<br>- `.env` (拡張子なし) は除外**されない**<br>- `.env.production` 等 `*.local` 以外の `.env*` は除外**されない**<br>- `*.backup` ファイル(`Footer.tsx.20260212` 等の日付サフィックス含む)は除外**されない**(`.backups` ディレクトリ単体のみ)<br>- `dev.db`、`dev.db-journal`、`dev.db-wal`、`dev.db-shm` は除外**されない** | NG | **判定根拠**: シークレットおよび機密データのビルドコンテキスト混入リスクあり:<br>- `.env` (拡張子なし)が混入していれば、Dockerfile が明示的にコピーしなくても `COPY . .` で含まれてしまう。本 Dockerfile は builder stage で `COPY . .` あり [Dockerfile.prod:13](frontend/docker/Dockerfile.prod#L13)。<br>- `dev.db*` が `frontend/` 直下にある状態でビルドすると、本番イメージに開発DB(認証情報・監査ログ・テストデータ)が**そのまま混入**するリスク → A-1-08 (dev.db) と直結。<br>- `*.backup` 残存ファイル(`Footer.tsx.20260212` 等、A-1-04/A-1-05/A-1-06 に該当)もビルドコンテキストに含まれる。<br>- 既存レポートでは未指摘の項目。本作業で新規発見。<br>- 引用元: [.dockerignore:6-29](frontend/.dockerignore) 全文。<br>- 関連: A-1-03 (.gitignore) / A-1-08 (dev.db) / J-1-01 (バックアップファイル)。 |
| D-5-01 | High | 3.1 / 4.C.35 | `frontend/` | `package.json`, `pnpm-lock.yaml` | 既知CVE、`pnpm.onlyBuiltDependencies`、postinstallスクリプトの有無 | (a) **既知CVE**: `pnpm audit` を本セッションで実行していないため、**判定不能**(動的検証で要確認)。依存パッケージリストは下記参照。<br>(b) **pnpm.onlyBuiltDependencies**: 設定済み [package.json:24-32](frontend/package.json#L24-L32)<br>```json<br>"pnpm": { "onlyBuiltDependencies": ["@prisma/engines", "better-sqlite3", "prisma", "esbuild", "sharp"] }<br>```<br>(c) **postinstall**: `scripts` セクションに `dev` / `build` / `start` / `lint` のみ。**postinstall スクリプトは存在しない**。<br><br>**主要依存パッケージ(リスク観点)**:<br>- `next: 16.1.6` (既存レポート §15 で 16.1.2→16.1.6 に修正済み)<br>- `@prisma/client: ^7.4.0`、`prisma: ^7.4.0`<br>- `bcryptjs: ^3.0.3`<br>- `iron-session: ^8.0.4`<br>- `otplib: ^13.3.0`<br>- `sharp: ^0.34.5`<br>- `react: 19.2.3` / `react-dom: 19.2.3`<br>- `eslint-config-next: 16.1.2`(next 本体は 16.1.6 で不一致) | 部分NG | **判定根拠**:<br>- **OK**: postinstall スクリプトなし → サプライチェーン経由の任意コード実行リスク低。`onlyBuiltDependencies` 設定済みで未承認パッケージのビルドフックが抑止される。<br>- **NG寄り**: `pnpm audit` 未実施のため CVE は判定不能。動的検証で確認必須。<br>- **軽微な不整合**: `next: 16.1.6` に対し `eslint-config-next: 16.1.2` で **0.4 マイナーバージョン不一致**(機能差異の可能性、セキュリティリスクは限定的)。<br>- 引用元: [package.json:1-49](frontend/package.json) 全文。<br>- 関連: 既存レポート §15 / 12章 (技術スタック)。 |
| D-6-01 | Medium | 3.1 | `frontend/` | `eslint.config.mjs`, `tsconfig.json` | セキュリティ関連ESLintルール、`tsconfig` の `strict`/`noImplicitAny`/`strictNullChecks` | (a) **eslint.config.mjs** [eslint.config.mjs:1-19](frontend/eslint.config.mjs):<br>- `eslint-config-next/core-web-vitals` と `eslint-config-next/typescript` を継承<br>- `globalIgnores` で `.next/**`、`out/**`、`build/**`、`next-env.d.ts` を除外<br>- **`no-eval`、`no-implied-eval`、`no-new-func`、`security/*` 等のセキュリティ専用ルールの明示的有効化なし**。`eslint-plugin-security` の導入なし。<br>(b) **tsconfig.json** [tsconfig.json:1-34](frontend/tsconfig.json):<br>- `"strict": true` ✅(これにより `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `useUnknownInCatchVariables`, `alwaysStrict` がまとめて有効)<br>- `"target": "ES2017"`、`"skipLibCheck": true`、`"isolatedModules": true`、`"esModuleInterop": true`<br>- `"noUncheckedIndexedAccess"` / `"noImplicitOverride"` / `"noFallthroughCasesInSwitch"` 等の追加 strict オプションは**設定なし** | 部分NG | **判定根拠**:<br>- **OK**: tsconfig `strict: true` で TypeScript の型安全性は最低限担保 ✅。<br>- **NG寄り**: ESLint にセキュリティ専用ルールの明示なし。`eslint-plugin-security` または `eslint-config-next` 内蔵の eval禁止系ルールへの依存度が確認できない。`as any` / `@ts-ignore` の検出も明示なし(K-2-06 で grep 確認予定)。<br>- **軽微な改善余地**: `noUncheckedIndexedAccess` や `noFallthroughCasesInSwitch` の追加で堅牢性向上余地あり(セキュリティ直接ではないが、ロジック誤りによる脆弱性予防)。<br>- 関連: K-2-06 (`as any`/`@ts-ignore` ヒット数)。 |

---

## 章サマリ

- **回答済項目数:** 7件
  - Critical: 1件
  - High: 5件
  - Medium: 1件
  - Low: 0件
- **OK 件数:** 0件
- **NG 件数:** 2件 (D-3-01, D-4-01)
- **部分NG 件数:** 5件 (D-1-01, D-2-01, D-3-02, D-5-01, D-6-01)
- **判定不能 件数:** 0件 (D-5-01 の CVE 部分は動的検証マターのため部分NG扱い)
- **判定不能の主な理由:** 該当なし

## 参照したファイル一覧

- `frontend/.env.local.example` (全10行)
- `frontend/next.config.ts` (全112行)
- `frontend/.dockerignore` (全29行)
- `frontend/eslint.config.mjs` (全19行)
- `frontend/tsconfig.json` (全34行)
- `frontend/package.json` (全49行)
- `frontend/docker/Dockerfile.prod` (全35行)
- `docker/docker-compose.prod.yml` (全13行)
- `docker/docker-compose.dev.yml` (全21行、参考)

## 実行した grep / コマンド一覧

- `ls -la /srv/git/arte-web-site-project/frontend/` (主要ファイル存在確認)
- `ls -la /srv/git/arte-web-site-project/frontend/docker/`
- `ls -la /srv/git/arte-web-site-project/docker/`

## 章をまたぐ関連項目

- D-1-01 ⇔ 既存レポート §10.3 (古い `STAFF_PASSWORD` の残存)
- D-2-01 ⇔ B-9-01 / B-11-01 (allowedOrigins / CSP)、既存レポート §6.4 / §12
- D-3-01 ⇔ 既存レポート §16 / §19、C-1-01 / C-2-01 (3章本番環境依存)
- D-3-02 ⇔ 既存レポート §16
- D-4-01 ⇔ A-1-03 / A-1-08 / J-1-01(バックアップファイル・dev.db のビルドコンテキスト混入リスク)
- D-5-01 ⇔ 既存レポート §15、12章(技術スタック)
- D-6-01 ⇔ K-2-06 (`as any`/`@ts-ignore` 検出)
