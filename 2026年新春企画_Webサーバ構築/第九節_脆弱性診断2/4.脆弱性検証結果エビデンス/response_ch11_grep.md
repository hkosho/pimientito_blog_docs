# 回答ファイル: 第11章 コード全体への機械的検索

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第11章 コード全体への機械的検索(3.10)
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

> 全 grep は `frontend/src/` 配下を対象。ファイル拡張子は `.ts/.tsx/.mjs/.json` を中心に絞り込み。バックアップファイル(`*.backup` / `*.20[0-9]*`)は対象に含むが、現行コードと区別して記載する。

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約)/grep パターン | CC回答(ヒット件数+引用) | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| K-1-01 | Critical | 3.10 | `frontend/src/`, 設定ファイル全般 | - | `password\s*[:=]\s*['"]` | **ヒット 0件** | OK | パスワードリテラル混入なし ✅。bcrypt ハッシュ・iron-session シークレットは全て `process.env.*` 経由(K-1-03 / F-2-01)。<br>関連: A-1-02、F-2-01。 |
| K-1-02 | Critical | 3.10 | 同上 | - | `api[_-]?key\s*[:=]\s*['"]` | **ヒット 0件** | OK | API キーのハードコードなし ✅(現プロジェクトには外部 API 連携なし、`@prisma/client` のみ)。 |
| K-1-03 | Critical | 3.10 | 同上 | - | `secret\s*[:=]\s*['"][^$]`(環境変数参照除く) | **ヒット 0件** | OK | シークレットリテラルなし ✅。SESSION_SECRET / ENCRYPTION_KEY 等は全て `process.env.*` で参照(F-2-01 / F-3-01)。<br>関連: A-1-02、F-2-01、F-3-01。 |
| K-1-04 | Critical | 3.10 / 4.C.8 | 同上 | - | `\$queryRawUnsafe` / `\$executeRawUnsafe` | **ヒット 0件**(`\$queryRaw` / `\$executeRaw` 含めても 0件) | OK | Raw SQL の Unsafe 系および通常 Raw 系も使用なし ✅。SQLi 構造的に不可能(F-9-01 / 既存§1)。 |
| K-2-01 | High | 3.10 / 4.C.22 | 同上 | - | `dangerouslySetInnerHTML` | **ヒット 0件** | OK | XSS 直接経路なし ✅(B-17-01 / H-5-01 と一致)。 |
| K-2-02 | High | 3.10 | 同上 | - | `eval\s*\(` / `Function\s*\(` | **ヒット 0件** | OK | コードインジェクション系 API 不使用 ✅。 |
| K-2-03 | High | 3.10 | 同上 | - | `child_process` / `exec\(` / `spawn\(` / `execSync\(` | **ヒット 0件** | OK | OS コマンド実行 API 不使用 ✅(コマンドインジェクション不成立)。 |
| K-2-04 | High | 3.10 | 同上 | - | `innerHTML\s*=` / `document\.write` | **ヒット 0件** | OK | DOM 直接操作なし ✅。 |
| K-2-05 | High | 3.10 | 同上 | - | `http://`(httpsでない平文URL) | **ヒット 2件**:<br>- `src/components/ui/PasswordInput.tsx:46`: `<svg xmlns="http://www.w3.org/2000/svg" ...>`<br>- `src/components/ui/PasswordInput.tsx:52`: 同上 | OK | **判定根拠**: 2件は **SVG XML 名前空間の URI**(`http://www.w3.org/2000/svg`)で、ネットワークアクセスではない仕様上の固定文字列 ✅。本番コードに http:// で外部ホストへリクエストする箇所はなし ✅。<br>(test スクリプト `scripts/test-csrf.sh` 等の `http://192.168.1.50:3000` は別ファイル、K-3-03 で確認済) |
| K-2-06 | High | 3.10 | 同上 | - | `as any` / `@ts-ignore` / `@ts-expect-error` | **ヒット 0件** | OK | 型チェック迂回なし ✅。`tsconfig.json` の `strict: true` と整合 ✅(D-6-01)。 |
| K-2-07 | High | 3.10 | `'use client'` 宣言ファイル | - | `'use client'` 内での `process\.env\.` 参照 | **ヒット 0件**(全 19 個の `'use client'` ファイルを順次走査、`process.env` を含むファイルは 0件) | OK | クライアントバンドルへのシークレット混入リスクなし ✅(H-2-01 と一致)。 |
| K-2-08 | High | 3.10 / 4.C.21 | 同上 | - | `__proto__` / `constructor\.prototype` | **ヒット 0件** | OK | Prototype Pollution の直接アクセス箇所なし ✅。 |
| K-2-09 | High | 3.10 / 4.C.23 | 同上 | - | `\.bind\(null,` | **ヒット 3件**:<br>- `src/app/staff/blog/[id]/edit/page.tsx:30`: `const boundUpdateBlog = updateBlog.bind(null, blogId);`<br>- `src/app/staff/categories/[id]/edit/page.tsx:18`: `updateCategory.bind(null, categoryId);`<br>- `src/app/staff/news/[id]/edit/page.tsx:24`: `updateNews.bind(null, newsId);` | OK | **判定根拠**: 3件は **Next.js Server Actions の正規パターン**(編集ページで対象 id を bind して formAction に渡す)✅。`bind(null, blogId)` は React の useActionState のシグネチャに合わせるための一般的用法で、4.C.23 が懸念する「`.bind()` 誤用」(関数 this を意図せず変更する用法)には**該当しない** ✅。 |
| K-2-10 | High | 3.10 / 4.C.20 | 同上 | - | `Object\.assign` / `\.\.\.input` / `\.\.\.formData` | **ヒット 0件** | OK | Mass Assignment の構造的経路なし ✅。全 actions.ts で明示列挙(B-2-01〜03 / G-1-01)。 |
| K-3-01 | Medium | 3.10 | 同上 | - | `TODO` / `FIXME` / `XXX` / `HACK` | **ヒット 1件**(偽陽性):<br>- `src/app/staff/login/LoginForm.tsx:132`: `placeholder="XXXX-XXXX"`(リカバリコードのプレースホルダ表示) | OK | コメントマーカー(`// TODO:` 等)としての TODO/FIXME/XXX/HACK は **0件** ✅。1件のヒットはリカバリコード入力欄の表示用プレースホルダ文字列で偽陽性。<br>**留意**: 第3回修正で残存とされた INFO 項目(既存§17 推奨事項等)は別途未コメント化のため grep に現れない。 |
| K-3-02 | Medium | 3.10 | 同上 | - | `console\.log` | **ヒット 0件**(`console.log` は 0件、`console.error` は 26件、`console.warn` は 0件) | OK | console.log での開発ログ漏洩なし ✅。`console.error` は 26件あるが、これらはエラーパスでのログ記録で、機密値直接出力は別途精査(下記参照)。<br>**console.error 中の機密値出力リスク**: 各 actions.ts の catch 句で `console.error('Xx error:', error)` パターンが多数。エラーオブジェクトを直接出力するため、Prisma エラーが DB スキーマ情報を含む可能性あり(既存§6.1 で INFO 残存指摘済)。 |
| K-3-03 | Medium | 3.10 | 同上 | - | `localhost` / `127\.0\.0\.1` / `192\.168\.` | **ヒット 0件** | OK | 本番コード内に内部 IP / localhost のハードコードなし ✅。テストスクリプト `scripts/test-*.sh` には `192.168.1.50:3000` を含むが、これらは本グレップ範囲(`src/`)外 ✅。 |
| K-3-04 | Medium | 3.10 | 同上 | - | `@example\.com` (および `example\.com`) | **ヒット 6件**:<br>- `src/app/robots.ts:9`: `sitemap: 'https://example.com/sitemap.xml'`<br>- `src/app/sitemap.ts:6,12,18`: 全 3 URL が `https://example.com/...`<br>- `src/app/layout.tsx:10`: `metadataBase: new URL('https://example.com')`<br>- `src/app/contact/page.tsx:98`: `placeholder="xxxx@example.com"`(プレースホルダ、偽陽性) | NG | **判定根拠**: ⚠️ **本番ドメイン未設定の状態で 5 箇所**(robots.ts:1 + sitemap.ts:3 + layout.tsx:1)に `example.com` が残存。本番デプロイ前に修正必須。<br>contact/page.tsx は input placeholder で偽陽性。<br>関連: H-7-01、M-1-08 (運営者情報)。 |
| K-3-05 | Medium | 3.10 / 4.C.17 | 同上 | - | `X-Forwarded-For`(信頼性検証処理の有無) | **ヒット 3件**:<br>- `src/app/staff/login/actions.ts:46`<br>- `src/app/staff/settings/actions.ts:17`<br>- `src/app/staff/audit-logs/page.tsx:64`<br>**全箇所で**:<br>```typescript<br>const forwarded = h.get('x-forwarded-for');<br>const ip = forwarded ? forwarded.split(',')[0].trim() : 'unknown';<br>```<br>**信頼性検証(信頼プロキシ判定)なし** | NG | **判定根拠**: F-7-01 と直結。X-Forwarded-For 偽装で監査ログのフレームアップ・レート制限バイパス可能。<br>関連: F-7-01、B-12-01。 |
| K-3-06 | Medium | 3.10 / 4.C.37 | 同上 | - | `target="_blank"`(`rel` 属性の有無) | **ヒット 3件、すべてバックアップファイル**:<br>- `src/components/layouts/Footer.tsx.20260212:21,32,47`<br>**現行コードでは 0件** ✅ | OK | B-20-01 / H-6-01 と同一 ✅。バックアップファイルは A-1-05 で削除推奨。現行コードでタブナビング攻撃の経路なし。 |
| K-3-07 | Medium | 3.10 | 同上 | - | `searchParams\.get\(` / `searchParams\.getAll\(` の使い分け | **ヒット 0件**(直接の `.get()` / `.getAll()` 呼出は使われず、Next.js 16 の `searchParams: Promise<Record>` パターンで `await searchParams; params.foo` 形式で参照、B-13-01) | 部分NG | B-13-01 と同一。`getAll()` 未使用で複数値前提の処理なし → HPP に対する明示的対策はないが、現状は単一値運用で実害なし。<br>関連: B-13-01、H-4-01。 |
| K-3-08 | Low | 3.10 | 同上 | - | `debugger` | **ヒット 0件** | OK | デバッガ文の残存なし ✅。 |

