# 脆弱性検証ヒアリングチェックリスト ver 1.0

**対象リポジトリ:** `arte-web-site-project`(Next.js 16 + TypeScript + Prisma 7 + SQLite)
**作成日:** 2026-04-23
**ベース資料:** `security_verification_plan_by_claude.md` ver 4.0
**対象AI:** Claude Code(開発環境gitリポジトリ接続)
**利用者:** 開発担当者(人間)、および検証結果を評価する3AI(ChatGPT / Claude / Gemini)

---

## 0. 本チェックリストの目的と使い方

### 0.1 目的

本チェックリストは、以下の3役割の作業結果を**1枚のマトリクス**に集約し、**AIの検証と人間による実機検証の乖離**を明示的にエビデンス化することを目的とする。

- **Claude Code**: リポジトリ内のコード・設定を読んで各質問に回答する(静的検証)
- **人間**: OSSの検証ツールを用い、Dockerコンテナ上の実環境で動的検証を行う
- **3AI(ChatGPT / Claude / Gemini)**: 両結果を突合し、修正要否・追加検証要否を判定する

### 0.2 記録フロー

```
  [Claude Code]                    [人間 + 検証ツール]
       |                                    |
       v                                    v
  「CC回答」欄記入              「テスト結果」欄記入
       |                                    |
       v                                    v
  「評価(AI)」欄                    「評価(人)」欄
       |                                    |
       +--------------+---------------------+
                      v
                 「突合」欄(4象限判定)
                      |
                      v
              [3AIによるレビュー]
                      |
                      v
          「追加検証要否」「修正要否」「備考」
```

### 0.3 4象限判定(「突合」欄)

| 突合コード | AI評価 | 人評価 | 意味 | 原則対応 |
|---|---|---|---|---|
| **OK-OK** | OK | OK | 両者問題なしと判定。 | 原則修正不要。 |
| **OK-NG** | OK | NG | **AIが見落とした脆弱性**。最重要エビデンス。 | 修正必須。AIの癖の記録対象。 |
| **NG-OK** | NG | OK | AIが問題指摘も実機で検出されず。過剰警告または環境依存。 | 修正要否は3AI判定。AIの癖の記録対象。 |
| **NG-NG** | NG | NG | 両者問題ありで一致。 | 修正必須。優先度順に対応。 |
| **OK-未** | OK | 未実施 | AIは問題なし、実機テスト未実施。 | ツール検証を追加するか、AI回答のみで受容するか3AI判定。 |
| **NG-未** | NG | 未実施 | AIが問題指摘、実機テスト未実施。 | ツール検証を追加する方向で3AI判定。 |
| **未-OK** | 未判定 | OK | AI未回答、実機で問題なし。 | AI回答を取得し再判定。 |
| **未-NG** | 未判定 | NG | AI未回答、実機で問題あり。 | 即修正対象。 |

### 0.4 列定義

| 列 | 記入者 | 内容 |
|---|---|---|
| **No.** | - | 連番(カテゴリ別接頭辞付与。例: A-1-01) |
| **重要度** | - | Critical / High / Medium / Low(方針資料 ver 4.0 準拠) |
| **区分** | - | 方針資料の章番号(例: `2.A.1`, `3.3`, `4.C.1`) |
| **タイプ** | - | 証拠区分(`Confirmed` / `Suspected` / `Env-dep`)+ 確認目的記号(S/V/A/I/C/F/N)。凡例は 0.5 参照。 |
| **パス** | - | 確認対象ディレクトリ |
| **ファイル名** | - | 確認対象ファイル(複数の場合は代表1件、他は質問欄に記載) |
| **質問** | - | Claude Code への具体的質問 |
| **CC回答** | Claude Code | 質問への回答(該当コード引用、設定値、有無、判定根拠) |
| **評価(AI)** | Claude Code | OK / NG / 部分NG / 判定不能 |
| **検証方法** | - | 人間が用いる検証ツール名・手順(代表例を提示) |
| **テスト結果** | 人間 | 検証ツール実行結果、手動テスト結果、スクリーンショット参照先等 |
| **評価(人)** | 人間 | OK / NG / 部分NG / 判定不能 / 未実施 |
| **突合** | 3AI | 0.3 の突合コード |
| **追加検証要否** | 3AI | 要 / 不要 / 保留 |
| **修正要否** | 3AI | 要 / 不要 / 保留 |
| **備考** | 3AI | 判定理由、参照すべき他項目番号、仕様決定者への確認が必要な事項等 |

### 0.5 凡例

**証拠区分(タイプ欄の接頭辞):**
- `Confirmed` : 技術レポート・画面分析・リポジトリツリーから既に兆候確認済み
- `Suspected` : 設計上強く疑われるが実装確認が必要
- `Env-dep` : 本番環境依存、repo単体では判定不能

**確認目的記号(タイプ欄の接尾辞):**
- `S` : 機密情報のハードコーディング検査
- `V` : 脆弱な関数・APIの使用検査
- `A` : 認証・認可・セッション実装確認
- `I` : 入力検証・出力エスケープ確認
- `C` : セキュリティ設定・ポリシー確認
- `F` : フレームワーク固有挙動・設定
- `N` : ネットワーク・インフラ層設定

**検証ツール対応表:**

| カテゴリ | 主ツール | 補足 |
|---|---|---|
| DAST(動的) | OWASP ZAP | XSS, CSRF, ディレクトリトラバーサル等の外部診断 |
| SQLi特化 | sqlmap | フォーム入力項目のSQLi検査 |
| ポートスキャン | Nmap | 公開ポート・TLS設定検査 |
| SAST(静的) | Semgrep | TS/JSのコード静的解析 |
| SCA(依存) | Trivy | npm/pnpm依存CVE、Dockerイメージスキャン |
| VA(既知CVE) | OpenVAS | OS・ミドルウェアの包括的診断 |
| grep | ripgrep / git grep | コード全文検索 |
| Git履歴 | `git log --all --full-history` | ファイル履歴・シークレット混入確認 |
| 手動 | curl / ブラウザDevTools | ヘッダ・Cookie・レスポンス検査 |

---

## 1. 確認済み重大論点(2.A / Confirmed)

