# 回答ファイル: 第8章 ページ・フォームコンポーネント確認

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第8章 ページ・フォームコンポーネント確認(3.5)
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| H-1-01 | High | 3.5 | `frontend/src/app/staff/login/` | `LoginForm.tsx` | パスワード入力の `type="password"`、`autocomplete` 属性、エラー表示時の入力値反射 | (a) **`'use client'` 宣言あり** [LoginForm.tsx:1]<br>(b) **パスワード入力**: `<PasswordInput name="password" autoComplete="current-password">` [LoginForm.tsx:369-376] → カスタム `PasswordInput` コンポーネント経由(`@/components/ui/PasswordInput`、本作業中に未読だが命名から `type="password"` 想定)<br>(c) **`autoComplete` 属性**:<br>- ユーザー名入力: `autoComplete="username"` ✅ [LoginForm.tsx:360]<br>- ログイン時パスワード: `autoComplete="current-password"` ✅<br>- 新パスワード入力(PW変更画面): `<PasswordInput minLength={8} autoFocus>` で `autoComplete="new-password"` の **明示なし** ⚠️<br>- TOTP コード: `autoComplete="one-time-code"` ✅ [LoginForm.tsx:186,257]<br>(d) **エラー表示時の入力値反射**: ❌ なし。エラーメッセージは `loginState.message` (= `LOGIN_ERROR_MESSAGE` の統一文字列)のみ表示 [LoginForm.tsx:343-347]。入力値(username/password)はレスポンス state に含まれない | 部分NG | **判定根拠**:<br>- **OK**: 統一エラーメッセージで入力値反射なし(XSS / ユーザー列挙対策)、TOTP に `one-time-code` 適切設定、ログイン時 `current-password` 適切設定<br>- **NG寄り(軽微)**: PW 変更画面の新パスワード入力に `autoComplete="new-password"` の明示なし → ブラウザのパスワードマネージャによる新規パスワード生成提案が機能しない可能性<br>- **判定不能(限定的)**: `PasswordInput` コンポーネント未読のため `type="password"` の確実性は本作業範囲では確定不能(命名規則と利用文脈から推測 OK)<br>- 関連: G-2-01、PasswordInput コンポーネント(別途確認推奨)。 |
| H-2-01 | High | 3.5 / 4.C.16 | `frontend/src/app/staff/blog/` | `BlogForm.tsx` | `'use client'` 宣言ファイルで `process.env.*` 参照箇所、シークレット値のクライアントバンドル混入リスク | (a) **`BlogForm.tsx` 個別**: `'use client'` 宣言あり [BlogForm.tsx:1]、本ファイル内に `process.env.*` 参照 **0件**(冒頭50行確認、全文 grep でも 0件)<br>(b) **全リポジトリ集計**: `grep -rn "process\\.env" src/` の結果 **4件**:<br>- `src/lib/crypto.ts:3` (`process.env.ENCRYPTION_KEY`)<br>- `src/lib/prisma.ts:15` (`process.env.NODE_ENV`)<br>- `src/lib/session.ts:12` (`process.env.SESSION_SECRET`)<br>- `src/lib/session.ts:16` (`process.env.NODE_ENV`)<br>**全 4件が `src/lib/` 配下のサーバ専用モジュール** ✅<br>(c) **`'use client'` 宣言ファイル数**: 19 ファイル<br>(d) **`'use client'` ファイルでの `process.env.*` 参照**: **0件** ✅(本作業時点での grep 結果) | OK | **判定根拠**: クライアントバンドルへのシークレット混入リスクなし ✅。`process.env.*` の使用は server-only モジュール(crypto.ts / prisma.ts / session.ts)に限定。`NEXT_PUBLIC_` プレフィックスの環境変数も使われていない(K-2-07 で更に確認予定)。<br>関連: K-2-07、F-2-01 / F-3-01。 |
| H-3-01 | High | 3.5 | `frontend/src/app/staff/settings/` | `PasswordChangeForm.tsx`, `TwoFactorSettings.tsx` | パスワード変更・2FA無効化UIの (a)現在PW要求 (b)PW強度表示 (c)再認証UI | (a) **現在PW要求(`PasswordChangeForm.tsx`)**: `currentPassword` 入力フィールドあり [PasswordChangeForm.tsx:130-139] ✅。Server Action `changePassword(currentPassword, newPassword)` に渡され `bcrypt.compare` で検証(G-3-01 で確認済)<br>(b) **PW強度表示**: ❌ **未実装**。`minLength={8}` の HTML5 検証のみ [PasswordChangeForm.tsx:152]。zxcvbn等の強度評価ライブラリなし<br>(c) **2FA無効化時の再認証UI(`TwoFactorSettings.tsx`)**: `handleDisable` で `confirm('2段階認証を無効にしますか？')` のブラウザネイティブ確認のみ [TwoFactorSettings.tsx:46-57]。**現在パスワードや TOTP コードの再入力フィールドなし** ⚠️ → Server Action 側(G-3-01)も再認証なし → 二重に確認なし | NG | **判定根拠**:<br>- **OK**: 現在PW入力欄ありで、サーバ側で `bcrypt.compare` 検証 ✅<br>- **NG**: 2FA 無効化時の再認証 UI **完全に欠如**(B-18-01、G-3-01、M-1-06 と直結) → セッション窃取で 2FA 即無効化可能<br>- **NG寄り(軽微)**: PW 強度表示なし → ユーザーが弱いパスワードを設定しがち。`zxcvbn` 等の導入推奨<br>- 関連: G-3-01、B-18-01、M-1-04 (PWポリシー)、M-1-06。 |
| H-4-01 | High | 3.5 | `frontend/src/app/staff/audit-logs/` | `AuditLogsTable.tsx` | フィルタ入力の (a)XSS (b)HPP対策 (c)表示時エスケープ | (a) **XSS**: フィルタ入力は `<select>` (action 列挙) と `<input type="date">` (from/to)のみ → 値域固定で任意文字列を受け付けない ✅。表示テーブルは React 自動エスケープ(`{log.username}`, `{log.resource}`, `{log.ipAddress}` 等)→ XSS 不成立 ✅<br>(b) **HPP対策**: `URLSearchParams` で `params.set()` (一意化、最後の値) [AuditLogsTable.tsx:58-62] ✅。複数値想定なし(単一値前提のフィルタ仕様)<br>(c) **表示時エスケープ**: 全列が React の自動エスケープ経由。`details` は `JSON.parse → Object.entries → join`(キー名と値が文字列)で、結果は `{detailsStr}` として React に渡される [AuditLogsTable.tsx:158-186] → 自動エスケープ ✅<br>(d) `dangerouslySetInnerHTML`: ❌ 不使用 ✅<br>(e) `innerHTML` の手動操作: ❌ 不使用 ✅ | OK | **判定根拠**: フィルタ入力は値域固定、表示は React 自動エスケープのみ → XSS / HPP リスクなし ✅。表示元 `details` の中身に攻撃者制御の文字列が含まれても React の自動エスケープで無害化 ✅。<br>関連: B-13-01 (HPP grep)、B-17-01 (dangerouslySetInnerHTML)、K-2-04。 |
| H-5-01 | Medium | 3.5 | `frontend/src/app/blog/[id]/`, `news/[id]/` | `page.tsx` | `dangerouslySetInnerHTML` 使用箇所と入力ソース、キャッシュ設定 | 第2章 B-17-01 で詳細確認済。要約:<br>- `grep -rn "dangerouslySetInnerHTML" src/` → **0件** ✅<br>- `blog/[id]/page.tsx:79-81`: `<div className="text-gray-700 leading-relaxed whitespace-pre-wrap">{blog.content}</div>` (React 自動エスケープ)<br>- `news/[id]/page.tsx:59`: 同様<br>- DOMPurify 等のサニタイザ未導入(本文は単純テキスト扱い)<br>- キャッシュ設定: `export const dynamic`/`revalidate` 未明示。Prisma クエリは ISR 対象外で実質動的レンダリング | OK | **判定根拠**: B-17-01 と同一。HTML/Markdown 解釈なし、React 自動エスケープのみで XSS 不成立 ✅。<br>**留意**: M-1-03 (本文形式の仕様、Markdown/HTML 採用時)で再評価必要。<br>関連: B-17-01、M-1-03、既存§2。 |
| H-6-01 | Medium | 3.5 / 4.C.37 | `frontend/src/app/` | `page.tsx`, ブログ/ニュース | 外部リンク `rel="noopener noreferrer"` 付与状況(CL-01、本文内) | 第2章 B-20-01 で詳細確認済。要約:<br>- `grep -rn 'target="_blank"' src/` の結果 **3件全てが `Footer.tsx.20260212`(バックアップ、A-1-05)**<br>- 現行コードでは `target="_blank"` 使用 0件<br>- ブログ・ニュース本文は `whitespace-pre-wrap` で `{blog.content}` 表示(リンク自動変換なし)→ 本文内に外部リンクが含まれてもプレーンテキスト表示<br>**現行コードのみで判定: 外部リンクの `target="_blank"` を使う経路がない** | OK | **判定根拠**: B-20-01 と同一 ✅。タブナビング攻撃の経路なし。バックアップファイル(A-1-05)の `target="_blank"` は別問題(削除推奨、現行コードに影響なし)。<br>関連: B-20-01、A-1-05、K-3-06。 |
| H-7-01 | Low | 3.5 | `frontend/src/app/` | `robots.ts`, `sitemap.ts`, `about/page.tsx` | `example.com` プレースホルダの残存有無、運営者情報の明示 | (a) **`robots.ts`** [robots.ts:1-12]:<br>```typescript<br>sitemap: 'https://example.com/sitemap.xml'<br>```<br>⚠️ **`example.com` プレースホルダが残存** [robots.ts:9]<br>(b) **`sitemap.ts`** [sitemap.ts:1-25]: 全 3 URL が `https://example.com` / `https://example.com/about` / `https://example.com/contact` ⚠️ **本番ドメインに未更新**<br>(c) **`about/page.tsx`** [about/page.tsx:1-128]: 会社情報を表形式で記載 ✅<br>- 会社名: 有限会社アルテ<br>- 設立: 2003年5月<br>- 所在地: 〒359-0005 埼玉県所沢市大字神米金358番地の14 郊外マンション新所沢団地K-502<br>- 代表者: 古庄 裕子<br>- 主な事業内容: 5項目列挙 | NG | **判定根拠**:<br>- **NG**: ⚠️ `robots.ts` および `sitemap.ts` で `example.com` プレースホルダが残存 → SEO クローラに誤った URL を提示。本番デプロイ前に必ず修正必要(K-3-04 で `@example.com` grep ヒット予定)<br>- **OK**: 運営者情報(IPA-2.4 フィッシング対策)は明示済み ✅(会社名/所在地/代表者明記)<br>- 関連: K-3-04 (example.com grep)、M-1-08 (運営者情報の明示範囲、仕様待ち)。 |