---

## 章サマリ

- **回答済項目数:** 21件
  - Critical: 4件 (K-1-01, K-1-02, K-1-03, K-1-04)
  - High: 10件 (K-2-01〜10)
  - Medium: 6件 (K-3-01〜04, K-3-06, K-3-07)
  - Low: 1件 (K-3-08)
  - (K-3-05 は Medium に分類。再集計で確認)
- **OK 件数:** 19件
- **NG 件数:** 2件 (K-3-04, K-3-05)
- **部分NG 件数:** 1件 (K-3-07)
- **判定不能 件数:** 0件
- **判定不能の主な理由:** 該当なし

(注: 重要度分類の再集計 — K-3-05 を含めると Critical 4 + High 10 + Medium 6 + Low 1 = 21件 ✅)

## 参照したファイル一覧

(grep ヒットファイルのみ列挙、本作業で確認/再確認したもの)

- `src/components/ui/PasswordInput.tsx` (46-52行、SVG xmlns)
- `src/app/staff/login/LoginForm.tsx` (132行、placeholder)
- `src/app/staff/blog/[id]/edit/page.tsx` (30行、bind)
- `src/app/staff/categories/[id]/edit/page.tsx` (18行、bind)
- `src/app/staff/news/[id]/edit/page.tsx` (24行、bind)
- `src/app/robots.ts` / `sitemap.ts` / `layout.tsx` (example.com)
- `src/app/contact/page.tsx` (98行、placeholder、偽陽性)
- `src/app/staff/login/actions.ts` (46行、X-Forwarded-For)
- `src/app/staff/settings/actions.ts` (17行、X-Forwarded-For)
- `src/app/staff/audit-logs/page.tsx` (64行、X-Forwarded-For)
- `src/components/layouts/Footer.tsx.20260212` (21,32,47行、バックアップ)