これらは既に問題の兆候が確認されている項目。**Claude Code への質問は「現状確認と影響範囲の特定」が主目的**であり、「問題の有無」を問うものではない。

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| A-1-01 | Critical | 2.A.1 | Confirmed / S | `frontend/` | `.env.local` | `.env.local` は現在 `.gitignore` で除外されているか。`git log --all --full-history -- frontend/.env.local` で過去コミット履歴にシークレットが含まれているか。 | | | git log 実行 + grep `SESSION_SECRET\|ENCRYPTION_KEY\|SEED_ADMIN_PASSWORD` | | | | | 要(漏洩前提のローテーション) | |
| A-1-02 | Critical | 2.A.1 | Confirmed / S | `frontend/` | `.env.local` | 現在 `.env.local` に格納されている `ENCRYPTION_KEY` / `SESSION_SECRET` / `SEED_ADMIN_PASSWORD` の値の長さと文字種を教えてほしい(値そのものは不要)。 | | | (AI回答のみで判定) | | | | 要 | 要 | 最低長未満ならローテーション時に強化 |
| A-1-03 | Critical | 2.A.1 | Confirmed / S | `frontend/` | `.gitignore` | `.gitignore` に `.env*` / `*.db` / `*.backup` / `*.20[0-9]*` 相当の除外設定が含まれているか。除外されていないパターンを列挙してほしい。 | | | 直接 `.gitignore` 目視 + リポジトリ全体で該当パターンのトラッキング状況を `git ls-files` で確認 | | | | | 要 | |
| A-1-04 | Medium | 2.A.2 | Confirmed / S,V | `frontend/src/app/` | `favicon.ico.backup`, `globals.css.backup`, `page.tsx.backup` | これら3ファイルの内容要約と、現行ファイルとの差分、残存理由の推測を教えてほしい。 | | | 目視確認 + `diff` | | | | | 要(削除判断) | |
| A-1-05 | Medium | 2.A.2 | Confirmed / S,V | `frontend/src/components/layouts/` | `Footer.tsx.20260212`, `Header.tsx.20260206` | 同上。日付サフィックスの命名規則と、これらが残っている運用的理由の推測。 | | | 目視確認 + `diff` | | | | | 要(削除判断) | |
| A-1-06 | High | 2.A.2 / 3.0 | Confirmed / S | `frontend/public/images/` | `background.png.20261302` | 日付形式が異常(`20261302` = 2026年13月02日)。**公開配信領域**にあるため外部からURL推測でアクセス可能。削除対象か、意図的配置か。 | | | curl で `/images/background.png.20261302` へアクセス、外部公開状態の確認 | | | | | 要(即時削除) | 最優先タスク |
| A-1-07 | Low | 2.A.4 / 3.0 | Confirmed / S | `frontend/public/uploads/blog/` | `1770992461005.png`, `1770994779680.png` | これらはテストアップロードか。個人情報・機密情報が含まれていないか。 | | | 画像目視 + EXIF確認(`exiftool`) | | | | | 要(削除判断) | |
| A-1-08 | Medium | 2.A.3 | Confirmed / S | `frontend/` | `dev.db` | `dev.db` のGit履歴(`git log --all --full-history -- frontend/dev.db`)と、含まれているテーブル、レコード件数、機密性のあるデータの有無。 | | | git log + `sqlite3 dev.db ".tables"` + `sqlite3 dev.db ".schema users"` | | | | | 要(履歴から削除判断) | WAL/journal ファイルの扱いも確認 |

---

## 2. Repoから強く疑われる論点(2.B / Suspected)

