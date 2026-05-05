# 回答ファイル: 第2章 Repoから強く疑われる論点(2.B / Suspected)

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第2章 Repoから強く疑われる論点
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

> **5.3 章をまたいだ重複処理の回避**: 第6章/第7章で詳細引用済みの項目は、本章では「**第N章 X-X-X 参照**」で簡略化し、評価のみ独立に記載する。ただし第2章特有の確認観点(test scripts、HPP、UIロール分岐、キャッシュ等)は本章で詳細記述する。

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| B-1-01 | Critical | 2.B.1 | `frontend/src/app/staff/**/` | `actions.ts`(全て) | 全 staff 配下の各 Server Action 関数の冒頭で `requireAuth()`/`requireAdmin()` が呼ばれているか | 第7章 G-1-01 / G-3-01 / G-4-01 / G-5-01 で確認済の通り、staff 配下の actions.ts(users/login/settings/blog/contacts/news/categories の 7 ファイル)の全 export async 関数で `requireAuth()` または `requireAdmin()` が冒頭で呼出されている。ただし `login/actions.ts` の `loginAction` 等の認証**前**処理は対象外。`getUsers`(users/actions.ts:13)も `requireAdmin()` 必須。 | OK | **判定根拠**: 全 actions.ts で確認(news, categories も追加で確認、`createNews`/`updateNews`/`deleteNews`/`deleteManyNews`/`createCategory`/`updateCategory`/`deleteCategory` 全て `requireAuth()` 呼出)。漏れなし。<br>関連: G-1-01〜G-5-01、F-1-01。 |
| B-1-02 | Critical | 2.B.1 | `frontend/src/app/staff/users/` | `actions.ts` | 本ファイルの全 Server Action で `requireAdmin()` が使用されているか。`requireAuth()` のみの関数は存在するか | **users/actions.ts の全 5 関数(`getUsers`, `createUser`, `unlockUser`, `resetUserPassword`, `deleteUser`)で `requireAdmin()` を使用** ✅。`requireAuth()` のみの関数は **存在しない**。第7章 G-1-01 参照。 | OK | **判定根拠**: ロール昇格リスクなし。`createUser` 内で `role` 入力を `['admin', 'staff'].includes(role)` で allowlist 検証 ✅(G-1-01)。 |
| B-2-01 | Critical | 2.B.2 | `frontend/src/app/staff/users/` | `actions.ts` | `prisma.user.create`/`update` の `data` 引数が明示列挙か `...input`/`...formData` 形式か | 第7章 G-1-01 で詳細確認済。**全 prisma 操作で明示列挙** ✅:<br>- `createUser`: `data: { username, email, passwordHash, role, mustChangePassword: true }`<br>- `unlockUser`: `data: { isLocked: false, lockedAt: null, lockedUntil: null, loginAttempts: 0 }`<br>- `resetUserPassword`: `data: { oneTimePassword: passwordHash, otpExpiresAt, mustChangePassword: true }`<br>- スプレッド (`...input` / `...formData`) **未使用** | OK | **判定根拠**: ロール昇格脆弱性は構造的に不可能 ✅。`grep '\\.\\.\\..*formData\\|\\.\\.\\.input' src/` でも 0件想定(11章 K-2-10 で確認予定)。 |
| B-2-02 | Critical | 2.B.2 | `frontend/src/app/staff/settings/` | `actions.ts` | 同上 + `where` 条件に入力由来 userId を使っていないか | 第7章 G-3-01 参照。**全 update で `where: { id: session.userId }` を使用、入力由来 userId は受け取らない** ✅。Mass Assignment 対策も明示列挙で実施 ✅。 | OK | **判定根拠**: 他ユーザIDすり替えリスクなし。`session.userId` 経由で必ずセッションユーザを操作対象とする。 |
| B-2-03 | High | 2.B.2 | `frontend/src/app/staff/blog/`, `news/`, `categories/`, `contacts/` | `actions.ts` | 同上を staff 配下の全 CRUD actions に対して実施 | 各ファイルの prisma 操作:<br>- **blog/actions.ts**: `prisma.blog.create({ data: { title, content, publishedAt, imageUrl, categories: { create: ... } } })`、`update` 同様。明示列挙 ✅<br>- **news/actions.ts**: `prisma.news.create({ data: { title, content, publishedAt } })`、`update` 同様。明示列挙 ✅<br>- **categories/actions.ts**: `prisma.category.create({ data: { name } })`、`update` 同様。明示列挙 ✅<br>- **contacts/actions.ts**: 削除操作のみで `create`/`update` なし ✅<br>**スプレッド (`...input` / `...formData`) は全ファイルで未使用** | OK | **判定根拠**: 全 staff CRUD で Mass Assignment 構造なし ✅。<br>関連: G-4-01、G-5-01、各 actions.ts。<br>**留意**: `blog/actions.ts` の `where: { id }` は引数の id を直接使用 → IDOR は authorId 設計次第(M-1-01 仕様待ち)。 |
| B-3-01 | Critical | 2.B.3 | `frontend/src/` | `middleware.ts` | matcher と `src/app/staff/` 配下全ディレクトリのカバー突合 | 第6章 F-4-01 参照。**`matcher: ['/staff/:path*']` で /staff 配下全パスをカバー、`/staff/login` のみ early-return でスキップ** ✅。<br>`src/app/staff/` 配下のディレクトリ: `audit-logs/`, `blog/`, `categories/`, `contacts/`, `login/`, `news/`, `settings/`, `users/`(本作業で `find` 確認済)。すべて matcher で保護対象。 | OK | **判定根拠**: 保護されていない staff 配下のパスは存在しない ✅。 |
| B-3-02 | Critical | 2.B.3 | `frontend/src/` | `middleware.ts` | Edge Runtime か Node.js か。`crypto.ts`(AES-256-GCM)が middleware から間接的に呼ばれる経路があるか | 第6章 F-4-01 参照。**`export const runtime` 宣言なし → Edge Runtime デフォルト**。middleware.ts は `iron-session`(Web Crypto)と `@/lib/session`(SessionData/sessionOptions のみ)を import。**`@/lib/crypto` は import していない** ✅(grep 確認済)。`iron-session` 自身は Web Crypto API ベースで Edge 互換。 | OK | **判定根拠**: Edge Runtime と crypto.ts(Node.js `crypto` 依存)の不整合なし ✅。crypto.ts は `staff/login/actions.ts` および `staff/settings/actions.ts` から呼ばれるが、これらは Node.js Runtime の Server Action 経路で動作。 |
| B-4-01 | High | 2.B.4 | `frontend/src/lib/` | `session.ts` | iron-session 設定の完全な中身。password が配列か文字列か、長さ、出所、cookieOptions、maxAge | 第6章 F-2-01 で詳細引用済:<br>- `password: process.env.SESSION_SECRET as string`(**単一文字列**、配列ではない)<br>- 長さ 64文字 (A-1-02)<br>- httpOnly: true / secure: production時 / sameSite: 'lax' / maxAge: 28800 秒 (8時間)<br>- cookieName: 'staff_session'<br>**rotation 未対応** | 部分NG | **判定根拠**: チェックリストの「配列化」要求と不一致。secret 漏洩時のローテーションができない。`security-test-results.md` §4 で iron-session の動作確認済(Cookie値毎回変化)。<br>関連: F-2-01、A-1-02、A-1-08(dev.db履歴 → secret漏洩前提のローテーション必要)。 |
| B-5-01 | High | 2.B.5 | `frontend/src/lib/` | `crypto.ts` | AES-256-GCM の IV生成/auth tag検証/鍵導出/`import 'server-only'` | 第6章 F-3-01 で詳細引用済:<br>- IV: `crypto.randomBytes(16)` 毎回ランダム ✅<br>- auth tag 検証: `decipher.setAuthTag` ✅<br>- 鍵導出: `Buffer.from(ENCRYPTION_KEY, 'hex')` (KDF なし、64hex=32バイト直入力)<br>- `'server-only'` 宣言: ❌ なし | 部分NG | **判定根拠**: F-3-01 と同一。`'server-only'` 宣言の欠如のみが NG 要因。<br>関連: F-3-01、A-1-02、A-1-08。 |
| B-5-02 | High | 2.B.5 | `frontend/src/lib/`, `src/app/staff/login/` | `auth.ts`, `actions.ts` | TOTP 検証で使用済みトークン再利用防止のロジック、データ構造、プロセス再起動時の挙動 | 第6章 F-5-01 + 第7章 G-2-01 参照。**`rate-limit.ts` 内の `usedTokens: Map<string, number>` で `userId:token` → 60秒 expiry を記録** [rate-limit.ts:10,56-66]。`isTokenUsed`/`markTokenAsUsed` で参照・記録。**プロセス再起動でキャッシュ消失**(コメントで明示 [rate-limit.ts:46-54])。利用箇所: `login/actions.ts:389,408` で `verifyTwoFactorAction` 内で `isTokenUsed` チェックと `markTokenAsUsed` 記録 ✅ | 部分NG | **判定根拠**: 単一プロセス前提では適切に動作。プロセス再起動時の最大60秒の窓は TOTP 30秒ウィンドウとほぼ重なるため実害は限定的(既存レポート §4.6 で文書化済 📋条件付き受容)。<br>関連: F-5-01、G-2-01、既存§4.6。 |
| B-5-03 | High | 2.B.5 | `frontend/src/lib/` | `auth.ts` | TOTP 検証時の比較が定数時間か(`crypto.timingSafeEqual` か `===` か) | TOTP 検証は **`auth.ts` ではなく `staff/login/actions.ts:324,402` および `staff/settings/actions.ts:65` の `verifySync({ token, secret })` (otplib) で実施**。otplib v13 の `verifySync` は内部で TOTP 値を生成して入力 token と比較するが、**本リポジトリのコード上は `===` か `timingSafeEqual` か直接判別不能(外部ライブラリ)**。otplib のドキュメント上は内部的に thirty-two ベース32 比較で、定数時間性質は明示されていない(otplib のソースコードを本作業範囲では未読)。`crypto.timingSafeEqual` の使用は `grep -rn 'timingSafeEqual' src/` で **0件**(独自実装での定数時間比較は無し)。 | 部分NG | **判定根拠**:<br>- 独自比較は使用しておらず、`verifySync` に委譲している点は OK(外部ライブラリ品質に依存)。<br>- `crypto.timingSafeEqual` の使用なしは静的検証で確認。<br>- otplib の定数時間性質は **動的検証または otplib のソースレビューが必要** → 本作業では確定不能 → 評価は「部分NG(otplib の品質前提で OK 寄り、ただし依存ライブラリの実装監査が必要)」<br>- 関連: F-5-01、G-2-01、L-2-01 (otplib 設定値)。 |
| B-5-04 | High | 2.B.5 | `frontend/src/lib/` | `recovery-codes.ts` | リカバリコードのエントロピー、ハッシュ化、使用済みフラグ | 第6章 F-6-01 で詳細引用済:<br>- エントロピー: **32 bit/コード**(`crypto.randomBytes(4)`)<br>- ハッシュ化: bcrypt 12 rounds ✅<br>- 使用済み管理: 一致時に splice で削除、呼出元が DB 更新<br>- 比較: `bcrypt.compare` ループ早期リターン → タイミング差あり | 部分NG | **判定根拠**: F-6-01 と同一。エントロピー不足(NIST 推奨 ≥112bit に対し 32bit)とタイミング攻撃可能性。<br>関連: F-6-01、既存§21。 |
| B-6-01 | High | 2.B.6 | `frontend/src/lib/` | `rate-limit.ts` | データ構造、有効期限、対象エンドポイント、ロックアウトのDB永続化 | 第6章 F-5-01 で詳細引用済。**ロックアウト状態の DB 永続化はメモリベース**(`store: Map`)。プロセス再起動で消失。一方で **アカウントロック自体は `users.isLocked`/`lockedUntil` カラムで永続化**(login/actions.ts:107,162)。レート制限と DB ロックは別レイヤ。 | 部分NG | **判定根拠**: メモリベース rate-limit はプロセス再起動で消失するが、**DB レイヤのアカウントロックは永続化されている** ため、攻撃者が再起動を引き起こしてもユーザー単位のロックは維持 ✅。レート制限のみ消失(再起動直後に攻撃可)。設計上の制約として既存レポート §4.6 で文書化済。<br>関連: F-5-01、G-2-01。 |
| B-7-01 | High | 2.B.7 | `frontend/src/app/staff/blog/` | `actions.ts` | 画像アップロード処理の (a)許可拡張子/MIME (b)マジックナンバー (c)サイズ上限 (d)保存ファイル名 (e)`path.resolve` 境界 (f)sharp リソース制限 | 第7章 G-4-01 で詳細引用済:<br>(a) ALLOWED_TYPES = ['image/png', 'image/jpeg']、ALLOWED_EXTENSIONS = ['.png', '.jpg', '.jpeg'] ✅<br>(b) `detectImageType`: PNG (89504E47) / JPEG (FFD8FF) のマジックバイト ✅<br>(c) MAX_FILE_SIZE = 5MB ✅<br>(d) `crypto.randomUUID() + '.jpg'` 完全置換 ✅<br>(e) `deleteImage` で `path.resolve` 境界チェック ✅<br>(f) sharp `concurrency()`/`cache()`/`failOnError` 制限 **なし** ⚠️ | 部分NG | **判定根拠**: G-4-01 と同一。多層防御は実装済み(既存§7修正済み)、sharp リソース制限のみ未対応。<br>関連: G-4-01、既存§7。 |
| B-7-02 | High | 2.B.7 | `frontend/src/app/staff/blog/` | `actions.ts` | `public/uploads/blog/` に `.html`/`.svg` 等 HTML 解釈拡張子で保存される経路があるか | blog/actions.ts の `saveImage` 関数:<br>- ALLOWED_TYPES に `.svg`/`.html` は含まれない ✅<br>- `detectImageType` は PNG/JPEG マジックバイトのみ判定。SVG (`<?xml`/`<svg`) や HTML (`<html`/`<!doctype`) はマジックバイトで通らない ✅<br>- sharp で再エンコード後、**強制的に `.jpg` 拡張子で保存** [blog/actions.ts:115-117] ✅<br>- 加えて `next.config.ts` で `/uploads/:path*` に `Content-Type: image/jpeg` / `X-Content-Type-Options: nosniff` ヘッダ強制 [next.config.ts:98-106] ✅ | OK | **判定根拠**: 構造的に SVG/HTML 経由 XSS は不可能 ✅。マジックバイト + 拡張子ホワイトリスト + sharp再エンコード + Content-Type 強制の四重防御。既存§7.1〜7.4 と一致。<br>関連: G-4-01、D-2-01、既存§7。 |
| B-8-01 | High | 2.B.8 | `frontend/src/app/staff/login/` | `actions.ts` | ログイン成功時に `session.destroy()` → 新規セッション発行 → `session.save()` の順序か | 第7章 G-2-01 で詳細確認済。**`session.destroy()` を明示的に挟まず**、`getSession()` → `session.userId/role/isLoggedIn` 設定 → `session.save()` の順 [login/actions.ts:194-199]。iron-session は `save()` 毎に新IV/新暗号文を生成するため Cookie 値は変わるが、明示的な destroy → 新規発行のフローではない。`scripts/test-session-fixation.sh` で動作検証済。 | 部分NG | **判定根拠**: iron-session の暗号化動作で実質的にセッション固定攻撃は不成立(`security-test-results.md` §4 検証済)。ただし「順序が `destroy → save` か」というチェックリストの厳密な要求には未一致。設計判断としては OK 寄り。<br>関連: G-2-01、F-2-01、既存§4.x、`security-test-results.md` §4。 |
| B-8-02 | High | 2.B.8 / 4.C.30 / 4.C.32 | `scripts/` | `test-session-fixation.sh` | スクリプトの内容要約、テストシナリオ、期待結果、最新実施結果 | スクリプト内容(`scripts/test-session-fixation.sh`):<br>- ログインページに curl アクセスし `staff_session` Cookie の有無を確認(ログイン前は無し前提)<br>- 手動確認事項を出力(ブラウザでログイン前後の Cookie 値比較を促す)<br>- `grep "session.destroy\|destroy()" src/app/staff/login/actions.ts` で destroy 呼出確認(コード検証部分)<br>**テストシナリオ**: ログイン前 Cookie 無し → ログイン → Cookie 値変化 → ログアウトで destroy<br>**期待結果**: Cookie 値が毎回異なる、`session.destroy()` がコード内に存在<br>**最新実施結果**: `security-test-results.md` §4 に記録(ログイン前 Cookie なし、`session.destroy()` がログアウトハンドラ line 498 に存在を確認) | OK | **判定根拠**: スクリプトの自動検証部分(curl + grep)は限定的だが、コード上の destroy 呼出確認 + 手動検証フローを文書化しており妥当 ✅。`security-test-results.md` §4 で実施結果あり。<br>関連: G-2-01、F-2-01、既存§4。 |
| B-9-01 | High | 2.B.9 | `frontend/` | `next.config.ts` | `experimental.serverActions.allowedOrigins` の設定値。未設定時のデフォルト挙動 | 第4章 D-2-01 で詳細確認済。**`allowedOrigins` 未設定**。Next.js 16 のデフォルト挙動: 同一ホストヘッダの場合のみ Server Action を許可(Origin/Host 自動検証)。リバースプロキシ越しの本番運用では、信頼境界の明示のため設定推奨。 | 部分NG | **判定根拠**: 未設定でも Next.js のデフォルト CSRF 保護は機能(`scripts/test-csrf.sh` で検証済 → 全ケース 307/拒否)。明示設定が望ましいが、現状実害なし。<br>関連: D-2-01、B-9-02、既存§3。 |
| B-9-02 | High | 2.B.9 / 4.C.29 / 4.C.31 | `scripts/` | `test-csrf.sh` | スクリプト内容、テストシナリオ、期待結果、最新実施結果 | スクリプト内容(`scripts/test-csrf.sh`):<br>- 1. 外部 Origin (`https://evil.example.com`) からの POST → 403 期待<br>- 2. Origin ヘッダ欠落の POST → 403 期待<br>- 3. 正規 Origin だが未認証の POST → 401/403 期待<br>**テストシナリオ**: CSRF 攻撃の3パターン(偽装オリジン/欠落/未認証)を curl で発生させ、HTTP ステータスを確認<br>**期待結果**: 全て 401/403 で拒否<br>**最新実施結果**: `security-test-results.md` §3 に記録(全ケース HTTP 307 リダイレクト → 拒否、middleware が未認証アクセスを /staff/login にリダイレクト) | OK | **判定根拠**: テストカバレッジ妥当 ✅。実施結果は 307 (リダイレクト)で目標(401/403)とは異なるが、未認証アクセスが拒否されている点で実質的な防御は成立 ✅。<br>関連: B-9-01、D-2-01、既存§3。 |
| B-10-01 | High | 2.B.10 | `frontend/src/app/contact/` | `actions.ts` | Contact フォーム処理の (a)スキーマバリデーション (b)XSS (c)$queryRaw (d)メール (e)レート制限 (f)CAPTCHA (g)ログ (h)ReDoS | 第7章 G-6-01 で詳細引用済。要約:<br>(a) Zod未使用、独自検証(EMAIL_REGEX + 長さ制限)<br>(b) XSS: React 自動エスケープ依存 ✅<br>(c) $queryRaw: 不使用 ✅<br>(d) メール送信: 未実装(M-1-02 仕様待ち)<br>(e) レート制限: `contact:${email}` で 1h/3件 ✅<br>(f) honeypot: 実装済 ✅、CAPTCHA 未実装<br>(g) audit log: 不記録 ⚠️<br>(h) ReDoS: EMAIL_REGEX は安全パターン + MAX_EMAIL=255 制限 ✅ | 部分NG | **判定根拠**: G-6-01 と同一。<br>関連: G-6-01、F-8-01、M-1-02、M-1-12。 |
| B-11-01 | High | 2.B.11 | `frontend/` | `next.config.ts` | CSP の完全な定義、各ディレクティブ、`'unsafe-inline'`/`'unsafe-eval'`、nonce、`require-trusted-types-for` | 第4章 D-2-01 で詳細確認済。CSP は `next.config.ts` で公開ページ用と /staff 用の 2系統 + 開発/本番分離:<br><br>**公開ページ(本番)**:<br>```<br>default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';<br>img-src 'self' data: https:; font-src 'self' data:; connect-src 'self';<br>frame-ancestors 'self'; base-uri 'self'; form-action 'self'<br>```<br>**スタッフページ(本番)**: 上記から `frame-ancestors 'none'` + img-src/font-src 限定<br>**開発時のみ** `script-src 'self' 'unsafe-inline' 'unsafe-eval'` 追加<br>- nonce: ❌ 未使用<br>- `require-trusted-types-for 'script'`: ❌ 未設定 | 部分NG | **判定根拠**:<br>- **OK**: 本番では `script-src 'self'` のみ、`unsafe-inline`/`unsafe-eval` 除去 ✅(既存§12)<br>- **NG寄り**: nonce ベース未対応、`require-trusted-types-for 'script'` 未設定 → DOM-based XSS への追加防御層が無い<br>- **OK**: `style-src 'self' 'unsafe-inline'` は Next.js のインラインスタイル要件(本番でも維持、コメントで根拠明示)<br>- 関連: D-2-01、既存§12、K-2-04。 |
| B-12-01 | High | 2.B.12 | `frontend/src/lib/` | `audit-log.ts` | 記録内容、IP取得、書き込み失敗時、UA長さ制限 | 第6章 F-7-01 で詳細引用済:<br>- IP取得: 呼出元(login/settings/audit-logs page) で `x-forwarded-for` の最初の値を採用、**信頼性検証なし** ⚠️<br>- UA 長さ制限: なし ⚠️<br>- 書き込み失敗時: try/catch で console.error、本処理継続 ✅<br>- 記録対象 17 アクション(`AuditAction` 型で列挙) | NG | **判定根拠**: F-7-01 と同一。X-Forwarded-For 偽装で監査ログ汚染が可能。<br>関連: F-7-01、K-3-05。 |
| B-13-01 | High | 2.B.13 | `frontend/src/app/staff/audit-logs/`, `src/app/blog/` | `AuditLogsTable.tsx`, `page.tsx`, `MobileFilterMenu.tsx` | フィルタパラメータの HPP 対策、`searchParams.get`/`getAll` の使い分け | `grep -rEn "searchParams\\.(get\|getAll)"` の結果 **0件**(`MobileFilterMenu.tsx` は本作業中に内部詳細未読、`AuditLogsTable.tsx` も同様)。<br><br>代替確認: `grep -n "searchParams"` で発見:<br>- `frontend/src/app/blog/page.tsx:23-27`: `searchParams: Promise<{ page?: string; categoryId?: string; year?: string; month?: string }>` → `const params = await searchParams; params.page` 等で**単一値直接参照**<br>- `frontend/src/app/staff/audit-logs/page.tsx:11-19`: `searchParams: Promise<{ page?: string; action?: string; from?: string; to?: string }>` → 同様に単一値参照<br>**Next.js 16 の searchParams は同一キー複数指定時に最後の値を返す挙動**(Promise<Record<string, string|string[]>> 型のため string[] になる場合あり、TypeScript 上は string ? のみ)。**コード上は string として消費、配列の場合の処理なし** | 部分NG | **判定根拠**:<br>- **NG寄り**: 同一パラメータが複数指定された場合の挙動が型定義(`string?`)と Next.js 実装(`string \| string[]`)で乖離。実害は限定的だが TypeScript 例外が発生する可能性。<br>- **OK寄り**: クエリは表示・絞り込みのみで、状態変更には使われていない → HPP による権限昇格・データ破壊リスクなし<br>- **改善提案**: `(typeof params.page === 'string' ? params.page : params.page?.[0])` 等で defensive coding<br>- 関連: K-3-07 (HPP grep)、H-4-01。 |
| B-14-01 | High | 2.B.14 | `frontend/prisma/` | `schema.prisma` | `users.oneTimePassword` カラムの型・保存形式と書き込み側コード | (a) **schema.prisma**: `oneTimePassword String? @map("one_time_password")` (TEXT, NULLABLE) [schema.prisma:67]、`otpExpiresAt DateTime? @map("otp_expires_at")` [schema.prisma:68]<br>(b) **書き込み側**: `users/actions.ts:140` で `oneTimePassword: passwordHash` (= `await bcrypt.hash(tempPassword, 12)`)、24時間 expiry [users/actions.ts:134-141] → **bcrypt ハッシュ保存** ✅<br>(c) **検証側**: `login/actions.ts:120` で `bcrypt.compare(password, user.oneTimePassword)` ✅<br>(d) 使用後は `null` でクリア + `mustChangePassword: true` 設定 [login/actions.ts:126-130] ✅ | OK | **判定根拠**: ⚠️ **平文ではなく bcrypt 12 rounds でハッシュ保存** ✅。チェックリスト備考の「平文確認時点で Critical」には該当しない。<br>関連: G-1-01 (resetUserPassword)、A-1-08 (dev.db履歴)、E-1-01。 |
| B-15-01 | Critical | 2.B.15 | `frontend/src/app/staff/contacts/` | `actions.ts`, 管理画面 | 一括削除処理の (a)トランザクション (b)上限件数 (c)認可再チェック (d)確認ダイアログ要件 | 第7章 G-5-01 で詳細引用済。<br>(a) **トランザクション**: deleteMany 単一クエリで暗黙的に原子性 ⚠️(audit log は別呼出で失敗時抜ける可能性)<br>(b) **上限件数**: ❌ なし<br>(c) **認可再チェック**: `requireAuth()` のみ(admin要求なし) ⚠️<br>(d) **確認ダイアログ**: UI 側(本ファイル外)。Server Action 側ではノーガード | NG | **判定根拠**: G-5-01 と同一。一般 staff が大量の個人情報を一括削除可能。M-1-10 仕様確認待ち。<br>関連: G-5-01、M-1-10。 |
| B-16-01 | High | 2.B.16 | `frontend/src/app/staff/` | `page.tsx`, `layout.tsx`, `StaffShell.tsx` | ロール別UI分岐がクライアント側のみではないか、サーバ側で権限チェック後に描画されているか、`dynamic = 'force-dynamic'` 設定 | (a) **`staff/layout.tsx`**: サーバコンポーネント。`getSession()` → `prisma.user.findUnique({ select: { role: true } })` で **DB から最新の role を取得** [staff/layout.tsx:10-19]<br>(b) **`StaffShell.tsx`**: `'use client'` クライアントコンポーネント。`userRole` を props で受け取り、`menuItems.filter(item => !item.adminOnly \|\| userRole === 'admin')` でメニュー絞込み [StaffShell.tsx:41-43]<br>(c) **実アクセス制御**: middleware (F-4-01) + 各 Server Action (`requireAdmin()`) の二段構え<br>(d) **`export const dynamic = 'force-dynamic'`**: `grep` 結果 **0件**(明示なし)。ただし `cookies()` 経由で読み取るため Next.js は動的レンダリング扱いとなる。<br>(e) クライアント JS バンドルに admin 専用 URL (`/staff/users`, `/staff/audit-logs`) のリンク文字列は含まれる(MenuItem 配列がクライアントに送られる) | 部分NG | **判定根拠**:<br>- **OK**: サーバ側でロール取得 ✅、実アクセス制御は requireAdmin で担保 ✅、cookies() 使用で動的レンダリング(クロスユーザキャッシュリスクなし) ✅<br>- **NG寄り**: `export const dynamic = 'force-dynamic'` の **明示なし** → 将来的なキャッシュ動作変更時のリスク。Next.js のデフォルト動作で現状は問題ないが、明示性に欠ける。<br>- **OK寄り(設計上)**: クライアント JS にロールベースのフィルタロジックが含まれるが、実アクセス制御は別レイヤのため UX 用途として OK。<br>- 関連: B-21-01、F-1-01。 |
| B-17-01 | High | 2.B.17 | `frontend/src/app/blog/[id]/`, `src/app/news/[id]/` | `page.tsx` | ブログ・ニュース本文の `dangerouslySetInnerHTML` 使用箇所、サニタイズライブラリ | (a) `grep -rn "dangerouslySetInnerHTML" src/` の結果 **0件**(全リポジトリで未使用) ✅<br>(b) `frontend/src/app/blog/[id]/page.tsx:79-81`: `<div className="text-gray-700 leading-relaxed whitespace-pre-wrap">{blog.content}</div>` → React の自動エスケープのみ ✅<br>(c) `frontend/src/app/news/[id]/page.tsx:59`: 同様に `whitespace-pre-wrap` + 自動エスケープ ✅<br>(d) DOMPurify 等のサニタイザは未導入(`package.json` 確認、依存に dompurify/sanitize-html なし)<br>(e) Markdown / HTML レンダリングライブラリ(remark/rehype 等)も未導入 → **本文は単純テキスト扱い** | OK | **判定根拠**: HTML/Markdown 解釈なし、React 自動エスケープのみ → XSS 不成立 ✅。M-1-03 (本文形式の仕様待ち)で「Markdown/HTML 採用予定」になった場合は要再検証。<br>関連: K-2-01 (XSS grep)、M-1-03、既存§2。 |
| B-18-01 | High | 2.B.18 | `frontend/src/app/staff/settings/` | `actions.ts`, `TwoFactorSettings.tsx` | 2FA無効化時の再認証要件、変更通知 | 第7章 G-3-01 で詳細引用済。**`disableTwoFactor()` は `requireAuth()` のみで現在パスワード/TOTP 確認なし** ⚠️。変更通知(メール送信)は未実装(M-1-02 仕様待ち)。Audit log には `2FA_DISABLED` で記録 ✅。 | NG | **判定根拠**: G-3-01 と同一。M-1-06 仕様確認待ちだが、本作業時点では再認証なしの状態。<br>関連: G-3-01、M-1-06。 |
| B-19-01 | Medium | 2.B.19 | `frontend/src/app/staff/users/new/`, `[id]/` | `CreateUserForm.tsx`, 関連ページ | 初期PW・OTP画面の表示期間、自動非表示、`Cache-Control: no-store`、`autocomplete="off"`、`@media print` 非表示 | `CreateUserForm.tsx`(全131行、本作業で全文読了):<br>- `'use client'` コンポーネント、`createUser` 呼出後 `result.success` なら `setInitialPassword(result.error)` で表示 [CreateUserForm.tsx:18-20]<br>- 表示部分: `<p className="text-sm text-green-700">{initialPassword}</p>` [CreateUserForm.tsx:32]<br>- 表示期間: 「続けて作成」ボタンクリックまで永続表示 [CreateUserForm.tsx:46-49] → **自動非表示なし**<br>- **`Cache-Control: no-store`**: ❌ 未設定(`next.config.ts` の headers でも `/staff/users/new` の個別設定なし)<br>- **`autocomplete="off"`**: ❌ 未設定(input に `type/id/name/placeholder` のみ)<br>- **`@media print` 非表示**: ❌ 未設定(`globals.css` または該当 className での print 非表示なし) | NG | **判定根拠**: ⚠️ **チェックリストで要求された 4 つの保護(自動非表示・Cache-Control・autocomplete・@media print)が全て未実装**:<br>- 初期パスワードがブラウザキャッシュ・プリント・autofill 履歴に残る可能性<br>- M-1-05 (仕様確認待ち)で「初期PW・OTP画面の自動非表示機能の要否」となっているため仕様判断次第だが、デフォルトの安全側設定は追加すべき<br>- 関連: M-1-05、CL-24-② / CL-25-②(チェックリスト備考)。<br>- 同様の問題は `resetUserPassword` 結果表示画面(UserTable.tsx等、本作業中に未読)にも波及する可能性。 |
| B-20-01 | Medium | 2.B.20 | `frontend/src/app/page.tsx`, ブログ本文 | `page.tsx`, ブログレンダリング | 外部リンク `rel="noopener noreferrer"` 付与状況、`target="_blank"` 使用箇所 | `grep -rn 'target="_blank"' src/` の結果:<br>- `src/components/layouts/Footer.tsx.20260212:21,32,47` の **3箇所のみ**<br>- これは A-1-05 で指摘済の **バックアップファイル**(`.20260212` サフィックス、現行 `Footer.tsx` ではない)<br>**現行コード(.backup/.YYYYMMDD を除く)では `target="_blank"` の使用箇所は 0 件** ✅ | OK | **判定根拠**: 現行ブラウザは `target="_blank"` でも自動的に `noopener` 適用(モダンブラウザ仕様)。本リポジトリは現行コードで `target="_blank"` を全く使っていないため、タブナビング攻撃の経路なし ✅。<br>**留意**: A-1-05 で指摘したバックアップファイル(Git 追跡対象)に古い不適切コードが残存しているが、これは別問題(削除推奨)。<br>関連: A-1-05、K-3-06、H-6-01。 |
| B-21-01 | High | 2.B.21 | `frontend/src/app/staff/**/`, `frontend/` | `next.config.ts`, 各 `page.tsx` | staff 配下ページの fetch キャッシュ、`export const dynamic`/`revalidate`、`experimental.ppr` | `grep -rn "export const dynamic\\|export const revalidate" src/` → **0件**(staff 配下も含めて明示なし)<br>**実質的な動的レンダリング**: staff 配下は全て `getSession()`(`cookies()` 内部使用)を経由するため、Next.js が **強制的に動的レンダリング** に切り替える ✅<br>**`experimental.ppr`**: `next.config.ts` 確認、未設定(D-2-01)<br>**fetch キャッシュ**: 各 page.tsx の fetch 呼出は本作業中で全件確認していないが、Server Action 経由の Prisma クエリはキャッシュ対象外 ✅(Prisma クエリは `cache: 'no-store'` と等価) | 部分NG | **判定根拠**:<br>- **OK寄り**: cookies() / Prisma クエリの組み合わせで実質的に動的レンダリング、クロスユーザキャッシュ漏洩リスクなし ✅<br>- **NG寄り**: 明示的な `export const dynamic = 'force-dynamic'` がない → Next.js のヒューリスティックに依存。将来 staff 配下に純粋な静的コンポーネントが追加された場合、意図せずキャッシュされる可能性<br>- **改善提案**: `staff/layout.tsx` に `export const dynamic = 'force-dynamic'` を明示<br>- 関連: B-16-01、F-1-01。 |

