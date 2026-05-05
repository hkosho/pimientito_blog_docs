# 回答ファイル: 第12章 追加の技術スタック固有観点

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第12章 追加の技術スタック固有観点(4.C補完)
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| L-1-01 | Medium | 4.C.18 | `frontend/prisma/`, `src/lib/`, `src/app/staff/`, `frontend/scripts/` | `seed.ts`, `auth.ts`, 各 actions.ts | bcryptjs の `saltRounds` 値。全呼出箇所で一致しているか | `bcrypt.hash` および `BCRYPT_ROUNDS` の grep 結果(全 8 箇所):<br>1. `src/app/staff/settings/actions.ts:13,163` — `BCRYPT_ROUNDS = 12`、`bcrypt.hash(newPassword, BCRYPT_ROUNDS)`<br>2. `src/app/staff/login/actions.ts:27,30,262` — `BCRYPT_ROUNDS = 12`、`bcrypt.hash('dummy-timing-equalization', BCRYPT_ROUNDS)`(ダミーハッシュ)、`bcrypt.hash(newPassword, BCRYPT_ROUNDS)`<br>3. `src/app/staff/users/actions.ts:10,64,135` — `BCRYPT_ROUNDS = 12`、`bcrypt.hash(initialPassword/tempPassword, BCRYPT_ROUNDS)`<br>4. `src/lib/recovery-codes.ts:4,18` — `BCRYPT_ROUNDS = 12`、`bcrypt.hash(code, BCRYPT_ROUNDS)`<br>5. `frontend/scripts/reset-password.ts:21` — `bcrypt.hash(tempPassword, 12)` (リテラル直書き)<br>6. `frontend/prisma/seed.ts:30` — `bcrypt.hash(seedPassword, 12)` (リテラル直書き) | 部分NG | **判定根拠**:<br>- **OK**: 全 8 箇所すべて **rounds = 12** で**統一** ✅。NIST SP 800-63 / OWASP の最低推奨(10〜12 rounds)を満たす ✅<br>- **NG寄り(軽微)**: `scripts/reset-password.ts` と `prisma/seed.ts` のみ **`12` リテラル直書き** で、残り 4 ファイルは `BCRYPT_ROUNDS` 定数経由 → 将来 rounds を変更する際に同期忘れリスクあり<br>- **改善提案**: 共通モジュール(例: `src/lib/auth-config.ts`)に `export const BCRYPT_ROUNDS = 12` を集約し、scripts/seed からも import<br>- 関連: I-1-01(reset-password.ts)、E-2-01(seed.ts)、F-6-01。 |
| L-2-01 | Medium | 4.C.19 | `frontend/src/lib/`, `src/app/staff/login/`, `src/app/staff/settings/` | `auth.ts`, `actions.ts` 等 | otplib の設定値(`window`, `algorithm`, `digits`, `step`) | otplib の使用箇所:<br>- `src/app/staff/login/actions.ts:6,208,278,324,402` — `import { generateSecret, generateURI, verifySync } from 'otplib'`<br>- `src/app/staff/settings/actions.ts:4,40,65` — 同上<br>**設定の明示的指定**:<br>- `generateSecret()`: 引数なし、デフォルト挙動<br>- `generateURI({ secret, issuer: '有限会社アルテ', accountName: user.username })`: issuer / accountName のみ指定<br>- `verifySync({ token, secret })`: 引数 token/secret のみ、`window` 等の上書きなし<br>**otplib v13 のデフォルト値(ドキュメント上)**:<br>- `algorithm`: `'sha1'`(RFC 6238 標準)<br>- `digits`: 6<br>- `step`: 30 秒<br>- `window`: 0(現在の時間枠のみ許容、過去/未来のコードは許可しない=最も厳格)<br>**プロジェクト内に otplib の設定上書き(`authenticator.options = ...` 等)は grep ヒット 0件** | 部分NG | **判定根拠**:<br>- **OK寄り**: デフォルト値は適切(window=0 は最厳格、time drift 許容なしで TOTP 本来の安全性確保)✅<br>- **NG寄り**: 設定値が **明示されていない**(ライブラリ更新時のデフォルト変更リスク)→ コード上で `authenticator.options = { window: 0, ... }` と明示するのが望ましい<br>- 既存レポート §21 で「TOTP 仕様」として記載されているが、本作業時点の設定値の明示なしを補完。<br>- 関連: F-5-01、G-2-01、B-5-03(定数時間比較)。 |
| L-3-01 | High | 4.C.24 | `frontend/src/app/` | `**/route.ts` | `route.ts` ファイル(Route Handler)の存在確認と、各ファイルの認証チェック | `find src/app -name "route.ts"` の結果 **0件** ✅<br>**Route Handler は1つも存在しない** → 全 API 操作は Server Actions 経由 | OK | **判定根拠**: Server Actions のみの API 構造のため、Route Handler 経由の認証漏れ脆弱性は**構造的に存在しない** ✅。<br>関連: F-1-01、F-4-01(middleware で /staff/* 保護)、第7章 G 系全件。 |
| L-4-01 | Medium | 4.C.26 | `frontend/src/lib/` | `auth.ts`, `session.ts` | React 19 taint API(`taintUniqueValue`, `taintObjectReference`)の使用有無 | `grep -rEn "taintUniqueValue\|taintObjectReference" src/` の結果 **0件**<br>**React 19 taint API は使用されていない** | 部分NG | **判定根拠**:<br>- **保留(L-4-01 の備考に「未使用なら保留」)**: React 19 の taint API は機密値のクライアント送信を防ぐ追加防御層。**使用なし**だが、本リポジトリでは:<br>  - DTO (`safeUserSelect` / `safeAuditLogSelect`) で機密フィールドのクライアント送信を防止 ✅(F-8-01 / 既存§6.5)<br>  - `'use client'` ファイルでの `process.env.*` 0件(K-2-07)<br>  → **既存対策で実質的にカバー済み**<br>- ただし、Server Component でうっかり `passwordHash` 等を props で渡してしまった場合の追加防御として、taint API 導入が望ましい(本作業範囲では未確認、将来検討)<br>- 関連: F-8-01、既存§6.5。 |
| L-5-01 | Medium | 4.C.34 | `docs/` | `architecture/`, `process/`, `failures/`, `ai-sessions/` 配下 | 各ディレクトリのドキュメントとセキュリティ関連記述。運用手順・インシデント対応・secretsローテーション手順 | `ls docs/*/` の結果:<br>- `docs/ai-sessions/`: **空ディレクトリ**(ファイルなし)<br>- `docs/architecture/`: **空ディレクトリ**<br>- `docs/failures/`: **空ディレクトリ**<br>- `docs/process/`: **空ディレクトリ**<br>**全 4 ディレクトリが空、ドキュメントは1件も存在しない** | NG | **判定根拠**: ⚠️ **運用ドキュメント・インシデント対応・secrets ローテーション手順 すべて未記述**:<br>- 4 ディレクトリは空のまま、`docs/` 配下にドキュメントが存在しない<br>- 本番運用に必要な要素(復旧手順、secret ローテーション、インシデント対応フロー、AI セッション記録)が**全くない**<br>- 既存レポート §17 / §18 で「ログローテーション・保持ポリシー、DBバックアップ、リカバリ手順書 未作成」と指摘済 → 本作業時点でも未対応<br>- A-1-08 (dev.db履歴によるシークレット漏洩)に対し、ローテーション手順が未文書化な状態は実害大<br>- 関連: 既存§17/§18、A-1-08、C-1-01〜C-6-01(本番環境依存項目)、E-3-01。 |
| L-6-01 | Medium | 4.C.36 | (CI/CD) | - | コンテナイメージの脆弱性スキャン(Trivy等)が CI/CD パイプラインに組み込まれているか | `ls /srv/git/arte-web-site-project/.github/workflows/` → **`.github` ディレクトリ自体が存在しない**<br>その他の CI 設定ファイル: `.gitlab-ci.yml` / `.circleci/` / `azure-pipelines.yml` 等も grep で 0件<br>**CI/CD パイプライン自体が未構築** | 判定不能 | **判定根拠**: チェックリストの「タイプ: Env-dep / C」と一致 → 本来から「運用担当ヒアリング併用」のカテゴリ。<br>- リポジトリ内の証拠から CI/CD 未構築は確認できる(ファクト)<br>- ただし本番デプロイ運用が将来 CI/CD で実施されるか、手動デプロイか、その場合の Trivy 等の組み込み有無は仕様/運用の判断 → 動的検証/運用ヒアリングマター<br>- 関連: D-3-02(Dockerfile.prod の Trivy 推奨)、L-5-01。 |
| L-7-01 | Medium | 4.C.38 | `frontend/src/app/staff/audit-logs/`, `src/lib/` | `audit-log.ts`, `actions.ts` | 監査ログの削除・エクスポート機能の有無(WORM原則違反検出) | `grep -rEn "auditLog\.delete\|auditLog\.update\|auditLog\.deleteMany\|export.*audit\|delete.*audit" src/` の結果 **0件**<br>**(a)** `prisma.auditLog.delete*` / `prisma.auditLog.update` の **使用 0件** ✅<br>**(b)** AuditLog のエクスポート機能(CSV/JSON download等) **未実装** ✅<br>**(c)** `AuditLogsTable.tsx`(全230行)で確認: 削除ボタン・export 機能なし。脚注に「監査ログは削除できません(改ざん防止)」明記 [AuditLogsTable.tsx:225-227]<br>**(d)** 既存レポート §17 で「**削除API不在、改ざん防止策実装済み**」評価と一致 | OK | **判定根拠**: WORM 原則に従い、書き込み専用 + 閲覧のみ ✅。改ざん防止策は実装済み(既存§17、3回修正済み)。<br>**留意**: M-1-07 (仕様確認待ち)で「監査ログの削除・エクスポート機能の要否」となっており、将来エクスポート機能を追加する場合は機密フィールド除外などの設計が必要。<br>関連: M-1-07、F-7-01、既存§17。 |

---

## 章サマリ

- **回答済項目数:** 7件
  - Critical: 0件
  - High: 1件 (L-3-01)
  - Medium: 6件 (L-1-01, L-2-01, L-4-01, L-5-01, L-6-01, L-7-01)
  - Low: 0件
- **OK 件数:** 2件 (L-3-01, L-7-01)
- **NG 件数:** 1件 (L-5-01)
- **部分NG 件数:** 3件 (L-1-01, L-2-01, L-4-01)
- **判定不能 件数:** 1件 (L-6-01)
- **判定不能の主な理由:**
  - L-6-01: CI/CD 自体が未構築。本番運用での Trivy 等の組み込みは運用判断マター(Env-dep カテゴリ)

## 参照したファイル一覧

(他章で参照済みは省略)

- `docs/ai-sessions/` (空)、`docs/architecture/` (空)、`docs/failures/` (空)、`docs/process/` (空)
- (再利用) `frontend/scripts/reset-password.ts:21`、`frontend/prisma/seed.ts:30`
- (再利用) `frontend/src/app/staff/settings/actions.ts`、`login/actions.ts`、`users/actions.ts`、`src/lib/recovery-codes.ts`(BCRYPT_ROUNDS 確認)
- (再利用) `frontend/src/app/staff/audit-logs/AuditLogsTable.tsx`(L-7-01)

## 実行した grep / コマンド一覧

- `grep -rEn "bcrypt\.hash\|saltRounds\|BCRYPT_ROUNDS" src/ frontend/scripts/ frontend/prisma/` → **8 箇所すべて rounds=12**
- `find src/app -name "route.ts"` → **0件**(L-3-01 OK)
- `grep -rEn "taintUniqueValue\|taintObjectReference" src/` → **0件**(L-4-01)
- `grep -rEn "auditLog\.delete\|auditLog\.update" src/` → **0件**(L-7-01 OK)
- `ls docs/*/` → **全 4 ディレクトリ空**(L-5-01 NG)
- `ls /srv/git/arte-web-site-project/.github/workflows/` → **`.github` 不在**(L-6-01 判定不能)

## 章をまたぐ関連項目

- L-1-01 ⇔ I-1-01 / E-2-01 / F-6-01(bcrypt rounds 統一性)
- L-2-01 ⇔ B-5-03 / F-5-01 / G-2-01(TOTP 比較・rate-limit)
- L-3-01 ⇔ F-1-01 / F-4-01(認証経路、Route Handler 不在で middleware 漏れリスクなし)
- L-4-01 ⇔ F-8-01 / 既存§6.5(DTO による機密漏洩防止)
- L-5-01 ⇔ 既存§17/§18 / A-1-08 / C-1〜C-6 / E-3-01(運用ドキュメント全般)
- L-6-01 ⇔ D-3-02 / D-5-01(Dockerfile / pnpm audit)
- L-7-01 ⇔ M-1-07 / F-7-01 / 既存§17(改ざん防止)
