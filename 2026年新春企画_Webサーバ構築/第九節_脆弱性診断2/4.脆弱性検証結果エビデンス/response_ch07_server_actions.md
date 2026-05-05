# 回答ファイル: 第7章 Server Actions確認

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第7章 Server Actions確認(3.4)
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| G-1-01 | Critical | 3.4 / 4.C.4 | `frontend/src/app/staff/users/` | `actions.ts` | (a)認可 (b)自己削除/降格防止 (c)PW生成 (d)入力検証 (e)Mass Assignment | (a) **認可**: 全 5 関数(`getUsers`, `createUser`, `unlockUser`, `resetUserPassword`, `deleteUser`)で **`requireAdmin()` 呼出済み** ✅ [users/actions.ts:13,35,91,127,162](frontend/src/app/staff/users/actions.ts)<br>(b) **自己削除防止**: `deleteUser` で `if (adminSession.userId === userId)` チェックあり ✅ [users/actions.ts:164-166](frontend/src/app/staff/users/actions.ts#L164-L166)。**自己降格(role 変更)機能はそもそも実装されておらず、role 更新パスは `createUser` のみ → 自己降格パス不在で実質的に防止済み**<br>(c) **パスワード生成**: `crypto.randomBytes(12).toString('base64url').slice(0, 16)` [users/actions.ts:63,134](frontend/src/app/staff/users/actions.ts) → 12バイト=96bit エントロピー、base64url で 16 文字に切り詰め(理論エントロピー 6×16=96 bit、十分)<br>(d) **入力検証**: `if (!username \|\| !email)` の必須チェックと `['admin', 'staff'].includes(role)` の allowlist [users/actions.ts:45-51](frontend/src/app/staff/users/actions.ts#L45-L51)。⚠️ **長さ制限なし、メール形式検証なし**<br>(e) **Mass Assignment 対策**: `data: { username, email, passwordHash, role, mustChangePassword: true }` で**明示的にフィールド列挙** ✅ [users/actions.ts:68](frontend/src/app/staff/users/actions.ts#L68)。**スプレッド (`...input`) 不使用** ✅ | 部分NG | **判定根拠**:<br>- **OK**: 認可・自己削除防止・Mass Assignment 対策・PW生成エントロピーは適切。<br>- **NG寄り**: 入力検証が薄い → username/email の長さ制限なし、email 形式検証なし、role 以外の `mustChangePassword`/`isLocked` 等のフィールドが攻撃者の formData で送信されても無視される構造(明示列挙)で対策済みだが、**Zod 等の体系的検証層が無い**(F-8-01 と直結)<br>- **NG寄り**: 初期パスワード `error: \`初期パスワード: ${initialPassword}\`` をエラーメッセージとして返す [users/actions.ts:79](frontend/src/app/staff/users/actions.ts#L79) → success 時に error 欄を流用しており **意味論が破綻**(UI が success 時に表示する想定だが、構造的に紛らわしい)。<br>- 関連: F-8-01 (Zod 不採用)、B-1-02 (admin 権限分離)、既存レポート §6.2 (初期PWレスポンス送信、MEDIUM)。 |
| G-2-01 | Critical | 3.4 / 4.C.13 | `frontend/src/app/staff/login/` | `actions.ts` | (a)セッション再生成順序 (b)レート制限連携 (c)エラーメッセージ統一 (d)2FA検証 (e)アカウントロック | (a) **セッション再生成**: `loginAction` は `getSession()` → `session.userId/role/isLoggedIn=false` → `session.save()` の順 [login/actions.ts:194-199](frontend/src/app/staff/login/actions.ts#L194-L199)。**`session.destroy()` を明示的に挟まない**。iron-session は `save()` 毎に新IV/新暗号文を生成するため Cookie 値は変わるが、`session.destroy()` → 新規作成という明示的フローではない。`security-test-results.md` §4 で検証済み(`scripts/test-session-fixation.sh`)。<br>(b) **レート制限**: ipLimit (10回/5分・15分ブロック) + userLimit (10回/5分) [login/actions.ts:66-68](frontend/src/app/staff/login/actions.ts#L66-L68) ✅<br>(c) **エラーメッセージ統一**: 全失敗パスで `LOGIN_ERROR_MESSAGE = 'ログイン情報が正しくないか、一時的に制限されています。'` ✅、ユーザー未存在/ロック中も `bcrypt.compare` ダミー実行でタイミング攻撃防止 ✅ [login/actions.ts:30,33,85,101,111](frontend/src/app/staff/login/actions.ts)<br>(d) **2FA検証**: `verifyTwoFactorAction` で rate-limit + `isTokenUsed` 再利用チェック + `decrypt` + `verifySync` ✅ [login/actions.ts:380-408](frontend/src/app/staff/login/actions.ts#L380-L408)<br>(e) **アカウントロック**: 5 試行で `isLocked=true`、`lockedUntil = now + LOCK_DURATIONS[index]`(15分→30分→1h→3h→24h)[login/actions.ts:36-42,154-164](frontend/src/app/staff/login/actions.ts) ✅ | 部分NG | **判定根拠**:<br>- **OK**: 既存レポート §4 / §1 / §2 / §4.7 / §4.8 で「修正済み」と評価された主要対策は本作業時点でも実装済み。タイミング攻撃防止・統一メッセージ・指数バックオフ・2FA再利用防止 すべて確認。<br>- **NG寄り**: ⚠️ **失敗時にも `lastLoginAt: new Date()` を更新** [login/actions.ts:150](frontend/src/app/staff/login/actions.ts#L150) → 「最終ログイン時刻」のセマンティクスが「最終アクセス時刻」になりブレる。攻撃者の試行も lastLoginAt を更新するため、ユーザーが「自分のログイン履歴」を確認しても異常検知できない。<br>- **OK寄り**: ロック期限切れ自動解除 ✅ [login/actions.ts:104-108](frontend/src/app/staff/login/actions.ts#L104-L108)<br>- **OK**: TOTP 6桁の長さ検証 ✅、bcrypt 12 rounds 統一 ✅<br>- 関連: B-8-01 (セッション固定)、B-5-02 (TOTP再利用)、F-1-01、既存レポート §4 全般。 |
| G-3-01 | Critical | 3.4 | `frontend/src/app/staff/settings/` | `actions.ts` | (a)自分PW変更時の現在PW確認 (b)2FA無効化時の再認証 (c)他ユーザIDすり替え防止 (d)Mass Assignment | (a) **自分PW変更時の現在PW確認**: `changePassword(currentPassword, newPassword)` で `bcrypt.compare(currentPassword, user.passwordHash)` 実施 ✅ [settings/actions.ts:158-161](frontend/src/app/staff/settings/actions.ts#L158-L161)<br>(b) **2FA無効化時の再認証**: ❌ **未実装**。`disableTwoFactor()` は `requireAuth()` のみで、現在パスワード/TOTP の再確認なし [settings/actions.ts:103-134](frontend/src/app/staff/settings/actions.ts#L103-L134)。攻撃者がセッション窃取に成功すれば即座に 2FA を無効化できる。<br>(c) **他ユーザIDすり替え防止**: 全 update で `where: { id: session.userId }` を使用 ✅。**入力 userId は引数で受け取らず、必ずセッションから取得**([settings/actions.ts:75,112,170,197](frontend/src/app/staff/settings/actions.ts) 全て同パターン)<br>(d) **Mass Assignment 対策**: 全 update で `data: { totpSecret: ..., twoFactorEnabled: ..., recoveryCodesHash: ... }` 等、**明示的にフィールド列挙** ✅。スプレッド不使用 | NG | **判定根拠**:<br>- **NG**: ⚠️ **2FA 無効化時の再認証なし**(B-18-01、M-1-06 の仕様確認待ちだが、デフォルトは「セッション窃取で即2FA無効化可能」のリスク状態)<br>- **OK**: PW変更時の現在PW確認 ✅、他ユーザID防止 ✅、Mass Assignment 対策 ✅<br>- **OK寄り(設計上)**: 2FA有効ユーザのPW変更時にリカバリコードを再生成 [settings/actions.ts:166-194](frontend/src/app/staff/settings/actions.ts#L166-L194) → PW漏洩=リカバリコードも漏洩前提のローテーション、良好。<br>- 関連: B-18-01、M-1-06 (仕様決定待ち)、F-1-01、既存レポート §21。 |
| G-4-01 | High | 3.4 | `frontend/src/app/staff/blog/` | `actions.ts` | (a)認可 (b)IDOR (c)XSS (d)ファイル名サニタイズ (e)sharp リソース制限 (f)Prototype Pollution | (a) **認可**: 全関数(`createBlog`, `updateBlog`, `deleteBlog`, `deleteManyBlogs`)で **`requireAuth()` 呼出済み** ✅ [blog/actions.ts:141,193,251,282](frontend/src/app/staff/blog/actions.ts)<br>(b) **IDOR**: blog の `authorId` 設計が **存在しない**(schema.prisma の Blog model 確認済み)→ staff全員が全 blog を編集・削除可能。**M-1-01 の仕様確認待ち**<br>(c) **XSS**: blog content は `Prisma` 経由で文字列保存。表示は B-17-01 で `dangerouslySetInnerHTML` の使用箇所を確認予定。本ファイル内では HTML 解釈なし<br>(d) **ファイル名サニタイズ**: `crypto.randomUUID()` ベースで完全置換 ✅ [blog/actions.ts:115-117](frontend/src/app/staff/blog/actions.ts#L115-L117)。ユーザ入力ファイル名は使わず<br>(e) **sharp リソース制限**: `sharp(buffer).resize(1920, 1080, { fit: 'inside', withoutEnlargement: true }).jpeg({ quality: 85 })` で固定パイプライン [blog/actions.ts:103-109](frontend/src/app/staff/blog/actions.ts#L103-L109)。⚠️ **`sharp.cache()` / `sharp.concurrency()` / `failOnError` 等の制限なし**。並列大量アップロード時の OOM リスクあり。`MAX_FILE_SIZE = 5MB` [blog/actions.ts:14](frontend/src/app/staff/blog/actions.ts#L14) で入力上限あり<br>(f) **Prototype Pollution**: `parseCategoryIds` で `JSON.parse(raw) as number[]` → `slice(0, MAX_CATEGORIES)` → `categoryIds.map((c) => ({ categoryId }))` [blog/actions.ts:79-87](frontend/src/app/staff/blog/actions.ts#L79-L87)。値は number 配列として slice/map されるだけで、**`__proto__` / `constructor` キー名アクセスなし** ✅ | 部分NG | **判定根拠**:<br>- **OK**: マジックバイト検証 + sharp再エンコード + UUID ファイル名 + path.resolve 境界チェック(`deleteImage` [blog/actions.ts:122-134](frontend/src/app/staff/blog/actions.ts#L122-L134))という多層防御 ✅(既存レポート §7 / §9 と一致)<br>- **NG寄り**: title/content の **長さ制限なし**(空判定のみ) → DoS リスク。MAX_FILE_SIZE は image のみ<br>- **NG寄り(リソース制限)**: sharp の concurrency 制御なし → 同時並列でのリソース枯渇リスク<br>- **OK寄り(IDOR)**: M-1-01 の仕様確認待ち。「staff全員が全記事編集」が仕様なら現実装は OK<br>- **OK**: parseCategoryIds の MAX_CATEGORIES=5 で配列サイズ上限あり ✅<br>- 関連: B-7-01 / B-7-02 / M-1-01 / M-1-03、既存レポート §7。 |
| G-5-01 | High | 3.4 | `frontend/src/app/staff/contacts/` | `actions.ts` | 一括削除の (a)トランザクション (b)上限件数 (c)認可再チェック | (a) **トランザクション**: `prisma.contact.deleteMany({ where: { id: { in: ids } } })` 単一クエリで実装 [contacts/actions.ts:46](frontend/src/app/staff/contacts/actions.ts#L46) → SQLite では暗黙的に1トランザクション。`createAuditLog` は別呼出のため、削除成功+ログ失敗の組み合わせでログ抜けの可能性(失敗時 console.error のみ)<br>(b) **上限件数**: ❌ **未制限**。`if (ids.length === 0)` の空判定のみ [contacts/actions.ts:34-36](frontend/src/app/staff/contacts/actions.ts#L34-L36)。M-1-10 で仕様確認待ち<br>(c) **認可再チェック**: `requireAuth()` のみ [contacts/actions.ts:40](frontend/src/app/staff/contacts/actions.ts#L40)。**`requireAdmin()` ではない** → 一般 staff も全件削除可能。仕様次第(M-1-10) | NG | **判定根拠**:<br>- **NG**: ⚠️ **上限件数なし** → 巨大な ids 配列(数万件以上)を POST されると SQL クエリが肥大化し DoS / リソース枯渇 / Cascade 削除遅延のリスク。SQLite の `IN` 句にも理論的に SQLITE_MAX_VARIABLE_NUMBER (デフォルト 999) の制約あり、超過時はエラーで全件失敗(ロールバックされる動作だが UX 悪化)<br>- **NG**: ⚠️ admin 権限要求なし → 一般 staff が個人情報含むコンタクトデータを一括削除可能(B-15-01 と直結)<br>- **OK寄り(トランザクション)**: deleteMany は単一クエリで原子性あり<br>- **改善提案**: ① `requireAdmin()` への変更、② `ids.length > 100` 等の上限、③ 確認ダイアログの仕様化(M-1-10)<br>- 関連: B-15-01、M-1-10。 |
| G-6-01 | High | 3.4 / 4.C.10 | `frontend/src/app/contact/` | `actions.ts` | B-10-01 と同範囲 + nodemailer 等のメール送信ライブラリ使用状況 | (a) **Zod等スキーマバリデーション**: ❌ **Zod 未使用**(F-8-01 参照)。独自検証で `EMAIL_REGEX = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$/` + `MAX_NAME=100` / `MAX_EMAIL=255` / `MAX_MESSAGE=5000` の長さ制限あり [contact/actions.ts:6-9](frontend/src/app/contact/actions.ts#L6-L9)<br>(b) **XSS対策**: 入力は `.trim()` のみ。表示時は React の自動エスケープ依存(管理画面側、本ファイル外)。HTML/Markdown 解釈なし ✅<br>(c) **`$queryRaw` 使用**: ❌ なし(Prisma の型安全 API のみ) ✅<br>(d) **メール送信実装**: ❌ **実装なし** [contact/actions.ts 全文](frontend/src/app/contact/actions.ts)。`nodemailer` / `resend` 等のライブラリ未使用、`prisma.contact.create` で DB 保存のみ。**M-1-02 で実装方針確認待ち**(将来実装時にヘッダインジェクション対策必須)<br>(e) **レート制限**: `rateLimit(\`contact:\${email}\`, 3, 60*60*1000)` で 1時間 3 件/メールアドレス [contact/actions.ts:45](frontend/src/app/contact/actions.ts#L45) ✅<br>(f) **CAPTCHA / honeypot**: ✅ honeypot あり: `formData.get('website')` で隠しフィールド検出、ボット時は成功レスポンスで偽装応答 [contact/actions.ts:34-42](frontend/src/app/contact/actions.ts#L34-L42)<br>(g) **ログ記録**: `createAuditLog` 呼出なし。コンタクト送信は audit_logs に記録されない。<br>(h) **ReDoS 対策**: `EMAIL_REGEX` は `[a-zA-Z0-9._%+-]+` のように **`+` 量指定子の重複領域なし**(各文字クラスが互いに排他的)で典型的な catastrophic backtracking パターンに該当しない。`MAX_EMAIL=255` で長さ制限ありのため実害は限定的。 | 部分NG | **判定根拠**:<br>- **OK**: 公開フォームに必要な対策(長さ制限・honeypot・rate-limit・Prisma 経由保存・自動エスケープ)はほぼ実装済み ✅。既存レポート §8.1 で「HIGH:残存」と指摘されていた「コンタクトフォームのレート制限なし」は **本作業時点で修正済み**(`rateLimit('contact:${email}', 3, ...)` 確認)。既存レポートの最終更新後の改善。<br>- **OK寄り**: EMAIL_REGEX は実用範囲で安全(ReDoS 不成立パターン)。<br>- **NG寄り**: メール送信機能未実装(M-1-02 仕様待ち)、CAPTCHA 未実装(M-1-12 仕様待ち)<br>- **NG寄り**: コンタクト送信が audit_logs に記録されない → 攻撃者の大量送信パターンが既存ログから追跡できない(レート制限ログとは別系統)<br>- **NG寄り**: 意図的な遅延 `setTimeout(1000)` [contact/actions.ts:28](frontend/src/app/contact/actions.ts#L28) → これは **DoS 増幅要因**(攻撃者から見れば1秒待機もコストだが、サーバ側でも 1秒間ハンドラがブロック)。本来UI演出はクライアント側で実装すべき。<br>- 関連: B-10-01 / M-1-02 / M-1-12 / F-8-01、既存レポート §8.1 / §20。 |

---

## 章サマリ

- **回答済項目数:** 6件
  - Critical: 3件
  - High: 3件
  - Medium: 0件
  - Low: 0件
- **OK 件数:** 0件
- **NG 件数:** 2件 (G-3-01, G-5-01)
- **部分NG 件数:** 4件 (G-1-01, G-2-01, G-4-01, G-6-01)
- **判定不能 件数:** 0件
- **判定不能の主な理由:** 該当なし(IDOR や仕様待ち項目は「現実装」の評価として `部分NG` で記録)

## 参照したファイル一覧

- `frontend/src/app/staff/users/actions.ts` (全186行)
- `frontend/src/app/staff/login/actions.ts` (1-499行、500行ファイル全体)
- `frontend/src/app/staff/settings/actions.ts` (全215行)
- `frontend/src/app/staff/blog/actions.ts` (全310行)
- `frontend/src/app/staff/contacts/actions.ts` (全61行)
- `frontend/src/app/contact/actions.ts` (全99行)

## 実行した grep / コマンド一覧

- `find frontend/src/app -name "actions.ts"` (公開1 + staff7 = 計8ファイル特定)
- `wc -l <8ファイル>` (主要6ファイルが合計1,371行)

## 章をまたぐ関連項目

- G-1-01 ⇔ B-1-02 (admin専用)、B-2-01 (Mass Assignment)、F-1-01 / F-8-01、既存§6.2
- G-2-01 ⇔ B-8-01 (セッション固定)、B-5-02 (TOTP)、既存§4.x、`security-test-results.md` §1/§2/§4
- G-3-01 ⇔ B-2-02、B-18-01、M-1-06 (2FA無効化時の再認証)
- G-4-01 ⇔ B-7-01 / B-7-02 (アップロード)、M-1-01 (author 権限)、M-1-03 (本文形式)、既存§7
- G-5-01 ⇔ B-15-01、M-1-10 (一括削除上限)、F-1-01 (admin判定)
- G-6-01 ⇔ B-10-01、M-1-02 (メール実装)、M-1-12 (HIBP/CAPTCHA)、F-8-01 (Zod)、既存§8.1/§20