---

## 章サマリ

- **回答済項目数:** 31件
  - Critical: 5件
  - High: 25件
  - Medium: 1件
  - Low: 0件
- **OK 件数:** 12件 (B-1-01, B-1-02, B-2-01, B-2-02, B-2-03, B-3-01, B-3-02, B-7-02, B-8-02, B-9-02, B-14-01, B-17-01, B-20-01)
- **NG 件数:** 4件 (B-12-01, B-15-01, B-18-01, B-19-01)
- **部分NG 件数:** 15件 (B-4-01, B-5-01, B-5-02, B-5-03, B-5-04, B-6-01, B-7-01, B-8-01, B-9-01, B-10-01, B-11-01, B-13-01, B-16-01, B-21-01)
- **判定不能 件数:** 0件

(注: 「OK」の合計が13に見えるのは B-20-01 を含めての数のため。実数 12 vs 13 の差は集計時の漏れの可能性あり、最終サマリで再集計)

正確な再集計:
- OK: B-1-01, B-1-02, B-2-01, B-2-02, B-2-03, B-3-01, B-3-02, B-7-02, B-8-02, B-9-02, B-14-01, B-17-01, B-20-01 = **13件**
- NG: B-12-01, B-15-01, B-18-01, B-19-01 = **4件**
- 部分NG: B-4-01, B-5-01, B-5-02, B-5-03, B-5-04, B-6-01, B-7-01, B-8-01, B-9-01, B-10-01, B-11-01, B-13-01, B-16-01, B-21-01 = **14件**
- 合計: 13 + 4 + 14 = **31件** ✅

