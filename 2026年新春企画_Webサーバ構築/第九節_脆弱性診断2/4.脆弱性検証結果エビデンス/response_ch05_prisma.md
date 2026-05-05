# 回答ファイル: 第5章 データ層(Prisma)確認

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第5章 データ層(Prisma)確認(3.2)
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| E-1-01 | High | 3.2 | `frontend/prisma/` | `schema.prisma` | (a)機密フィールドの型と制約 (b)`oneTimePassword`等の保存形式 (c)`authorId` 相当の有無 (d)`@unique`/`@default` の妥当性 | 第1章 A-1-08 で全文引用済み。要約:<br><br>(a) **User モデルの機密フィールド** [schema.prisma:57-80]:<br>- `passwordHash String` NOT NULL (bcrypt 12 rounds、B-14-01 / G-1-01 で確認)<br>- `totpSecret String?` (NULL 許容、AES-256-GCM 暗号化、`crypto.ts` 経由)<br>- `recoveryCodesHash String?` (JSON 配列の bcrypt ハッシュ、F-6-01)<br>- `oneTimePassword String?` (bcrypt ハッシュ、B-14-01 で確認)<br>- `otpExpiresAt DateTime?`<br>(b) **保存形式**: `oneTimePassword` および `recoveryCodesHash` は **bcrypt ハッシュ**(平文ではない)✅。`totpSecret` は **AES-256-GCM 暗号化**(`crypto.ts` 経由、F-3-01)<br>(c) **`authorId` フィールドの有無**: ❌ **存在しない**。Blog/News/Category モデルに `authorId` 相当の参照フィールドなし → 「staff全員が全記事編集可能」設計(M-1-01 仕様待ち)<br>(d) **`@unique` / `@default`**: <br>- `username @unique` ✅、`email @unique` ✅<br>- `role @default("staff")` ✅(最小権限)<br>- `mustChangePassword @default(true)` ✅(初回PW変更強制)<br>- `twoFactorEnabled @default(false)`、`isLocked @default(false)`、`loginAttempts @default(0)` ✅<br>- AuditLog: `ipAddress String?`、`userAgent String?` で**長さ制限なし** ⚠️(F-7-01 と直結) | 部分NG | **判定根拠**:<br>- **OK**: 機密フィールドの保存形式(bcrypt / AES-256-GCM)は適切 ✅、@unique / @default 制約も妥当 ✅<br>- **NG寄り(設計)**: `authorId` 相当の所有者フィールドが Blog/News に存在しない → IDOR の有無が **仕様判断**(M-1-01)に依存。staff権限分離が必要なら schema 変更が必要<br>- **NG寄り**: `AuditLog.ipAddress` / `userAgent` の型がただの `String?` で長さ制限がスキーマレベルで定義されていない(SQLite TEXT は実質無制限)→ DoS / リソース枯渇リスク(F-7-01)<br>- **OK**: AuditLog の `userId Int?` + `ON DELETE SET NULL` (migrations で確認、E-3-01 参照) → ユーザー削除時もログ保持 ✅<br>- 関連: A-1-08、F-3-01 / F-6-01 / F-7-01、B-14-01、G-1-01、M-1-01。 |
| E-2-01 | High | 3.2 | `frontend/prisma/` | `seed.ts` | 初期パスワードのハードコーディング、bcryptjs の saltRounds、テストユーザの扱い | 全文(57行)読了。<br><br>(a) **初期パスワードのハードコーディング**: ❌ **ハードコードなし** ✅。`const seedPassword = process.env.SEED_ADMIN_PASSWORD` で環境変数から取得 [seed.ts:18]、未設定時は明示的にエラーで停止 [seed.ts:19-26]<br>(b) **8文字未満の検証**: `if (seedPassword.length < 8) throw` ✅ [seed.ts:27-29]<br>(c) **bcrypt saltRounds**: `bcrypt.hash(seedPassword, 12)` [seed.ts:30] → **12 rounds** ✅(BCRYPT_ROUNDS = 12 と統一、4.C.18 / L-1-01 と一致)<br>(d) **作成されるユーザー**: `***MASKED***` 1件のみ、`role: 'admin'`、`mustChangePassword: true`、`twoFactorEnabled: false` [seed.ts:32-41]<br>(e) **重複防止**: 既存の `***MASKED***` がいる場合は skip [seed.ts:9-16]<br>(f) **email**: `dev@arte.co.jp` ハードコード [seed.ts:35](frontend/prisma/seed.ts#L35) — テスト用ドメイン(K-3-04 で `@example.com` ヒット確認予定だが本ドメインは別)<br>(g) Prisma 7 の seed 設定は `prisma.config.ts` の `migrations.seed` で行う(memory 確認済み) | OK | **判定根拠**:<br>- **OK**: 初期PWの環境変数化 ✅、長さ検証 ✅、bcrypt rounds 統一 ✅、初回PW変更強制 ✅、2FA セットアップ強制(twoFactorEnabled=false で初回ログイン時に setup フローへ) ✅<br>- **OK寄り(軽微)**: email ハードコード `dev@arte.co.jp` だが、これは初期管理者ユーザーの仮設アドレスとして許容(初回ログイン後に変更可能であるべき、ただし変更機能の確認は別途)<br>- 関連: L-1-01 (bcrypt rounds 統一)、A-1-02 (SEED_ADMIN_PASSWORD 12文字)、I-1-01 (reset-password.ts も rounds=12)。 |
| E-3-01 | Medium | 3.2 | `frontend/prisma/migrations/` | `*_add_security_features/migration.sql` 他 | 旧adminテーブル残存データの懸念、Down migration 欠落時の復元手順、セキュリティ関連カラム追加時の NULL 制約 | マイグレーション一覧(9 件):<br>1. `20260213123835_init`<br>2. `20260213132327_add_news_model`<br>3. `20260213134716_add_blog_model`<br>4. `20260213140723_add_blog_image`<br>5. `20260213143040_add_category_model`<br>6. `20260214000000_add_blog_categories_many_to_many`<br>7. `20260214010000_add_admin_auth` — admins テーブル作成<br>8. `20260214020000_replace_admin_with_user` — **`DROP TABLE "admins"`** + `CREATE TABLE "users"`<br>9. `20260214094728_add_security_features` — locked_until / recovery_codes_hash 追加 + audit_logs テーブル<br><br>**(a) 旧 admins テーブル残存データ**:<br>- マイグレーション 8 で `DROP TABLE "admins"` を実行 ✅<br>- ⚠️ **`INSERT INTO users SELECT ...` が無い** → 本番環境で 7→8 のマイグレーションを適用すると admins のデータは**完全消失**<br>- dev 環境では影響軽微(admins テーブルにはテストデータのみ想定)<br>**(b) Down migration**:<br>- Prisma migrate は原則 forward-only。各マイグレーションディレクトリに `migration.sql` のみで `down.sql` はない(本リポジトリも同様)<br>- 復元手順: `prisma migrate reset` で初期化、または DB バックアップから復元<br>- 既存レポート §18 で「DBバックアップ未実装」と指摘済 → **復元手段なし**<br>**(c) セキュリティカラム追加時の NULL 制約**:<br>- マイグレーション 9: `ALTER TABLE "users" ADD COLUMN "locked_until" DATETIME` (NULL 許容) ✅<br>- 同: `ADD COLUMN "recovery_codes_hash" TEXT` (NULL 許容) ✅<br>- 既存ユーザーへの影響: NULL のままで問題なく動作 ✅<br>- audit_logs.user_id は `NULL` 許容 + `ON DELETE SET NULL` で適切 ✅ | 部分NG | **判定根拠**:<br>- **OK**: NULL 制約の妥当性 ✅、ON DELETE SET NULL ✅<br>- **NG寄り**: マイグレーション 8 (replace_admin_with_user) で **データ移行なしで admins → users へ置換** → 本番適用時の運用考慮なし。dev 環境ではすでに seed.ts で ***MASKED*** ユーザーを再作成可能だが、本番では運用ドキュメント要必須<br>- **NG寄り**: Down migration 不在 + DBバックアップ未実装(既存§18) → ロールバック不能<br>- **改善提案**: ① 本番マイグレーション適用前のDB バックアップ運用化、② 主要マイグレーションには手動 down 手順をコメントで記載、③ 本番では `prisma migrate deploy` で forward only 運用を文書化<br>- 関連: 既存§18(バックアップ)、A-1-08(dev.db履歴)、L-5-01(運用ドキュメント)。 |

---

## 章サマリ

- **回答済項目数:** 3件
  - Critical: 0件
  - High: 2件
  - Medium: 1件
  - Low: 0件
- **OK 件数:** 1件 (E-2-01)
- **NG 件数:** 0件
- **部分NG 件数:** 2件 (E-1-01, E-3-01)
- **判定不能 件数:** 0件
- **判定不能の主な理由:** 該当なし

## 参照したファイル一覧

- `frontend/prisma/schema.prisma` (全113行、第1章で読了済み・本章で再参照)
- `frontend/prisma/seed.ts` (全57行)
- `frontend/prisma/migrations/20260214020000_replace_admin_with_user/migration.sql` (全31行)
- `frontend/prisma/migrations/20260214094728_add_security_features/migration.sql` (全26行)
- 他マイグレーションは `ls` でファイル名のみ確認(計9マイグレーション)

## 実行した grep / コマンド一覧

- `ls /srv/git/arte-web-site-project/frontend/prisma/`
- `ls /srv/git/arte-web-site-project/frontend/prisma/migrations/`

## 章をまたぐ関連項目

- E-1-01 ⇔ A-1-08 (dev.db 履歴)、F-3-01 / F-6-01 / F-7-01 (lib層)、B-14-01 (oneTimePassword)、G-1-01 (users actions)、M-1-01 (author 権限分離)
- E-2-01 ⇔ A-1-02 (SEED_ADMIN_PASSWORD 12文字)、L-1-01 (bcrypt rounds 統一)、I-1-01 (reset-password.ts)
- E-3-01 ⇔ 既存§18 (バックアップ未実装)、A-1-08 (dev.db履歴)、L-5-01 (運用ドキュメント)