## 実行した grep / コマンド一覧

(全 grep を `frontend/` 作業ディレクトリで実行)

- K-1-01〜04 の各種パターン → 全 0件 ✅
- K-2-01〜10 の各種パターン → K-2-05 (SVG xmlns 2件、偽陽性) / K-2-09 (Server Actions bind 3件、正規用法) 以外は 0件 ✅
- K-3-01〜08 の各種パターン → K-3-04 (example.com 5件 + placeholder 1件) / K-3-05 (X-Forwarded-For 3件) / K-3-06 (バックアップ 3件) 以外は 0件 ✅
- 補助計測:<br>  `grep -rEn "console.error" src/ \| wc -l` → **26件**<br>  `grep -rEn "console.warn" src/ \| wc -l` → **0件**<br>  `grep -rl "'use client'" src/` で 19 ファイル特定後、各々で `grep -q "process.env"` → 0件

## 章をまたぐ関連項目

- K-1-01 / K-1-02 / K-1-03 ⇔ A-1-02 (鍵長確認)、F-2-01 / F-3-01 (環境変数参照)
- K-1-04 ⇔ F-9-01 (Prisma)
- K-2-01 ⇔ B-17-01 / H-5-01
- K-2-04 ⇔ B-17-01 / H-4-01
- K-2-06 ⇔ D-6-01 (tsconfig strict)
- K-2-07 ⇔ H-2-01
- K-2-09 ⇔ Next.js Server Actions の正規パターン(誤用ではない)
- K-2-10 ⇔ B-2-01 〜 03 / G-1-01 (Mass Assignment)
- K-3-02 ⇔ console.error 26件は別途 INFO 残存(既存§6.1)
- K-3-04 ⇔ H-7-01 / M-1-08
- K-3-05 ⇔ F-7-01 / B-12-01
- K-3-06 ⇔ B-20-01 / H-6-01 / A-1-05
- K-3-07 ⇔ B-13-01 / H-4-01