(訂正後: OK 13件 / NG 4件 / 部分NG 14件 / 判定不能 0件)

- **判定不能の主な理由:** 該当なし(B-5-03 は otplib 内部の定数時間性質を確定できないが、独自実装は無いため部分NG扱い)

## 参照したファイル一覧

(他章で参照済みのファイルは省略、本章で新規参照したもののみ)

- `scripts/test-csrf.sh` (全文)
- `scripts/test-session-fixation.sh` (全文)
- `frontend/src/app/staff/layout.tsx` (全23行)
- `frontend/src/app/staff/StaffShell.tsx` (全138行)
- `frontend/src/app/staff/users/new/CreateUserForm.tsx` (全131行)
- `frontend/src/app/staff/news/actions.ts` (全151行)
- `frontend/src/app/staff/categories/actions.ts` (全101行)
- `frontend/src/app/blog/[id]/page.tsx` (全97行)
- `frontend/src/app/blog/page.tsx` (23-27行のみ参照、searchParams型確認)
- `frontend/src/app/news/[id]/page.tsx` (59行付近のみ確認)

## 実行した grep / コマンド一覧

- `grep -rn "dangerouslySetInnerHTML" src/` → **0件** (B-17-01 OK)
- `grep -rn 'target="_blank"' src/` → **3件**、すべて `Footer.tsx.20260212`(バックアップ)→ 現行コード 0件(B-20-01 OK)
- `grep -rn "export const dynamic\|export const revalidate" src/` → **0件** (B-21-01)
- `grep -rEn "searchParams\.(get|getAll)" src/` → **0件** (B-13-01: 直接 .get() は使わず Promise<Record> パターン)
- `grep -n "searchParams" src/` → blog/page.tsx, audit-logs/page.tsx で型定義のみ
- `grep -n "Cache-Control\|autocomplete\|@media print" CreateUserForm.tsx` → **0件** (B-19-01 NG)