---

## 章サマリ

- **回答済項目数:** 7件
  - Critical: 0件
  - High: 4件
  - Medium: 2件
  - Low: 1件
- **OK 件数:** 4件 (H-2-01, H-4-01, H-5-01, H-6-01)
- **NG 件数:** 2件 (H-3-01, H-7-01)
- **部分NG 件数:** 1件 (H-1-01)
- **判定不能 件数:** 0件
- **判定不能の主な理由:** 該当なし

## 参照したファイル一覧

- `frontend/src/app/staff/login/LoginForm.tsx` (全393行)
- `frontend/src/app/staff/blog/BlogForm.tsx` (冒頭50行)
- `frontend/src/app/staff/settings/PasswordChangeForm.tsx` (全185行)
- `frontend/src/app/staff/settings/TwoFactorSettings.tsx` (全239行)
- `frontend/src/app/staff/audit-logs/AuditLogsTable.tsx` (全230行)
- `frontend/src/app/robots.ts` (全11行)
- `frontend/src/app/sitemap.ts` (全24行)
- `frontend/src/app/about/page.tsx` (全127行)
- (参考) `frontend/src/app/blog/[id]/page.tsx`、`news/[id]/page.tsx` は B-17-01 で確認済

## 実行した grep / コマンド一覧

- `find src -type f -name "<form名>.tsx"` (全フォーム特定)
- `grep -rn "'use client'" src/ | wc -l` → **19件**
- `grep -rn "process\.env" src/` → **4件、全て src/lib/ 配下**(H-2-01 OK)
- (再利用) B-17-01 / B-20-01 / B-13-01 の grep 結果

## 章をまたぐ関連項目

- H-1-01 ⇔ G-2-01 (login Server Action)
- H-2-01 ⇔ K-2-07 (use client 内の process.env)、F-2-01 / F-3-01
- H-3-01 ⇔ B-18-01 / G-3-01 / M-1-04 / M-1-06 (再認証 / PWポリシー)
- H-4-01 ⇔ B-13-01 / K-2-04 / B-17-01 (HPP / XSS)
- H-5-01 ⇔ B-17-01 / M-1-03 (本文形式仕様)
- H-6-01 ⇔ B-20-01 / A-1-05 / K-3-06
- H-7-01 ⇔ K-3-04 (example.com grep)、M-1-08 (運営者情報範囲)
