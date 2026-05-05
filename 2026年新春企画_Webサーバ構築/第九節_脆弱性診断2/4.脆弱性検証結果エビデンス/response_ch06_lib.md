# 回答ファイル: 第6章 認証・セキュリティライブラリ層確認

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第6章 認証・セキュリティライブラリ層確認(3.3)
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| F-1-01 | Critical | 3.3 / 4.C.3 | `frontend/src/lib/` | `auth.ts` | `requireAuth()` / `requireAdmin()` の内部実装、セッション取得経路、ロール判定、未認証時の挙動、`import 'server-only'` 宣言 | (a) **`getSession()`**: `getIronSession<SessionData>(await cookies(), sessionOptions)` で iron-session 経由でセッション取得 [auth.ts:7-9](frontend/src/lib/auth.ts#L7-L9)<br>(b) **`requireAuth()`**: `session.isLoggedIn && session.userId` を確認、未認証なら `throw new Error('認証が必要です。')` [auth.ts:38-44](frontend/src/lib/auth.ts#L38-L44)<br>(c) **`requireAdmin()`**: `requireAuth()` 後に `prisma.user.findUnique({ select: { id, role } })` で **DB から最新の role を取得**、`user.role !== 'admin'` で `throw` [auth.ts:47-57](frontend/src/lib/auth.ts#L47-L57)<br>(d) **未認証時挙動**: `throw` (redirect ではない) → Next.js Server Action 呼び出し元で例外として伝播<br>(e) **`import 'server-only'`**: ❌ **宣言なし** | NG | **判定根拠**:<br>- **OK寄り**: requireAdmin で **DB から最新ロールを取得** している点は良好(セッション内 `role` への依存ではなく、サーバ側真実)<br>- **NG**: `import 'server-only'` 宣言なし → クライアントバンドルへの誤ロード防止策が無い。Server Actions 経由でのみ呼ばれる構造だが、import グラフ上の保護策として `'server-only'` は**必須レベル**。<br>- **NG寄り**: throw で伝播するため、上位の Server Action が catch せずに UI に Error が漏れた場合、内部メッセージ「認証が必要です。」が露出する可能性。redirect でなく throw を使う設計は意図的だが、catch の徹底が前提となる。<br>- 既存レポート §5.2 で「修正済み」と評価されているが、**`import 'server-only'` の欠如は未指摘の追加観点**。<br>- 関連: B-1-01 (staff actions の認証チェック呼出有無)、F-2-01。 |
| F-2-01 | Critical | 3.3 / 4.C.1 | `frontend/src/lib/` | `session.ts` | iron-session `cookieOptions` 全属性、`password` 出所と長さ、rotation 設計、`import 'server-only'` 宣言 | (a) **`password`**: `process.env.SESSION_SECRET as string` (**単一文字列**、配列ではない) [session.ts:12](frontend/src/lib/session.ts#L12)。長さは A-1-02 で確認済の **64 文字**。<br>(b) **`cookieName`**: `'staff_session'` [session.ts:13](frontend/src/lib/session.ts#L13)<br>(c) **`cookieOptions`**: [session.ts:14-19](frontend/src/lib/session.ts#L14-L19)<br>- `httpOnly: true` ✅<br>- `secure: process.env.NODE_ENV === 'production'` ✅(本番のみ)<br>- `sameSite: 'lax'` ✅<br>- `maxAge: 60 * 60 * 8` (8 時間)<br>- `path` / `domain` は明示なし(デフォルト = `/` / current domain)<br>(d) **rotation 設計**: ❌ **未対応**(password が単一文字列。iron-session 8 は `password: ['current-secret', 'old-secret']` 形式の配列でローテーションをサポートするが、ここでは未利用)<br>(e) **`import 'server-only'`**: ❌ **宣言なし**(ただしこのファイルは middleware.ts(Edge Runtime)からも import されるため、`'server-only'` を付けると middleware が破損する。意図的な省略の可能性あり) | 部分NG | **判定根拠**:<br>- **OK**: cookieOptions 主要属性 (httpOnly/secure/sameSite/maxAge) すべて適切。既存レポート §4.1 と一致。<br>- **NG寄り**: SESSION_SECRET の **rotation 機能が未実装**。チェックリスト B-4-01 が「配列化」を要求している意図と一致。secret 漏洩時の即時無効化のため、配列 `[newSecret, oldSecret]` 化が望ましい。<br>- **設計的制約**: `'server-only'` を付けないのは middleware.ts が Edge Runtime で本ファイルを import する構造に起因。session.ts 単体は Web Crypto のみ使用で Edge 互換のため OK。`auth.ts`/`crypto.ts` 側に `'server-only'` を付ければよい(現状ではどちらにも無い)。<br>- 関連: A-1-02 (鍵長)、B-4-01、F-1-01、F-4-01。 |
| F-3-01 | Critical | 3.3 / 4.C.2 | `frontend/src/lib/` | `crypto.ts` | AES-256-GCM の (a)IV生成 (b)auth tag検証 (c)鍵導出方法 (d)エラー時の挙動 (e)`import 'server-only'` 宣言 | (a) **IV生成**: `crypto.randomBytes(16)` で **毎回 16 バイト(128 bit)のランダム IV** [crypto.ts:9](frontend/src/lib/crypto.ts#L9)<br>(b) **auth tag 検証**: `decipher.setAuthTag(Buffer.from(authTagHex, 'hex'))` で復号前にセット → `decipher.final()` 呼出時に検証され、不一致なら例外 [crypto.ts:31-33](frontend/src/lib/crypto.ts#L31-L33) ✅<br>(c) **鍵導出**: `Buffer.from(ENCRYPTION_KEY, 'hex')` で **hex デコードのみ**(KDF/HKDF は経由しない)。鍵自体が 64 hex chars = 32 バイト = 256 bit で AES-256 直入力 [crypto.ts:11-13](frontend/src/lib/crypto.ts#L11-L13)<br>(d) **エラー挙動**: 鍵長不一致時に `throw new Error('ENCRYPTION_KEY must be 64 hex characters (32 bytes)')` を encrypt/decrypt の両方の入口で実施 [crypto.ts:6-7,22-23](frontend/src/lib/crypto.ts)。auth tag 不一致時は `decipher.final()` が `Error: Unsupported state or unable to authenticate data` を投げる(Node.js デフォルト)。<br>(e) **`import 'server-only'`**: ❌ **宣言なし** | 部分NG | **判定根拠**:<br>- **OK**: IV 毎回ランダム ✅、auth tag 検証 ✅、鍵長検証 ✅、AES-256-GCM 標準実装 ✅<br>- **OK寄り**: 鍵導出に KDF を使わないのは、`ENCRYPTION_KEY` を生成時点で十分なエントロピーを持つ 256-bit 鍵として運用する前提なら問題ない(本実装はその前提)。<br>- **NG**: `import 'server-only'` **宣言なし** → Node.js `crypto` モジュールに依存するため、誤ってクライアントバンドルに混入するとビルドエラーになる(自然な防御)が、明示的な保護として `'server-only'` を入れるのが推奨。<br>- **軽微**: GCM の IV は NIST SP 800-38D で 96-bit (12 バイト)が推奨だが、本実装は 128-bit (16 バイト)。仕様上は問題ないが標準とは異なる。<br>- **重大級リスク(間接)**: ENCRYPTION_KEY は dev.db を**復号**する用途には現状使われていない(crypto.ts の利用先要grep)が、`pendingTotpSecret` 暗号化等で使われている可能性。A-1-08 (dev.db 履歴) と組み合わせ、ENCRYPTION_KEY が漏洩していれば過去の暗号化値も復号可能。<br>- 関連: A-1-02、A-1-08、B-5-01、F-2-01。 |
| F-4-01 | Critical | 3.3 / 4.C.5 | `frontend/src/` | `middleware.ts` | matcher 設定、Edge Runtime 対応、cryptoモジュール呼出の有無、認証失敗時の挙動 | (a) **matcher**: `['/staff/:path*']` で **/staff 配下全パスをカバー** [middleware.ts:30](frontend/src/middleware.ts#L30)<br>(b) **`/staff/login` のスキップ**: `if (pathname === '/staff/login') return NextResponse.next()` で除外 [middleware.ts:9-11](frontend/src/middleware.ts#L9-L11)<br>(c) **Runtime**: `export const runtime` 宣言なし → **Next.js のデフォルト = Edge Runtime**<br>(d) **import**: `iron-session` と `@/lib/session`(SessionData / sessionOptions のみ)。`@/lib/crypto`(Node.js `crypto` モジュール依存)は **import していない** ✅<br>(e) **認証失敗挙動**: `NextResponse.redirect(new URL('/staff/login', request.url))` ✅<br>(f) **権限チェック**: `session.isLoggedIn && session.userId` の存在のみ確認(role チェックなし) | 部分NG | **判定根拠**:<br>- **OK**: matcher で /staff 配下全保護 ✅、redirect 動作 ✅、crypto.ts への依存なし(Edge Runtime 互換)✅、iron-session は Web Crypto ベースで Edge 互換 ✅<br>- **OK寄り**: role チェックを middleware で行わない設計は意図的(管理者専用ページは Server Action 側で `requireAdmin()` を呼ぶ二段構え)。<br>- **軽微な懸念**: `pathname === '/staff/login'` は正確なパス一致のみで、クエリ文字列やトレイリングスラッシュ等で迂回されないが、`/staff/login/` (trailing slash) などのパス変種への対応は Next.js の正規化に依存する(実装側は OK)。<br>- 関連: B-3-01 (matcher 全カバー)、B-3-02 (Edge Runtime + crypto)、F-2-01。<br>- 既存レポート §5.1 で「修正済み」と評価。本作業でも矛盾なし。 |
| F-5-01 | High | 3.3 / 4.C.12 | `frontend/src/lib/` | `rate-limit.ts` | データ構造、有効期限処理、競合状態、プロセス再起動時挙動、対象エンドポイントリスト | (a) **データ構造**: `Map<string, RateLimitEntry>` (count/resetTime/blocked) + `Map<string, number>` (TOTP 使用済みトークン userId:token → 60秒の expiry) [rate-limit.ts:7-10](frontend/src/lib/rate-limit.ts#L7-L10)<br>(b) **有効期限処理**: `now > entry.resetTime` 検出時にエントリ再生成 [rate-limit.ts:21-23](frontend/src/lib/rate-limit.ts#L21-L23)、`setInterval` で 1 時間毎に期限切れエントリの全削除 [rate-limit.ts:69-83](frontend/src/lib/rate-limit.ts#L69-L83)<br>(c) **競合状態**: Node.js は単一スレッドのため Map 操作の競合はない。ただし複数プロセス(クラスタ/k8s pods)では各プロセスが独自の Map を持つため、合算なしの不正確なカウント。<br>(d) **プロセス再起動時挙動**: **キャッシュ全消失**(コメントで明示的に文書化 [rate-limit.ts:46-54](frontend/src/lib/rate-limit.ts#L46-L54))<br>(e) **対象エンドポイント**: rate-limit.ts は汎用関数のみ。利用先は `staff/login/actions.ts` (login:ip / login:user / 2fa:user / 2fa-setup:user / recovery:user) など — 既存レポート §4.3 / §4.5 / §4.6 で網羅的に記載 | 部分NG | **判定根拠**:<br>- **OK**: 単一プロセス前提では適切に動作 ✅、文書化済みの設計制約 ✅、ブロック機構あり ✅、自動クリーンアップ ✅<br>- **NG寄り(設計上の限界)**: メモリベースのため水平スケール時に効果が分散し制限緩和される(既存レポート §4.6 / §8.3 で「📋条件付き受容」、Redis 移行が必要と文書化済み)。<br>- 既存レポート §4.3 と一致。本作業時点で実装変更なし。<br>- 関連: B-6-01、F-7-01、既存レポート §4.3 / §4.6。 |
| F-6-01 | High | 3.3 | `frontend/src/lib/` | `recovery-codes.ts` | リカバリコードのエントロピー源、ハッシュ化有無、使用済みフラグ管理、定数時間比較 | (a) **エントロピー源**: `crypto.randomBytes(4).toString('hex').toUpperCase()` → 4 バイト = **32 bit エントロピー** / 1コード [recovery-codes.ts:9](frontend/src/lib/recovery-codes.ts#L9)。10 コード生成。<br>(b) **ハッシュ化**: `bcrypt.hash(code, 12)` で全コードを bcrypt 12 rounds でハッシュ [recovery-codes.ts:18](frontend/src/lib/recovery-codes.ts#L18)<br>(c) **使用済みフラグ管理**: `verifyRecoveryCode` で一致したインデックスを `splice(i, 1)` で配列から除去し、残りを返却。呼出元が DB の `recoveryCodesHash` を更新する設計 [recovery-codes.ts:30-36](frontend/src/lib/recovery-codes.ts#L30-L36)<br>(d) **定数時間比較**: `bcrypt.compare(code, hashes[i])` を使用。bcryptjs の compare 関数は内部で定数時間比較を実施(ライブラリ仕様)。ただし**ループによる早期リターン (`return { valid: true, ... }` でループ終了)** があり、コード位置(配列内 index)に応じて呼出時間が変わる。 | 部分NG | **判定根拠**:<br>- **NG寄り**: ⚠️ **エントロピー 32 bit /コードは NIST SP 800-63 推奨(≥112 bit) を大幅に下回る**。1 コード総当たりは 2^32 ≈ 4.3×10⁹ 試行。bcrypt 12 rounds (~250ms/試行) で逐次なら ~34 年だが、レート制限(5回/5分)が効かないオフライン状況では意味ない。**実質的脅威**: 攻撃者が DB からハッシュを取得した時のオフライン総当たりに対し、bcrypt rounds=12 でも長期間の総当たりは可能性ゼロではない。<br>- **NG寄り(タイミング)**: ループ早期リターンで配列内位置によりレスポンス時間が変動。攻撃者が複数の同一コードで応答時間を測れば、配列内のハッシュ位置を特定可能(ただし回復コード自体を特定する直接的攻撃にはならない)。<br>- **OK**: bcrypt 12 rounds でハッシュ保存 ✅、使用済みコード削除 ✅、`crypto.randomBytes` 使用 ✅<br>- **改善提案**: コード長を 8 文字 → 16 文字以上(64 bit エントロピー以上)に増強。`crypto.timingSafeEqual` ベースのループに変更(全要素を比較してから結果を返す)。<br>- 関連: B-5-04、既存レポート §21。 |
| F-7-01 | High | 3.3 / 4.C.17 | `frontend/src/lib/` | `audit-log.ts` | IP取得時の `X-Forwarded-For` 信頼性処理、User-Agent長さ制限、書き込み失敗時の動作 | (a) **IP 取得**: 本ファイル `audit-log.ts` は IP を引数として受け取るのみ [audit-log.ts:34](frontend/src/lib/audit-log.ts#L34)。**IP 取得は呼出元(Server Action)** が実施: `staff/login/actions.ts:46` `staff/settings/actions.ts:17` `staff/audit-logs/page.tsx:64` の 3 箇所で<br>```typescript<br>const forwarded = h.get('x-forwarded-for');<br>const ip = forwarded ? forwarded.split(',')[0].trim() : 'unknown';<br>```<br>**信頼性検証なし**: 信頼プロキシリスト判定なし、X-Real-IP 等の優先順位なし、ヘッダ偽装防止なし。<br>(b) **User-Agent 長さ制限**: ❌ **なし** [audit-log.ts:48](frontend/src/lib/audit-log.ts#L48)。値をそのまま `prisma.auditLog.create({ data: { userAgent: params.userAgent } })` に渡す。Prisma の `String` 型は SQLite では TEXT で実質無制限。<br>(c) **書き込み失敗時動作**: try/catch で捕捉、`console.error('Audit log error:', error)` で出力後、本処理は継続(throw しない)[audit-log.ts:50-53](frontend/src/lib/audit-log.ts#L50-L53) | NG | **判定根拠**:<br>- **NG**: ⚠️ **X-Forwarded-For 信頼性検証なし** → リバースプロキシ未経由のリクエストや、信頼境界外のクライアントが任意の IP を `x-forwarded-for: 1.2.3.4` で偽装可能。攻撃者が監査ログ上で「他人の IP 」に偽装してフレームアップが可能。<br>- **NG**: User-Agent 長さ制限なし → 巨大な UA 文字列(数 MB)を送信されると DB 容量を消費する DoS / リソース枯渇リスク。Prisma + SQLite はデフォルト無制限。<br>- **OK**: ログ書き込み失敗時に本処理を継続(可用性優先) ✅<br>- **OK**: `details` は `JSON.stringify` で型安全に格納 ✅<br>- **改善提案**: ① 信頼プロキシリストを環境変数で定義し、X-Forwarded-For は信頼プロキシ経由時のみ採用、② User-Agent を 1024 文字程度で truncate、③ ipAddress も IPv6 上限の 45 文字に truncate<br>- 関連: B-12-01、K-3-05、既存レポート §17。 |
| F-8-01 | High | 3.3 / 4.C.11 | `frontend/src/lib/` | `dto.ts` | 全フォームのZodスキーマ内容、`.strict()` 使用有無、ReDoS 対策、ランタイム型検証 | (a) **`dto.ts` の実態**: ファイルは **Prisma の `select` 句定義のみ**(`safeUserSelect` / `safeAuditLogSelect`)。**Zod スキーマは含まれない** [dto.ts:1-39](frontend/src/lib/dto.ts) 全文。<br>(b) **Zod の使用状況**: `grep -rn 'from .zod.' src/` の結果 **0件**。`package.json` の依存関係にも `zod` は含まれない(D-5-01 参照)。**プロジェクト全体で Zod は使用されていない**。<br>(c) **ランタイム型検証**: フォーム入力に対する型検証は、各 Server Action 内で `formData.get('field') as string` でキャスト + 必要時に `if (!field)` 等の null/empty check のみ。Zod ベースのスキーマ検証は**全く存在しない**。<br>(d) **`.strict()`**: Zod 不採用のため該当なし。<br>(e) **ReDoS 対策**: 正規表現の使用箇所は別途 11章 (K 系)で grep 予定。dto.ts には正規表現なし。 | NG | **判定根拠**:<br>- **NG**: チェックリスト F-8-01 の前提(Zod スキーマ内容)が **崩れている**。`dto.ts` は Prisma DTO 定義のみで、入力検証ライブラリは未採用。<br>- 各 Server Action が独自に `formData.get('x') as string` で取り出して null/空判定するのみ → 型安全性が薄く、未知のフィールドや余剰データの入力検証が無い(Mass Assignment と直結、B-2 系問題)。<br>- 既存レポートでは Zod 不採用は明記されていない(レポート §1 で「全データベースアクセスは Prisma の型安全な API を通じて」とあるが、入力検証層は別問題)。<br>- **改善提案**: Zod を導入し、`safeParseAsync` で各 Server Action 入口で検証。`.strict()` で余剰フィールド拒否。<br>- 関連: B-2-01 / B-2-02 / B-2-03 (Mass Assignment)、F-9-01 (Prisma)、各 actions.ts の入力検証。 |
| F-9-01 | High | 3.3 / 4.C.15 | `frontend/src/lib/` | `prisma.ts` | PrismaClient 初期化パターン(`globalThis`)、`log` オプション、`$queryRaw*`/`$executeRaw*` の使用有無 | (a) **初期化パターン**: `globalThis` 経由のシングルトン [prisma.ts:4-15](frontend/src/lib/prisma.ts#L4-L15) ✅(Next.js Hot Reload 時の重複インスタンス防止)<br>(b) **`log` オプション**: ❌ **未設定**(デフォルトの `error` のみ stderr 出力)<br>(c) **DB url**: `'file:./dev.db'` をハードコード(環境変数経由ではない)[prisma.ts:9](frontend/src/lib/prisma.ts#L9)<br>(d) **adapter**: `@prisma/adapter-better-sqlite3` 使用(Prisma 7 の SQLite 公式アダプタ)<br>(e) **`$queryRawUnsafe` / `$executeRawUnsafe`**: `grep -rn '\\$queryRaw\\|\\$executeRaw' src/` の結果 **0件**。**Raw SQL は使用されていない** ✅ | 部分NG | **判定根拠**:<br>- **OK**: globalThis シングルトン ✅、Raw SQL 不使用 ✅(SQLi 起こりえない)、Prisma 7 公式パターン ✅<br>- **OK寄り**: `log` オプション未設定はデフォルトの `error` のみで、本番では適切。dev で詳細ログが必要なら `query` レベル追加余地あり。<br>- **NG寄り**: DB url が**ハードコード** `'file:./dev.db'`。本番では永続ボリュームの絶対パス(例: `file:/var/lib/arte/data.db`)に切り替える必要があり、現状ではコード変更が必須。Prisma 7 では `prisma.config.ts` で datasource を定義する設計だが、本実装は adapter 直渡しでハードコード。**本番運用時に問題化**(D-3-01 / D-3-02 と連動)。<br>- 既存レポート §1 で「全データベースアクセスは Prisma の型安全な API」と評価され OK。本作業でも矛盾なし(Raw SQL 不使用は確認済み)。<br>- 関連: K-1-04 (queryRawUnsafe grep)、D-3-01 (本番 compose の DB 永続化)、E-1-01。 |

---

## 章サマリ

- **回答済項目数:** 9件
  - Critical: 4件
  - High: 5件
  - Medium: 0件
  - Low: 0件
- **OK 件数:** 0件
- **NG 件数:** 3件 (F-1-01, F-7-01, F-8-01)
- **部分NG 件数:** 6件 (F-2-01, F-3-01, F-4-01, F-5-01, F-6-01, F-9-01)
- **判定不能 件数:** 0件
- **判定不能の主な理由:** 該当なし

## 参照したファイル一覧

- `frontend/src/lib/auth.ts` (全58行)
- `frontend/src/lib/session.ts` (全21行)
- `frontend/src/lib/crypto.ts` (全36行)
- `frontend/src/middleware.ts` (全32行)
- `frontend/src/lib/rate-limit.ts` (全84行)
- `frontend/src/lib/recovery-codes.ts` (全45行)
- `frontend/src/lib/audit-log.ts` (全55行)
- `frontend/src/lib/dto.ts` (全39行)
- `frontend/src/lib/prisma.ts` (全16行)
- `frontend/src/app/staff/login/actions.ts` (40-65行のみ参照、IP 取得部)

## 実行した grep / コマンド一覧

- `grep -rn "import 'server-only'" src/` → **0件**(NG: 全ファイルで未宣言)
- `grep -rn 'from .zod.' src/` → **0件**(Zod 未使用)
- `grep -rn '\$queryRaw\|\$executeRaw' src/` → **0件**(Raw SQL 不使用)
- `grep -rn "x-forwarded-for" src/` → 3件 (login/settings/audit-logs)
- `grep -rn "otplib\|generateSecret" src/` → 12件 (login/settings actions のみ)
- `grep -rn "isTokenUsed\|markTokenAsUsed" src/` → 5件

## 章をまたぐ関連項目

- F-1-01 ⇔ B-1-01 / G-1-01 / G-2-01 (Server Actions の認証チェック呼出)
- F-2-01 ⇔ A-1-02 / B-4-01 (SESSION_SECRET 鍵長と rotation)
- F-3-01 ⇔ A-1-02 / A-1-08 / B-5-01 (ENCRYPTION_KEY と dev.db 履歴)
- F-4-01 ⇔ B-3-01 / B-3-02 (matcher / Edge Runtime)
- F-5-01 ⇔ B-6-01 / 既存§4.3 / §4.6 (Rate Limit の単一プロセス制約)
- F-6-01 ⇔ B-5-04 / 既存§21(リカバリコード)
- F-7-01 ⇔ B-12-01 / K-3-05 / 既存§17 (X-Forwarded-For・UserAgent)
- F-8-01 ⇔ B-2-01 / B-2-02 / B-2-03 (Mass Assignment)、各 actions.ts(Zod 不採用が全フォームに影響)
- F-9-01 ⇔ K-1-04 (queryRawUnsafe grep)、D-3-01 (DB url ハードコード)、E-1-01 (Prisma スキーマ)

## 重要な共通指摘(全 lib に跨る)

- ⚠️ **`src/lib/` 配下の全ファイル(`auth.ts`, `session.ts`, `crypto.ts`, `dto.ts`, `prisma.ts`, `audit-log.ts`, `recovery-codes.ts`, `rate-limit.ts`)で `import 'server-only'` が宣言されていない**。クライアントバンドル混入の防御層が一切無い。本番では明示宣言を推奨(ただし `session.ts` は middleware.ts(Edge)との両立のため例外)。
- ⚠️ **Zod 未採用** — 全 Server Action の入力検証が独自実装に依存。Mass Assignment 系の問題を構造的に発生させやすい。