これらは実装確認が必要な重大論点。Claude Code による静的検証の主戦場。

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| B-1-01 | Critical | 2.B.1 | Suspected / A | `frontend/src/app/staff/**/` | `actions.ts`(全て) | 全 staff 配下の `actions.ts` の各 Server Action 関数の先頭で、`requireAuth()` / `requireAdmin()` が呼ばれているか。呼ばれていない関数があれば関数名とファイル名を列挙してほしい。 | | | Semgrep rule: `requireAuth\|requireAdmin` が各 exported async function の冒頭 N 行以内に存在するかチェック + OWASP ZAP で未認証からのServer Action直接POST | | | | 要 | 要 | |
| B-1-02 | Critical | 2.B.1 | Suspected / A | `frontend/src/app/staff/users/` | `actions.ts` | 本ファイルの全 Server Action で `requireAdmin()` が使用されているか。`requireAuth()` のみの関数は存在するか。 | | | Semgrep + 手動レビュー + OWASP ZAP で一般staff セッションで /staff/users 配下のaction実行試行 | | | | 要 | 要 | 一般staff→admin昇格のリスク |
| B-2-01 | Critical | 2.B.2 | Suspected / V,I | `frontend/src/app/staff/users/` | `actions.ts` | `prisma.user.create` / `update` / `upsert` の `data` 引数が、明示的にフィールド列挙されているか、`...input` や `...formData` でスプレッドしていないか。 | | | ripgrep: `data:\s*\{\s*\.\.\.`, `data:\s*input`, `data:\s*formData` + Semgrep rule | | | | 要 | 要 | ロール昇格脆弱性の可能性 |
| B-2-02 | Critical | 2.B.2 | Suspected / V,I | `frontend/src/app/staff/settings/` | `actions.ts` | 同上。加えて、`where` 条件に現在ログイン中のユーザIDではなく、入力由来の userId を使っている箇所がないか。 | | | ripgrep + Semgrep + OWASP ZAP で他ユーザIDパラメータ注入試験 | | | | 要 | 要 | 他ユーザすり替えリスク |
| B-2-03 | High | 2.B.2 | Suspected / V,I | `frontend/src/app/staff/blog/`, `news/`, `categories/`, `contacts/` | `actions.ts` | 同上の確認を staff 配下の全 CRUD actions に対して実施。 | | | 同上 | | | | 要 | 要 | |
| B-3-01 | Critical | 2.B.3 | Suspected / A,F | `frontend/src/` | `middleware.ts` | `matcher` 配列の内容と、`src/app/staff/` 配下の全ディレクトリが matcher でカバーされているかの突合。matcher で保護されていない staff 配下のパスを列挙してほしい。 | | | 目視 + OWASP ZAP で全 /staff/* パスに未認証アクセスし401/403確認 | | | | 要 | 要 | |
| B-3-02 | Critical | 2.B.3 | Suspected / A,F | `frontend/src/` | `middleware.ts` | middleware が Edge Runtime で実行されるか、Node.js runtime か。`src/lib/crypto.ts`(AES-256-GCM実装)が middleware から間接的に呼ばれる経路があるか。 | | | import graph 解析(`madge` 等) + runtime 宣言確認 | | | | 要 | 要 | Edge Runtime で crypto 呼出は動作不能 |
| B-4-01 | High | 2.B.4 | Suspected / A,C,F | `frontend/src/lib/` | `session.ts` | iron-session 設定の完全な中身。`password` が配列か文字列か、長さ、出所(環境変数か)、全 `cookieOptions` 属性、`maxAge` 秒数。 | | | 目視 + 起動後にブラウザDevTools でCookie属性確認 | | | | | 要(配列化) | パスワードローテーション設計 |
| B-5-01 | High | 2.B.5 | Suspected / V,S,F | `frontend/src/lib/` | `crypto.ts` | AES-256-GCM 実装の IV 生成方法(毎回ランダムか固定か)、auth tag 検証の有無、鍵の導出方法、`import 'server-only'` 宣言の有無。 | | | 目視コードレビュー + Semgrep カスタムルール | | | | | 要(不備があれば) | |
| B-5-02 | High | 2.B.5 | Suspected / V,A | `frontend/src/lib/`, `src/app/staff/login/` | `auth.ts`, `actions.ts` | TOTP 検証処理で、使用済みトークンの再利用防止ロジックがあるか。そのデータ構造(Map/DB)とプロセス再起動時の挙動。 | | | 目視 + 実機でTOTP 2回連続使用 + コンテナ再起動後の挙動試験 | | | | 要 | 要 | プロセス再起動時の時間窓 |
| B-5-03 | High | 2.B.5 | Suspected / V,A | `frontend/src/lib/` | `auth.ts` | TOTP 検証時の比較が定数時間比較か(`crypto.timingSafeEqual` 使用か、`===` か)。 | | | ripgrep: `timingSafeEqual` + 目視 | | | | | 要(不備があれば) | タイミング攻撃対策 |
| B-5-04 | High | 2.B.5 | Suspected / V,A | `frontend/src/lib/` | `recovery-codes.ts` | リカバリコードの生成エントロピー源、ハッシュ化有無、使用済みフラグ管理。 | | | 目視 + Semgrep | | | | | 要(不備があれば) | |
| B-6-01 | High | 2.B.6 | Suspected / A,V | `frontend/src/lib/` | `rate-limit.ts` | データ構造の詳細(`Map<string, {count, resetAt}>` 等)、有効期限処理、対象エンドポイントのリスト、ロックアウト状態のDB永続化の有無。 | | | 目視 + OWASP ZAP Fuzzer で連続POST → コンテナ再起動 → 再度連続POST | | | | 要 | 要 | プロセス再起動で消失する設計上の限界 |
| B-7-01 | High | 2.B.7 | Suspected / V,I | `frontend/src/app/staff/blog/` | `actions.ts` | 画像アップロード処理の (a)許可拡張子/MIMEタイプ (b)マジックナンバー検証 (c)ファイルサイズ上限 (d)保存ファイル名生成(ユーザ入力使用有無) (e)`path.resolve` 境界外チェック (f)sharp リソース制限 を具体的に教えてほしい。 | | | 目視 + OWASP ZAP で拡張子偽装・巨大ファイル・画像爆弾・パストラバーサル試験 | | | | 要 | 要 | CWE-434 |
| B-7-02 | High | 2.B.7 | Suspected / V,I | `frontend/src/app/staff/blog/` | `actions.ts` | `public/uploads/blog/` に保存されたファイルが、`.html` や `.svg` 等HTMLとして解釈され得る拡張子で保存される経路があるか。 | | | 同上 + curl で `.svg` アップロード試験 → Content-Type 応答確認 | | | | 要 | 要 | |
| B-8-01 | High | 2.B.8 | Suspected / A | `frontend/src/app/staff/login/` | `actions.ts` | ログイン成功時に、`session.destroy()` → 新規セッション発行 → `session.save()` の順序で実装されているか。該当コード箇所を引用してほしい。 | | | `scripts/test-session-fixation.sh` 実行 + 手動でログイン前後のCookie値変化確認 | | | | 要 | 要 | |
| B-8-02 | High | 2.B.8 / 4.C.30 / 4.C.32 | Suspected / C | `scripts/` | `test-session-fixation.sh` | 本スクリプトの内容要約、テストシナリオ、期待結果、最新実施結果の記録がリポジトリ内にあるか。 | | | スクリプト実行 + 出力ログ確認 | | | | | 要(結果次第) | |
| B-9-01 | High | 2.B.9 | Suspected / A,F | `frontend/` | `next.config.ts` | `experimental.serverActions.allowedOrigins` の設定値。未設定の場合のデフォルト挙動の理解。 | | | 目視 + 手動で異なるOrigin からのServer Action POST 試験 | | | | 要 | 要(未設定なら設定) | |
| B-9-02 | High | 2.B.9 / 4.C.29 / 4.C.31 | Suspected / C | `scripts/` | `test-csrf.sh` | 本スクリプトの内容要約、テストシナリオ、期待結果、最新実施結果の記録。 | | | スクリプト実行 + 出力ログ確認 | | | | | 要(結果次第) | |
| B-10-01 | High | 2.B.10 | Suspected / I,V,N | `frontend/src/app/contact/` | `actions.ts` | Contact フォーム処理の (a)Zod等スキーマバリデーション (b)XSS対策(管理画面表示時のエスケープ経路) (c)`$queryRaw` 使用有無 (d)メール送信実装有無とヘッダインジェクション対策 (e)レート制限 (f)CAPTCHA/honeypot (g)ログ記録方法 (h)ReDoS 対策。 | | | OWASP ZAP(XSS/CSRF/入力ファズ) + sqlmap(SQLi) + 手動でメールヘッダインジェクション試験 | | | | 要 | 要 | 公開面=最重要 |
| B-11-01 | High | 2.B.11 | Suspected / C,F | `frontend/` | `next.config.ts` | CSP ヘッダの完全な定義(公開用/staff用)。各ディレクティブ(`script-src`, `style-src`, `img-src`, `connect-src`, `frame-ancestors`, `form-action`) の値、`'unsafe-inline'` / `'unsafe-eval'` の有無、nonce 使用の有無、`require-trusted-types-for 'script'` の有無。 | | | 目視 + curl でレスポンスヘッダ確認 + OWASP ZAP CSP Evaluator | | | | 要 | 要(unsafe-inline なら強化) | |
| B-12-01 | High | 2.B.12 | Suspected / A,V | `frontend/src/lib/` | `audit-log.ts` | 監査ログの記録内容、IP取得方法(`X-Forwarded-For` 信頼性検証の有無)、書き込み失敗時の動作、User-Agent長さ制限。 | | | 目視 + 手動でヘッダ偽装試験 | | | | | 要(IP検証なければ) | |
| B-13-01 | High | 2.B.13 | Suspected / V,I | `frontend/src/app/staff/audit-logs/`, `src/app/blog/` | `AuditLogsTable.tsx`, `page.tsx`, `MobileFilterMenu.tsx` | フィルタパラメータのHPP(HTTP Parameter Pollution)対策。`searchParams.get` / `getAll` の使い分け、想定外複数値の扱い。 | | | OWASP ZAP で同名パラメータ複数送信試験 | | | | 要 | 要(対策なければ) | |
| B-14-01 | High | 2.B.14 | Suspected / S,A | `frontend/prisma/` | `schema.prisma` | `users.oneTimePassword` カラムの型・保存形式(平文/ハッシュ/暗号化)、および書き込み側コード。 | | | 目視 + `sqlite3 dev.db "SELECT oneTimePassword FROM users LIMIT 1"` で実データ形式確認 | | | | 要 | 要(平文なら Critical 昇格) | 平文確認時点で Critical |
| B-15-01 | Critical | 2.B.15 | Suspected / A,I | `frontend/src/app/staff/contacts/` | `actions.ts`, 管理画面 | 一括削除処理の (a)トランザクション使用有無 (b)上限件数制限 (c)認可再チェック (d)確認ダイアログの要件。 | | | 目視 + OWASP ZAP で巨大ID配列POST試験 | | | | 要 | 要(上限なければ) | DoS / 誤操作リスク |
| B-16-01 | High | 2.B.16 | Suspected / F,A | `frontend/src/app/staff/` | `page.tsx`, `layout.tsx`, `StaffShell.tsx` | ロール別UIの分岐ロジックが、クライアント側のみで制御されていないか。サーバ側で権限チェック後に描画されているか。`dynamic = 'force-dynamic'` 設定の有無。 | | | 目視 + OWASP ZAP で一般staff セッションでadmin専用UI要素がHTML応答に含まれるか確認 | | | | 要 | 要 | キャッシュによるクロスユーザ漏洩 |
| B-17-01 | High | 2.B.17 | Suspected / V,I | `frontend/src/app/blog/[id]/`, `src/app/news/[id]/` | `page.tsx` | ブログ・ニュース本文の `dangerouslySetInnerHTML` 使用箇所と、入力ソース、サニタイズライブラリ(DOMPurify等)経由の有無。 | | | ripgrep: `dangerouslySetInnerHTML` + OWASP ZAP XSS Fuzz + 手動で `<script>` 注入試験 | | | | 要 | 要(サニタイズなければ) | |
| B-18-01 | High | 2.B.18 | Suspected / A,I | `frontend/src/app/staff/settings/` | `actions.ts`, `TwoFactorSettings.tsx` | 2FA無効化時の再認証要件(現在パスワード要求の有無)、変更通知(メール等)の有無。 | | | 目視 + 手動で2FA無効化操作時の再認証UI確認 | | | | 要 | 要(再認証なければ) | CL-29 |
| B-19-01 | Medium | 2.B.19 | Suspected / V,I | `frontend/src/app/staff/users/new/`, `src/app/staff/users/[id]/` | `CreateUserForm.tsx`, 関連ページ | 初期パスワード・OTP画面の表示期間、自動非表示機能、`Cache-Control: no-store`、`autocomplete="off"`、`@media print` 非表示の有無。 | | | 目視 + 手動で該当画面リロード試験 + curl でレスポンスヘッダ確認 | | | | 要 | 仕様決定と連動 | CL-24-② / CL-25-② |
| B-20-01 | Medium | 2.B.20 | Suspected / V | `frontend/src/app/page.tsx`, ブログ本文 | `page.tsx`, ブログレンダリング | 外部リンク(`<a href="http...">`)への `rel="noopener noreferrer"` 付与状況、`target="_blank"` 使用箇所。 | | | ripgrep: `target="_blank"` + 手動で各リンクの属性確認 | | | | 要 | 要(不備あれば) | CL-01、タブナビング対策 |
| B-21-01 | High | 2.B.21 | Suspected / F,A | `frontend/src/app/staff/**/`, `frontend/` | `next.config.ts`, 各 `page.tsx` | staff 配下ページの fetch キャッシュ設定、`export const dynamic`、`export const revalidate` 設定、`experimental.ppr` の有無。ユーザ固有データが他ユーザにキャッシュ流出するリスクがないか。 | | | ripgrep: `export const dynamic\|revalidate` + 手動で異ユーザ連続アクセス時のレスポンス差異確認 | | | | 要 | 要(設定不備あれば) | クロスユーザ漏洩 |

---

## 3. 本番環境依存論点(2.C / Environment-dependent)

これらは Claude Code だけでは判定できない項目。**CC回答欄には「本番環境設定に依存するため回答不能」が想定され、主に人間によるドキュメント確認・運用担当ヒアリングで埋める。**

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| C-1-01 | High | 2.C.2 | Env-dep / C,N | `frontend/docker/`, `docker/` | `Dockerfile.prod`, `docker-compose.prod.yml`, `.dockerignore` | これらのファイルに基づく本番環境構成で、リバースプロキシ・TLS終端・WAF・CDN の配置前提はドキュメント化されているか。`docs/architecture/` 配下を確認してほしい。 | | | 目視 + 運用担当ヒアリング | | | | 要 | | 運用担当ヒアリング併用 |
| C-2-01 | High | 2.C.3 | Env-dep / N | (リポジトリ外) | - | 本番環境でのリバースプロキシ、TLS終端、CDN/WAF採用、HSTS preload登録予定。リポジトリ内に記述があるか。 | | | 運用担当ヒアリング + `docs/architecture/` 目視 | | | | 要 | | |
| C-3-01 | High | 2.C.4 | Env-dep / N | (リポジトリ外) | - | DDoS/DoS多層防御の現状。アプリ層(アカウントロック・Rate Limit)以外の層の防御策。 | | | 運用担当ヒアリング | | | | 要 | | |
| C-4-01 | High | 2.C.6 | Env-dep / N | (リポジトリ外) | - | Fail2ban 等のホストレベル侵入検知の導入予定。 | | | 運用担当ヒアリング | | | | 要 | | |
| C-5-01 | High | 2.C.7 | Env-dep / N | (リポジトリ外) | - | ホストOSハードニング方針(UFW/iptables, SSH設定, unattended-upgrades, AppArmor, auditd)。 | | | 運用担当ヒアリング + CIS Benchmark 照合 | | | | 要 | | |
| C-6-01 | Medium | 2.C.8 | Env-dep / N | `docker/`, `frontend/` | `docker-compose.prod.yml`, `prisma.config.ts`, `.env.local.example` | 本番構成でのDBファイル配置、パーミッション、バックアップ設計、PostgreSQL移行計画。 | | | 目視 + 運用担当ヒアリング | | | | 要 | | |

---

## 4. 設定・環境ファイル確認(3.1)

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| D-1-01 | Critical | 3.1 | Confirmed / S | `frontend/` | `.env.local.example` | プレースホルダ値が実値(実在しそうなシークレット値)を含んでいないか。全キー名と値のパターンを教えてほしい。 | | | 目視 + ripgrep で `example\|changeme\|your_` 含有確認 | | | | | 要(実値混入なら) | |
| D-2-01 | High | 3.1 / 4.C.7 | Suspected / C,F,N | `frontend/` | `next.config.ts` | `experimental.serverActions.allowedOrigins`、`bodySizeLimit`、`encryptionKey`、`experimental.ppr`、`images.remotePatterns`、`poweredByHeader` の設定値。 | | | 目視 | | | | 要 | 要(不備あれば) | |
| D-3-01 | High | 3.1 | Env-dep / S,C,N | `docker/` | `docker-compose.prod.yml` | 環境変数の注入方法(`env_file`/`environment` の使い分け)、`secrets` 使用有無、ネットワーク分離設計、`cap_drop`/`security_opt`、`ports`/`expose`、`read_only`、`restart` ポリシー。 | | | 目視 + Trivy でコンテナ設定スキャン | | | | 要 | 要(不備あれば) | |
| D-3-02 | High | 3.1 | Suspected / C,N | `frontend/docker/` | `Dockerfile.prod` | ベースイメージの digest 固定有無、非rootユーザ(`nextjs` UID 1001)の実装、multi-stage ビルド、`NODE_OPTIONS`、`init` 指定、不要パッケージの残存。 | | | 目視 + Trivy image scan + `docker run --user` 確認 | | | | 要 | 要(不備あれば) | |
| D-4-01 | High | 3.1 | Suspected / S,C | `frontend/` | `.dockerignore` | `.env*`、`node_modules`、`.git`、`*.backup`、`dev.db*`(WAL/journalを含む)の除外状況。除外されていない潜在的機密の有無。 | | | 目視 + `docker build --no-cache` + `docker run --rm IMAGE ls -la /app` で混入確認 | | | | 要 | 要(不備あれば) | |
| D-5-01 | High | 3.1 / 4.C.35 | Suspected / V | `frontend/` | `package.json`, `pnpm-lock.yaml` | 依存パッケージのうち、既知CVEがあるもの(`pnpm audit` 実行結果)。`pnpm.onlyBuiltDependencies` の設定。`postinstall` スクリプトの有無。 | | | `pnpm audit` + `Trivy fs` + ripgrep `postinstall` | | | | 要 | 要(Critical/High CVEあれば) | |
| D-6-01 | Medium | 3.1 | Suspected / C | `frontend/` | `eslint.config.mjs`, `tsconfig.json` | セキュリティ関連ESLintルール(`no-eval`, `no-implied-eval` 等)の有効化、`tsconfig` の `strict`/`noImplicitAny`/`strictNullChecks`。 | | | 目視 + `tsc --noEmit` 実行 | | | | | | |

---

## 5. データ層(Prisma)確認(3.2)

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| E-1-01 | High | 3.2 | Suspected / C | `frontend/prisma/` | `schema.prisma` | (a)機密フィールドの型と制約 (b)`oneTimePassword`、`sessionSecret`等の保存形式 (c)`authorId` 相当のフィールドの有無(IDOR判定用) (d)`@unique`/`@default` 制約の妥当性。 | | | 目視 | | | | 要 | 要(不備あれば) | |
| E-2-01 | High | 3.2 | Suspected / S,V | `frontend/prisma/` | `seed.ts` | 初期パスワードのハードコーディング有無、bcryptjs の saltRounds 値(4.C.18)、テストユーザの扱い。 | | | 目視 + ripgrep `saltRounds\|bcrypt.hash` | | | | 要 | 要(saltRounds<10 なら) | |
| E-3-01 | Medium | 3.2 | Suspected / C | `frontend/prisma/migrations/` | `*_add_security_features/migration.sql` 他 | 旧adminテーブル残存データの懸念、Down migration欠落時の復元手順、セキュリティ関連カラム追加時のNULL制約の妥当性。 | | | 目視 + `sqlite3 dev.db ".schema"` | | | | | | |

---

## 6. 認証・セキュリティライブラリ層確認(3.3)

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| F-1-01 | Critical | 3.3 / 4.C.3 | Suspected / A,F | `frontend/src/lib/` | `auth.ts` | `requireAuth()` / `requireAdmin()` の内部実装、セッション取得経路、ロール判定ロジック、未認証時の挙動(throw/redirect)、`import 'server-only'` 宣言。 | | | 目視 + Semgrep + OWASP ZAP 未認証アクセス試験 | | | | 要 | 要(不備あれば) | B-1-01 と連動 |
| F-2-01 | Critical | 3.3 / 4.C.1 | Suspected / A,C,F | `frontend/src/lib/` | `session.ts` | iron-session `cookieOptions` 全属性、`password` 出所と長さ、`rotation`設計、`import 'server-only'` 宣言。 | | | 目視 + 起動後ブラウザDevTools でCookie属性確認 | | | | | 要(B-4-01 参照) | |
| F-3-01 | Critical | 3.3 / 4.C.2 | Suspected / V,S,F | `frontend/src/lib/` | `crypto.ts` | AES-256-GCM の (a)IV生成(毎回ランダムか) (b)auth tag検証 (c)鍵導出方法 (d)エラー時の挙動 (e)`import 'server-only'` 宣言。 | | | 目視 + Semgrep カスタムルール | | | | | 要(B-5-01 参照) | |
| F-4-01 | Critical | 3.3 / 4.C.5 | Suspected / A,F | `frontend/src/` | `middleware.ts` | matcher 設定、Edge Runtime 対応、cryptoモジュール呼出の有無、認証失敗時の挙動(redirect先)。 | | | 目視 + OWASP ZAP + `next build` ログで runtime 確認 | | | | | 要(B-3-01/B-3-02 参照) | |
| F-5-01 | High | 3.3 / 4.C.12 | Suspected / A,V | `frontend/src/lib/` | `rate-limit.ts` | データ構造、有効期限処理、競合状態、プロセス再起動時挙動、対象エンドポイントリスト。 | | | 目視 + OWASP ZAP Fuzzer(連続POST→再起動→再試行) | | | | | 要(B-6-01 参照) | |
| F-6-01 | High | 3.3 | Suspected / A,V | `frontend/src/lib/` | `recovery-codes.ts` | リカバリコードのエントロピー源、ハッシュ化有無、使用済みフラグ管理、定数時間比較。 | | | 目視 + Semgrep | | | | | 要(B-5-04 参照) | |
| F-7-01 | High | 3.3 / 4.C.17 | Suspected / A,V,N | `frontend/src/lib/` | `audit-log.ts` | IP取得時の `X-Forwarded-For` 信頼性処理(信頼プロキシ判定有無)、User-Agent長さ制限、書き込み失敗時の動作。 | | | 目視 + 手動でヘッダ偽装試験 | | | | | 要(B-12-01 参照) | |
| F-8-01 | High | 3.3 / 4.C.11 | Suspected / I | `frontend/src/lib/` | `dto.ts` | 全フォームのZodスキーマ内容、`.strict()` 使用有無、ReDoS 対策(`safe-regex` 通過確認等)、ランタイム型検証の徹底。 | | | 目視 + `safe-regex-cli` で全正規表現チェック + OWASP ZAP ファズ | | | | 要 | 要(不備あれば) | |
| F-9-01 | High | 3.3 / 4.C.15 | Suspected / V,F | `frontend/src/lib/` | `prisma.ts` | PrismaClient 初期化パターン(`globalThis`)、`log` オプション、`$queryRaw*`/`$executeRaw*` の使用有無。 | | | 目視 + ripgrep `queryRaw\|executeRaw` | | | | | 要(Unsafe使用あれば) | |

---

## 7. Server Actions確認(3.4)

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| G-1-01 | Critical | 3.4 / 4.C.4 | Suspected / A,I,V | `frontend/src/app/staff/users/` | `actions.ts` | 全 Server Action 関数の (a)認可チェック呼出有無 (b)自己削除・自己降格防止 (c)パスワード生成ロジック (d)入力検証 (e)Mass Assignment対策。 | | | 目視 + Semgrep + OWASP ZAP 一般staff権限で admin用action直接POST試験 | | | | 要 | 要(不備あれば) | B-1-02/B-2-01 と連動 |
| G-2-01 | Critical | 3.4 / 4.C.13 | Suspected / A,I,V | `frontend/src/app/staff/login/` | `actions.ts` | (a)セッション再生成順序 (b)レート制限連携 (c)エラーメッセージ統一(ユーザ列挙対策) (d)2FA検証 (e)アカウントロック機構(試行回数・ロック期間・解除条件、4.C.14)。 | | | 目視 + OWASP ZAP ブルートフォース試験 + 存在/非存在ユーザのエラー差異確認 | | | | 要 | 要(不備あれば) | B-8-01 と連動 |
| G-3-01 | Critical | 3.4 | Suspected / A,I,V | `frontend/src/app/staff/settings/` | `actions.ts` | (a)自分のパスワード変更時の現在PW確認 (b)2FA無効化時の再認証 (c)他ユーザIDへのすり替え防止 (d)Mass Assignment対策。 | | | 目視 + OWASP ZAP 他ユーザIDパラメータ注入試験 | | | | 要 | 要(不備あれば) | B-2-02/B-18-01 と連動 |
| G-4-01 | High | 3.4 | Suspected / A,I,V | `frontend/src/app/staff/blog/` | `actions.ts` | (a)認可チェック (b)IDOR検証 (c)XSS(管理画面反射) (d)ファイル名サニタイズ (e)sharpリソース制限 (f)Prototype Pollution対策(`__proto__`・`constructor.prototype`アクセス)。 | | | 目視 + OWASP ZAP + ripgrep `__proto__\|Object.assign\|\\.\\.\\.input` | | | | 要 | 要(不備あれば) | B-7-01/B-7-02 と連動 |
| G-5-01 | High | 3.4 | Suspected / A,V | `frontend/src/app/staff/contacts/` | `actions.ts` | 一括削除の (a)トランザクション (b)上限件数 (c)認可再チェック。 | | | 目視 + OWASP ZAP で巨大ID配列POST試験 | | | | 要 | 要(B-15-01 参照) | |
| G-6-01 | High | 3.4 / 4.C.10 | Suspected / I,V,N | `frontend/src/app/contact/` | `actions.ts` | B-10-01 と同一範囲の詳細確認。加えて nodemailer 等のメール送信ライブラリ使用状況(4.C.10)。 | | | B-10-01 と同じ | | | | 要 | 要(B-10-01 参照) | |

---

## 8. ページ・フォームコンポーネント確認(3.5)

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| H-1-01 | High | 3.5 | Suspected / I | `frontend/src/app/staff/login/` | `LoginForm.tsx` | パスワード入力の `type="password"`、`autocomplete` 属性、エラー表示時の入力値反射有無。 | | | 目視 + ブラウザDevTools | | | | | | |
| H-2-01 | High | 3.5 / 4.C.16 | Suspected / F | `frontend/src/app/staff/blog/` | `BlogForm.tsx` | `'use client'` 宣言ファイルで `process.env.*` 参照している箇所の有無、シークレット値のクライアントバンドル混入リスク。 | | | ripgrep `'use client'` 含ファイルで `process.env` 検索 + `next build` 後の `.next/static` 内検索 | | | | 要 | 要(混入あれば) | |
| H-3-01 | High | 3.5 | Suspected / I | `frontend/src/app/staff/settings/` | `PasswordChangeForm.tsx`, `TwoFactorSettings.tsx` | パスワード変更・2FA無効化UIの (a)現在PW要求 (b)パスワード強度表示 (c)再認証UI。 | | | 目視 + 手動操作 | | | | | | |
| H-4-01 | High | 3.5 | Suspected / I,V | `frontend/src/app/staff/audit-logs/` | `AuditLogsTable.tsx` | フィルタ入力の (a)XSS (b)HPP対策 (c)表示時エスケープ。 | | | OWASP ZAP + ripgrep `dangerouslySetInnerHTML` | | | | 要 | 要(不備あれば) | B-13-01 と連動 |
| H-5-01 | Medium | 3.5 | Suspected / V,F | `frontend/src/app/blog/[id]/`, `news/[id]/` | `page.tsx` | `dangerouslySetInnerHTML` 使用箇所と入力ソース、キャッシュ設定。 | | | ripgrep + OWASP ZAP XSS Fuzz | | | | 要 | 要(B-17-01 参照) | |
| H-6-01 | Medium | 3.5 / 4.C.37 | Suspected / V | `frontend/src/app/` | `page.tsx`, ブログ/ニュース | 外部リンク `rel="noopener noreferrer"` 付与状況(CL-01、本文内)。 | | | ripgrep `target="_blank"` + 手動確認 | | | | 要 | 要(B-20-01 参照) | |
| H-7-01 | Low | 3.5 | Suspected / C | `frontend/src/app/` | `robots.ts`, `sitemap.ts`, `about/page.tsx` | `example.com` プレースホルダの残存有無、運営者情報(会社名・所在地・連絡先)の明示(IPA-2.4フィッシング対策)。 | | | ripgrep `example.com` + 目視 | | | | | 仕様決定連動 | |

---

## 9. スクリプト・既存レポート確認(3.6 / 3.7)

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| I-1-01 | High | 3.6 | Suspected / S,A | `frontend/scripts/` | `reset-password.ts` | パスワードリセット処理の引数取扱い、実行権限、パスワード生成方法、ログ出力内容。 | | | 目視 | | | | | | |
| I-2-01 | High | 3.6 | Suspected / S | `frontend/scripts/` | `list-users.ts` | 出力内容に機密情報(ハッシュ/メールアドレス等)が含まれるか。 | | | 目視 + 実行結果確認 | | | | | 要(機密含有あれば) | |
| I-3-01 | High | 3.7 / 4.C.33 | Confirmed / C | `security-report/` | `security-report.md`, `security-review-response.md`, `security-test-results.md` | 既存レポートの指摘事項と対応状況。今回の診断範囲と重複する項目の洗い出し。 | | | 目視 | | | | 要 | | 重複回避のため必ず先に確認 |

---

## 10. バックアップ・不要ファイル確認(3.9)

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| J-1-01 | Medium | 3.9 | Confirmed / S,V | `frontend/src/app/`, `frontend/src/components/layouts/` | `*.backup`, `*.20260212`, `*.20260206` | 各ファイルと現行ファイルの差分、旧コードに残存する機密情報・APIキー・デバッグコード。 | | | 目視 + `diff` + ripgrep `key\|secret\|password\|token` | | | | 要 | 要(全削除判断) | A-1-04/A-1-05 と連動 |

---

## 11. コード全体への機械的検索(3.10)

これらは **Claude Code 側での grep 結果一覧**を回答として記録する。ヒット件数が0でも記録対象。

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| K-1-01 | Critical | 3.10 | Suspected / S | `frontend/src/`, 設定ファイル全般 | - | `password\s*[:=]\s*['"]` にマッチする箇所のリスト。 | | | Semgrep `generic.secrets.*` + ripgrep | | | | 要 | 要(ヒットあれば) | |
| K-1-02 | Critical | 3.10 | Suspected / S | 同上 | - | `api[_-]?key\s*[:=]\s*['"]` にマッチする箇所。 | | | 同上 | | | | 要 | 要(ヒットあれば) | |
| K-1-03 | Critical | 3.10 | Suspected / S | 同上 | - | `secret\s*[:=]\s*['"][^$]`(環境変数参照除く)のヒット箇所。 | | | 同上 | | | | 要 | 要(ヒットあれば) | |
| K-1-04 | Critical | 3.10 / 4.C.8 | Suspected / V | 同上 | - | `$queryRawUnsafe` / `$executeRawUnsafe` のヒット箇所。 | | | ripgrep | | | | 要 | 要(使用あれば) | |
| K-2-01 | High | 3.10 / 4.C.22 | Suspected / V | 同上 | - | `dangerouslySetInnerHTML` のヒット箇所とその入力ソース。 | | | ripgrep + 呼出元トレース | | | | 要 | 要(サニタイズなければ) | |
| K-2-02 | High | 3.10 | Suspected / V | 同上 | - | `eval\s*\(` / `Function\s*\(` のヒット箇所。 | | | ripgrep | | | | 要 | 要(使用あれば) | |
| K-2-03 | High | 3.10 | Suspected / V | 同上 | - | `child_process` / `exec\(` / `spawn\(` / `execSync\(` のヒット箇所。 | | | ripgrep | | | | 要 | 要(ユーザ入力連動あれば) | |
| K-2-04 | High | 3.10 | Suspected / V | 同上 | - | `innerHTML\s*=` / `document\.write` のヒット箇所。 | | | ripgrep | | | | 要 | 要(使用あれば) | |
| K-2-05 | High | 3.10 | Suspected / N | 同上 | - | `http://` (httpsでない平文URL) のヒット箇所(テストコード除く)。 | | | ripgrep + 手動でテストコードを除外 | | | | 要 | 要(本番コードあれば) | |
| K-2-06 | High | 3.10 | Suspected / V | 同上 | - | `as any` / `@ts-ignore` / `@ts-expect-error` のヒット箇所と件数。 | | | ripgrep | | | | | 要(多数あれば個別精査) | 型チェック迂回 |
| K-2-07 | High | 3.10 | Suspected / S,F | `'use client'` 宣言ファイル | - | `'use client'` 宣言ファイル内での `process\.env\.` 参照箇所。 | | | ripgrep 段階的(先に'use client'のファイルを特定→各ファイル内検索) | | | | 要 | 要(シークレット参照あれば) | H-2-01 と連動 |
| K-2-08 | High | 3.10 / 4.C.21 | Suspected / V | 同上 | - | `__proto__` / `constructor\.prototype` のヒット箇所。 | | | ripgrep | | | | 要 | 要(使用あれば) | Prototype Pollution |
| K-2-09 | High | 3.10 / 4.C.23 | Suspected / F | 同上 | - | `\.bind\(null,` のヒット箇所(Server Actions `.bind()` 誤用)。 | | | ripgrep | | | | 要 | 要(使用あれば) | |
| K-2-10 | High | 3.10 / 4.C.20 | Suspected / V | 同上 | - | `Object\.assign` / `\.\.\.input` / `\.\.\.formData` のヒット箇所。 | | | ripgrep | | | | 要 | 要(Prisma `data` 直結あれば) | Mass Assignment |
| K-3-01 | Medium | 3.10 | Suspected / C | 同上 | - | `TODO` / `FIXME` / `XXX` / `HACK` のヒット箇所と件数。 | | | ripgrep | | | | | | |
| K-3-02 | Medium | 3.10 | Suspected / S | 同上 | - | `console\.log` のヒット箇所(本番で残ると情報漏洩)。 | | | ripgrep | | | | 要 | 要(機密値出力あれば) | |
| K-3-03 | Medium | 3.10 | Suspected / S | 同上 | - | `localhost` / `127\.0\.0\.1` / `192\.168\.` のヒット箇所(本番コードに残存していないか)。 | | | ripgrep | | | | | 要(本番コードあれば) | |
| K-3-04 | Medium | 3.10 | Suspected / S | 同上 | - | `@example\.com` のヒット箇所。 | | | ripgrep | | | | | 要(本番コードあれば) | |
| K-3-05 | Medium | 3.10 / 4.C.17 | Suspected / N | 同上 | - | `X-Forwarded-For` のヒット箇所と、信頼性検証処理の有無。 | | | ripgrep + コードレビュー | | | | 要 | 要(検証なければ) | F-7-01 と連動 |
| K-3-06 | Medium | 3.10 / 4.C.37 | Suspected / V | 同上 | - | `target="_blank"` のヒット箇所と `rel` 属性の有無。 | | | ripgrep | | | | 要 | 要(rel欠落あれば) | H-6-01 と連動 |
| K-3-07 | Medium | 3.10 | Suspected / V | 同上 | - | `searchParams\.get\(` / `searchParams\.getAll\(` の使い分け状況(HPP対策)。 | | | ripgrep | | | | 要 | 要(getAll未使用で複数値前提あれば) | |
| K-3-08 | Low | 3.10 | Suspected / C | 同上 | - | `debugger` のヒット箇所。 | | | ripgrep | | | | | 要(本番コードあれば) | |

---

## 12. 追加の技術スタック固有観点(4.C補完)

方針資料 4.C のうち、上記で未カバーの項目を列挙する。

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| L-1-01 | Medium | 4.C.18 | Suspected / A | `frontend/prisma/`, `src/lib/` | `seed.ts`, `auth.ts` 等 | bcryptjs の `saltRounds` 値。全呼出箇所で一致しているか。 | | | ripgrep `bcrypt.hash\|saltRounds` | | | | | 要(<10 なら) | |
| L-2-01 | Medium | 4.C.19 | Suspected / A | `frontend/src/lib/` | `auth.ts` | otplib の設定値(`window`, `algorithm`, `digits`, `step`)。 | | | 目視 + `otplib` デフォルト値との差異確認 | | | | | 要(設定不備あれば) | |
| L-3-01 | High | 4.C.24 | Suspected / A,F | `frontend/src/app/` | `**/route.ts` | `route.ts` ファイル(Route Handler)の存在確認と、各ファイルの認証チェック実装有無。 | | | `find src/app -name "route.ts"` + 各ファイル目視 | | | | 要 | 要(認証漏れあれば) | Server Actions 以外のAPIエンドポイント |
| L-4-01 | Medium | 4.C.26 | Suspected / F | `frontend/src/lib/` | `auth.ts`, `session.ts` | React 19 taint API(`taintUniqueValue`, `taintObjectReference`)の使用有無。 | | | ripgrep `taintUniqueValue\|taintObjectReference` | | | | | 保留(未使用なら) | |
| L-5-01 | Medium | 4.C.34 | Confirmed / C | `docs/` | `architecture/`, `process/`, `failures/`, `ai-sessions/` 配下 | 各ディレクトリに含まれるドキュメントのリストと、セキュリティ関連記述の要約。運用手順・インシデント対応・secretsローテーション手順の記載の有無。 | | | 目視 | | | | 要 | | |
| L-6-01 | Medium | 4.C.36 | Env-dep / C | (CI/CD) | - | コンテナイメージの脆弱性スキャン(Trivy等)がCI/CDパイプラインに組み込まれているか。 | | | 運用担当ヒアリング + `.github/workflows/` 目視 | | | | 要 | | |
| L-7-01 | Medium | 4.C.38 | Suspected / A | `frontend/src/app/staff/audit-logs/`, `src/lib/` | `audit-log.ts`, actions.ts | 監査ログの削除・エクスポート機能の有無(WORM原則違反の検出)。 | | | ripgrep `delete.*audit\|export.*audit` + 目視 | | | | 要 | 仕様決定連動(4.A.7) | |

---

## 13. 仕様決定者への確認が必要な項目(4.A / 参考)

これらは Claude Code では回答不能。**CC回答欄は空欄とし、「仕様決定者確認中」と記載**。人間が仕様決定者から回答を得た後、3AI判定へ進む。

| No. | 重要度 | 区分 | タイプ | パス | ファイル名 | 質問 | CC回答 | 評価(AI) | 検証方法 | テスト結果 | 評価(人) | 突合 | 追加検証要否 | 修正要否 | 備考 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| M-1-01 | High | 4.A.1 | - / 仕様 | - | - | ブログ記事の author 単位の権限分離設計(staff全員が全記事を編集か、author別スコープか)。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | 仕様次第 | IDOR 判定に影響 |
| M-1-02 | High | 4.A.2 | - / 仕様 | - | - | メール送信機能の実装方針(未実装のまま運用か、将来実装予定か)。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | B-10-01 に影響 |
| M-1-03 | High | 4.A.3 | - / 仕様 | - | - | ブログ本文・ニュース本文の入力形式(Markdown / HTML / リッチエディタ)。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | B-17-01 に影響 |
| M-1-04 | High | 4.A.4 | - / 仕様 | - | - | パスワードポリシー(最小文字数、使い回し禁止、有効期限、初期PW失効条件)。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | |
| M-1-05 | Medium | 4.A.5 | - / 仕様 | - | - | 初期PW・OTP画面表示の自動非表示機能の要否。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | B-19-01 に影響 |
| M-1-06 | High | 4.A.6 | - / 仕様 | - | - | 2FA無効化時の再認証・メール通知の要否。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | B-18-01 に影響 |
| M-1-07 | Medium | 4.A.7 | - / 仕様 | - | - | 監査ログの削除・エクスポート機能の要否(WORM原則適用可否)。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | L-7-01 に影響 |
| M-1-08 | Medium | 4.A.8 | - / 仕様 | - | - | 運営者情報のサイト上への明示範囲(IPA-2.4)。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | H-7-01 に影響 |
| M-1-09 | High | 4.A.9 | - / 仕様 | - | - | 本番環境のDB選定(SQLiteのままか、PostgreSQL移行予定か)。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | 1.A.1.6 等の将来観点に影響 |
| M-1-10 | High | 4.A.10 | - / 仕様 | - | - | 一括削除機能の上限値(CL-10 の対象ID配列最大件数)。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | B-15-01 に影響 |
| M-1-11 | Medium | 4.A.11 | - / 仕様 | - | - | 退職者アカウント無効化・権限変更承認フローの要否。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | |
| M-1-12 | Medium | 4.A.12 | - / 仕様 | - | - | Credential Stuffing 対策としての漏洩パスワードDB(HIBP)連携の要否。 | (回答不能) | - | 仕様決定者確認 | | | | 要 | | |

---

## 14. 集計・判定サマリ

### 14.1 突合結果の集計(記入欄)

全項目の記入完了後、以下のサマリを作成する。

| 突合コード | 件数 | 主な項目No. | コメント |
|---|---|---|---|
| OK-OK | | | |
| OK-NG | | | **最重要エビデンス**: AIが見落とした項目の分析 |
| NG-OK | | | AIの過剰警告 / 環境依存の分析 |
| NG-NG | | | 修正必須項目の件数 |
| OK-未 | | | 実機テスト未実施の件数 |
| NG-未 | | | 実機テスト未実施かつAI問題指摘の件数 |
| 未-OK | | | AI未回答の件数 |
| 未-NG | | | AI未回答かつ実機で問題検出の件数 |

### 14.2 修正対象リスト(3AIによる最終判定)

「修正要否=要」のすべての項目を以下に転記する。

| No. | 重要度 | 区分 | 概要 | 修正方針 | 対応優先度 | 担当(Claude Code / 人間 / 仕様決定者) |
|---|---|---|---|---|---|---|
| | | | | | | |

### 14.3 追加検証対象リスト(3AIによる最終判定)

「追加検証要否=要」のすべての項目を以下に転記する。

| No. | 重要度 | 区分 | 概要 | 追加検証方法 | 実施担当 |
|---|---|---|---|---|---|
| | | | | | |

### 14.4 ヒアリングチェックリスト自体の漏れ分析

実験趣旨(方針資料「期待する結論」)に従い、本チェックリストに含まれていなかった項目が検証工程で発見された場合、以下に記録する。

| 発見日 | 発見項目 | 発見方法 | 本チェックリストに含まれなかった理由 | 次版での追加方針 |
|---|---|---|---|---|
| | | | | |

### 14.5 AIの癖・傾向の記録(実験結果の中核)

突合コード OK-NG / NG-OK の項目について、AIの判定傾向を以下に記録する。これが実験の主要アウトプット。

| 突合コード | 項目No. | AIの判定 | 実機結果 | AIの癖の分類 | コメント |
|---|---|---|---|---|---|
| OK-NG | | | | 例:静的解析の限界 / 実行時挙動への推測不足 / 設定値の内容推測 / キャッシュ動作の見落とし | |
| NG-OK | | | | 例:過剰警告 / 一般論からの類推 / 実環境の防御層未考慮 / ドキュメントにない前提の想定 | |

---

## 15. 運用ルール

### 15.1 記入順序

1. **第1段階: Claude Code による記入**
   - 本チェックリスト全体を Claude Code に渡し、「CC回答」「評価(AI)」欄を埋める
   - 回答不能な項目(本番環境依存・仕様確定)はその旨を明記
   - Claude Code が参照したファイルパスとコミットハッシュを冒頭に記録する
2. **第2段階: 人間による実機検証**
   - 「検証方法」欄の指定に従い、OSSツール(OWASP ZAP / sqlmap / Nmap / Semgrep / Trivy / OpenVAS)で検証
   - 「テスト結果」「評価(人)」欄を埋める
   - 検証未実施の項目は「未実施」と明記(空欄にしない)
3. **第3段階: 3AI による突合判定**
   - 両欄が埋まった項目について、「突合」「追加検証要否」「修正要否」「備考」欄を埋める
   - 判定根拠を備考欄に必ず記録(「3AIの意見が割れた場合は人間が最終判断」)
4. **第4段階: 14章サマリの作成**
   - 集計、修正対象リスト、AIの癖の記録を作成
   - ブログ記事化・ドキュメント化する

### 15.2 項目の追加・変更ルール

- 検証工程で新たな観点が発見された場合、14.4 に記録し、**本ドキュメントは改版しない**
- 次版(ver 1.1 以降)で反映
- 改版時は版履歴を追記

### 15.3 未完了項目の扱い

- ヒアリング期間内に完了しない項目は、「評価」欄を「判定不能」または「未実施」とし、突合欄は「未」系コードで記録
- 中途半端な OK 判定をしないことが実験エビデンスとして重要

### 15.4 コンテキスト長制限への対応

本チェックリストは全項目で約100項目を超える。Claude Code のコンテキスト長制限により、一度に処理できない場合は、**カテゴリ単位(1章〜13章)での分割処理**を推奨する。分割時は冒頭の 0章(凡例・列定義)を必ず添付する。

---

## 16. 参考資料

- `security_verification_plan_by_claude.md` ver 4.0(本チェックリストのベース)
- IPA「安全なウェブサイトの作り方 改訂第7版」
- IPA「セキュリティ実装チェックリスト」
- IPA「安全なSQLの呼び出し方」
- OWASP ZAP, sqlmap, Nmap, Semgrep, Trivy, OpenVAS の各公式ドキュメント
- 「体系的に学ぶ 安全なWebアプリケーションの作り方」徳丸浩著(SB Creative)

---

## 17. 版履歴

| 版 | 日付 | 変更概要 |
|---|---|---|
| 1.0 | 2026-04-23 | 初版。方針資料 ver 4.0 に基づき、2.A / 2.B / 2.C / 3.0 / 3.1 / 3.2 / 3.3 / 3.4 / 3.5 / 3.6 / 3.7 / 3.9 / 3.10 / 4.A / 4.C から約100項目を抽出、16列マトリクス化。 |

---

以上。