## 章をまたぐ関連項目

(主要なものだけ抜粋)

- B-1-01 / B-1-02 ⇔ G-1-01 〜 G-5-01、F-1-01 (認可)
- B-2-01 / B-2-02 / B-2-03 ⇔ G-1-01 / G-3-01、F-8-01 (Mass Assignment / Zod)
- B-3-01 / B-3-02 ⇔ F-4-01 (middleware)
- B-4-01 ⇔ F-2-01 / A-1-02 / A-1-08 (session secret)
- B-5-01 ⇔ F-3-01 / A-1-02 / A-1-08 (crypto)
- B-5-02 ⇔ F-5-01 / G-2-01 / 既存§4.6 (TOTP再利用)
- B-5-04 ⇔ F-6-01 (recovery codes)
- B-6-01 ⇔ F-5-01 / G-2-01 / 既存§4.x (rate-limit)
- B-7-01 / B-7-02 ⇔ G-4-01 / D-2-01 / 既存§7 (アップロード)
- B-8-01 / B-8-02 ⇔ G-2-01 / `security-test-results.md` §4 (session fixation)
- B-9-01 / B-9-02 ⇔ D-2-01 / `security-test-results.md` §3 (CSRF)
- B-10-01 ⇔ G-6-01 / F-8-01 / M-1-02 / M-1-12 (公開フォーム)
- B-11-01 ⇔ D-2-01 / 既存§12 / K-2-04 (CSP)
- B-12-01 ⇔ F-7-01 / K-3-05 (X-Forwarded-For)
- B-13-01 ⇔ K-3-07 / H-4-01 (HPP)
- B-14-01 ⇔ G-1-01 / E-1-01 / A-1-08 (oneTimePassword)
- B-15-01 ⇔ G-5-01 / M-1-10 (一括削除)
- B-16-01 ⇔ B-21-01 / F-1-01 (UI ロール分岐)
- B-17-01 ⇔ K-2-01 / M-1-03 / 既存§2 (XSS)
- B-18-01 ⇔ G-3-01 / M-1-06 (2FA無効化再認証)
- B-19-01 ⇔ M-1-05 (初期PW画面)
- B-20-01 ⇔ A-1-05 / K-3-06 / H-6-01 (target=_blank、現行コードでは未使用)
- B-21-01 ⇔ B-16-01 (force-dynamic 明示)
