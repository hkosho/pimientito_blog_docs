# 回答ファイル: 第1章 確認済み重大論点(2.A / Confirmed)

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第1章 確認済み重大論点
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| A-1-01 | Critical | 2.A.1 | `frontend/` | `.env.local` | `.env.local` は `.gitignore` で除外されているか。git history にシークレットが含まれているか | (a) **`.env.local` は除外済み**: `git check-ignore -v frontend/.env.local` → `frontend/.gitignore:34:.env*	frontend/.env.local` で適合。<br>(b) **git history**: `git log --all --full-history -- frontend/.env.local` の結果は **0件**(ファイルは履歴に存在しない)。<br>(c) `.env.local` は現在もリポジトリ作業ツリーに存在(193バイト、2026-02-16 更新)。 | OK | **判定根拠**:<br>- `frontend/.gitignore:34: .env*` で `.env.*` 系を一括除外。<br>- 全コミット履歴に `.env.local` は無い。<br>- 既存レポート §10.1 / §10.2 で確認済み事項を再確認。<br>- ⚠️補足: ルート `.gitignore` は `.env*.local` および `.env` を別途設定 [.gitignore](.gitignore)。重複だが矛盾はない。<br>- 関連: D-1-01 (`.env.local.example` の乖離)。 |
| A-1-02 | Critical | 2.A.1 | `frontend/` | `.env.local` | `ENCRYPTION_KEY` / `SESSION_SECRET` / `SEED_ADMIN_PASSWORD` の値の長さと文字種(値そのものは不要) | `.env.local` 全3キー。値は伏せ字で長さのみ集計:<br>- `ENCRYPTION_KEY`: 値の長さ **64 文字**(おそらく hex で 32 バイト = 256 bit、AES-256 の鍵長として適切)<br>- `SESSION_SECRET`: 値の長さ **64 文字**(iron-session のドキュメントは最低 32 文字推奨、64 文字は十分)<br>- `SEED_ADMIN_PASSWORD`: 値の長さ **12 文字**<br>(文字種は本回答では明記しない=実値推測のヒントにしないため。) | 部分NG | **判定根拠**:<br>- ENCRYPTION_KEY 64文字 ✅(AES-256-GCM 用、`crypto.ts` の鍵長要件を満たす)<br>- SESSION_SECRET 64文字 ✅(iron-session 最低32文字を上回る)<br>- SEED_ADMIN_PASSWORD **12文字は最低限のレベル**。memory `Initial login: ***MASKED*** / ***MASKED***` (12文字、英数記号)が実値とすればパスワード強度として十分とは言えない(辞書攻撃耐性は中程度)。ただし「初期パスワードであり、初回ログイン時に強制変更」(memory: `must change password + setup 2FA on first login`)の運用ならリスクは大幅低下。<br>- 引用元: `awk '{print "len="length($2)}' .env.local` の結果(値そのものは引用しない=指示書 6.1 の制約)。<br>- 関連: M-1-04 (パスワードポリシー、仕様決定者確認)。 |
| A-1-03 | Critical | 2.A.1 | `frontend/` | `.gitignore` | `.gitignore` に `.env*` / `*.db` / `*.backup` / `*.20[0-9]*` 相当の除外設定が含まれているか。除外されていないパターン | **`frontend/.gitignore`** [frontend/.gitignore:1-40] の主要設定:<br>- `.env*` ✅(line 34)<br>- `*.pem` ✅<br>- `node_modules`, `.next`, `coverage`, `*.tsbuildinfo` 等の標準的な除外<br>- **`*.db` / `*.db-journal` / `*.db-wal` / `*.db-shm` の除外**: **未設定**(NG)<br>- **`*.backup` の除外**: **未設定**(NG)<br>- **`*.20[0-9]*` (日付サフィックス)の除外**: **未設定**(NG)<br><br>**ルート `.gitignore`** [.gitignore]:<br>- `.env*.local`、`.env` 設定済み(重複)<br>- 同様に `*.db` / `*.backup` / 日付サフィックスの除外設定なし<br><br>**実害(`git ls-files` で確認)**:<br>- ✗ `frontend/dev.db` が **トラッキングされている**(下記 A-1-08 参照、4コミット履歴あり)<br>- ✗ `frontend/src/app/favicon.ico.backup`<br>- ✗ `frontend/src/app/globals.css.backup`<br>- ✗ `frontend/src/app/page.tsx.backup`<br>- ✗ `frontend/src/components/layouts/Footer.tsx.20260212`<br>- ✗ `frontend/src/components/layouts/Header.tsx.20260206`<br>- ✗ `frontend/public/images/background.png.20261302`<br>- ✗ `frontend/public/uploads/blog/1770992461005.png`<br>- ✗ `frontend/public/uploads/blog/1770994779680.png` | NG | **判定根拠**: 単に「除外パターンが無い」だけでなく、**実際にバックアップファイル・dev.db・テストアップロード画像が git に追跡されている** ため、`.gitignore` の不備が表面化している:<br>- `*.db` / `dev.db*` の除外なし → A-1-08 と直結(dev.db が履歴に含まれる)。<br>- `*.backup` 除外なし → A-1-04 / J-1-01 と直結。<br>- 日付サフィックス(`*.[12][09]*`、`*.20*`、`*.2026*`)の除外なし → A-1-05 / A-1-06 と直結。<br>- `public/uploads/` 配下を除外していない → A-1-07 と直結。<br>- 既存レポートでは未指摘の項目を含む(public/uploads/ の git 追跡は新規発見)。<br>- 関連: D-4-01 (`.dockerignore` も同様に不備)、A-1-04 〜 A-1-08。 |
| A-1-04 | Medium | 2.A.2 | `frontend/src/app/` | `favicon.ico.backup`, `globals.css.backup`, `page.tsx.backup` | これら3ファイルの内容要約と現行ファイルとの差分、残存理由 | (a) **`favicon.ico.backup`** (25,931 バイト、2026-01-16): バイナリ ico ファイル。現行 `favicon.ico` はリポジトリ内別所(`/app/favicon.ico` 等)に存在予定。<br>(b) **`globals.css.backup`** (488 バイト、2026-02-05): CSS バックアップ。<br>(c) **`page.tsx.backup`** (3,117 バイト、2026-02-05): トップページ TSX バックアップ。<br>**git ls-files で全3ファイルが Git 追跡対象**として確認された。<br>**残存理由(推測)**: A-1-05 / A-1-06 と同パターンで、過去のリファクタ・デザイン変更時のバックアップが削除されないまま残存。意図的な配置とは考えにくい。 | NG | **判定根拠**: 既存レポート §10 / 既存指摘でも同様。本作業時点でも未削除:<br>- バックアップ自体が機密情報を含むかは未読(コードレベルでの差分確認は J-1-01 / 10章で詳細実施)。<br>- `*.backup` を `.gitignore` で除外していないため(A-1-03)、追加するとリポジトリから消えるが既履歴は残る。<br>- 関連: A-1-03、A-1-05、J-1-01、D-4-01(.dockerignore も不備)。 |
| A-1-05 | Medium | 2.A.2 | `frontend/src/components/layouts/` | `Footer.tsx.20260212`, `Header.tsx.20260206` | 同上。日付サフィックスの命名規則と残存理由 | (a) **`Footer.tsx.20260212`** (2026-02-12 サフィックス): Footer コンポーネントの 2026-02-12 時点バックアップ。<br>(b) **`Header.tsx.20260206`** (2026-02-06 サフィックス): Header コンポーネントの 2026-02-06 時点バックアップ。<br>命名規則: `<元ファイル名>.YYYYMMDD`(8桁日付)。本作業範囲内の他のサフィックス例として `frontend/public/images/background.png.20261302`(A-1-06、不正な日付)あり。<br>**git ls-files で全2ファイルが Git 追跡対象**。 | NG | **判定根拠**:<br>- 古いコードが Git 履歴と並列に作業ツリーに残ることで、混乱・機密情報残存・依存関係不整合のリスク。Git 自体が履歴を保持しているためバックアップとして冗長。<br>- 内容の機密情報・APIキーの有無は J-1-01 で詳細精査予定。<br>- 関連: A-1-03、A-1-04、J-1-01、D-4-01。 |
| A-1-06 | High | 2.A.2 / 3.0 | `frontend/public/images/` | `background.png.20261302` | 日付形式が異常(`20261302` = 2026年13月02日)。**公開配信領域**にあるため外部からURL推測でアクセス可能。削除対象か、意図的配置か | ファイル: PNG 画像 1536×1024、1,402,435 バイト、2026-02-03 mtime。日付サフィックス `20261302` は不正(13月)→ 命名ミスと推測。<br>**git ls-files で `frontend/public/images/background.png.20261302` が Git 追跡対象**。<br>**配信経路**: Next.js の `public/` 配下は静的配信されるため、URL `/images/background.png.20261302` で公開アクセス可能(本作業では curl 実行は控える=指示書 6.1 動的検証は別工程)。<br>意図的配置とする根拠は無い(運用ドキュメントに `*.20261302` 形式の言及なし)。 | NG | **判定根拠**: ⚠️ **最優先削除対象**:<br>- 公開ディレクトリ `public/images/` 内に古いバックアップ画像が存在し、URL推測で外部からアクセス可能。<br>- 既存レポートチェックリスト備考に「最優先タスク」と記載されており、本作業時点でも未削除。<br>- 削除に加え、`public/` 配下のバックアップ命名禁止のガイドライン化が必要。<br>- 関連: A-1-03、A-1-05、A-1-07、D-4-01、J-1-01。 |
| A-1-07 | Low | 2.A.4 / 3.0 | `frontend/public/uploads/blog/` | `1770992461005.png`, `1770994779680.png` | これらはテストアップロードか。個人情報・機密情報が含まれていないか | (a) **`1770992461005.png`** (3,205,675 バイト、PNG 1024×1536、2026-02-13 mtime)<br>(b) **`1770994779680.png`** (540,013 バイト、PNG 368×552、2026-02-13 mtime)<br>**git ls-files で両ファイルとも Git 追跡対象**。ファイル名は Unix タイムスタンプ(ms)で 1770992461005 → 2026-02-13 23:01 JST(タイムスタンプから推測される命名規則)。<br>**注意**: 既存レポート §7.3 で「ファイル名がタイムスタンプベース → UUID へ変更済み」とされているが、これら2ファイルは UUID 移行 **以前** にアップロードされた残存ファイル。<br>**画像内容の検査(EXIF/個人情報)**: `file` コマンドの出力では PNG メタデータを含むかは判別できない(本作業範囲外、動的検証で `exiftool` 等で確認推奨)。Claude Code は画像内容を視覚的に確認可能だが、本作業ではプロジェクトの個人情報判定権限が無いため判定不能。 | 判定不能 | **判定根拠**: 機密情報含有の有無は画像内容と運用情報(誰の何の写真か)に依存:<br>- ファイルが Git に追跡されていることは確認(構造的問題)。<br>- 個人情報含有の有無は仕様決定者・運用者へのヒアリングが必要(A-1-07 備考の「テストアップロードか」は仕様確認マター)。<br>- EXIF 確認は `exiftool` 実行(動的検証)が必要。<br>- 関連: A-1-03、B-7-01 / B-7-02 (現実装の画像処理)、§7.3 既存レポート。 |
| A-1-08 | Medium | 2.A.3 | `frontend/` | `dev.db` | `dev.db` の Git 履歴、テーブル、レコード件数、機密性のあるデータの有無 | **重大事項発見**: `frontend/dev.db` は **Git 追跡対象**であり、現在も追跡されている:<br>- `git ls-files frontend/dev.db` → `frontend/dev.db` がリスト化される<br>- `git log --all --full-history -- frontend/dev.db` → **4 コミットの履歴**<br>  - `7b515d3` (2026-04-26 22:22) 「dev.db更新、scripts追加、技術スタックレポートver2追加」<br>  - `6bb6851` (パスワード管理UI改善とセキュリティ強化)<br>  - `dbaafa4` (Add blog categories, multi-user auth, security hardening, and UI improvements)<br>  - `c5526e1` (Implement staff management features including contacts and news)<br>- ファイルサイズ: 102,400 バイト(2026-04-18 mtime)<br>- WAL/journal/shm: **存在しない**(`dev.db-journal`, `dev.db-wal`, `dev.db-shm` 未生成、コンテナ停止状態)<br><br>**現状の dev.db 内テーブル/レコード件数**(node + better-sqlite3 で確認):<br>- `contacts`: 3 件(本物のお問い合わせデータが入っている可能性、name/email/message)<br>- `news`: 5 件<br>- `categories`: 6 件<br>- `blog_categories`: 5 件<br>- `blogs`: 7 件<br>- `users`: **2 件**(`***MASKED***` / `***MASKED***`、passwordHash, totpSecret, recoveryCodesHash, oneTimePassword 等の機密フィールドを含む可能性)<br>- `audit_logs`: **70 件**(IPアドレス/ユーザーエージェント/操作種別を含む)<br><br>**スキーマ上の機密フィールド**(prisma/schema.prisma):<br>- User: `passwordHash`, `totpSecret`, `recoveryCodesHash`, `oneTimePassword`<br>- AuditLog: `ipAddress`, `userAgent` | NG | **判定根拠**: ⚠️ **最重大級**:<br>- 開発DBが Git に追跡されており、過去4コミットの履歴に **bcrypt パスワードハッシュ・TOTP シークレット・リカバリコードハッシュ** がスナップショットとして残存。<br>- 平文ではないが、bcrypt rounds=12 でも長期間放置すれば総当たり可能性ゼロではない(特に弱パスワード)。<br>- TOTP secret は base32 文字列で、漏洩すれば 2FA を攻撃者が任意の時刻でバイパス可能(平文同等のリスク)。<br>- IPアドレス/UA を含む監査ログ70件 → 個人特定可能情報の混入リスク。<br>- 既存レポート §14 では「Prisma 使用 + AuditLog 実装済み」と OK 評価だが、**「Git 追跡されている」という構造的問題は別観点**。<br>- 修正方針: ① `git rm --cached frontend/dev.db` で追跡解除、② `.gitignore` に `dev.db*` 追加(A-1-03 修正)、③ 履歴クリーンアップ(`git filter-repo` 等)、④ 漏洩前提のシークレットローテーション(ENCRYPTION_KEY/SESSION_SECRET/全ユーザーパスワード/TOTP シークレット)。<br>- 関連: A-1-01 (`.env.local` は OK)、A-1-03、D-4-01、E-1-01 (Prisma スキーマ詳細)、B-14-01 (oneTimePassword 保存形式)。 |

---

## 章サマリ

- **回答済項目数:** 8件
  - Critical: 3件
  - High: 1件
  - Medium: 3件
  - Low: 1件
- **OK 件数:** 1件 (A-1-01)
- **NG 件数:** 5件 (A-1-03, A-1-04, A-1-05, A-1-06, A-1-08)
- **部分NG 件数:** 1件 (A-1-02)
- **判定不能 件数:** 1件 (A-1-07)
- **判定不能の主な理由:**
  - A-1-07: 画像内の個人情報含有有無は運用情報(撮影内容・テスト目的か実データか)と EXIF 解析に依存し、本作業範囲では確定不能

## 参照したファイル一覧

- `frontend/.gitignore` (40行)
- `.gitignore` (ルート、約40行)
- `frontend/.env.local`(値本体は読まず、長さのみ awk で集計)
- `frontend/dev.db` (better-sqlite3 経由で件数のみ取得)
- `frontend/prisma/schema.prisma` (113行、User/AuditLog 機密フィールド確認)
- `frontend/src/app/favicon.ico.backup`、`globals.css.backup`、`page.tsx.backup`(存在のみ確認)
- `frontend/src/components/layouts/Footer.tsx.20260212`、`Header.tsx.20260206`(存在のみ確認)
- `frontend/public/images/background.png.20261302`(存在・file コマンドのみ)
- `frontend/public/uploads/blog/1770992461005.png`、`1770994779680.png`(存在・file コマンドのみ)

## 実行した grep / コマンド一覧

- `git check-ignore -v frontend/.env.local frontend/dev.db frontend/dev.db-journal`
- `git log --all --full-history --oneline -- frontend/.env.local` (0件)
- `git log --all --full-history --oneline -- frontend/dev.db` (4件)
- `git ls-files frontend/dev.db frontend/dev.db-journal frontend/dev.db-wal frontend/dev.db-shm`
- `git ls-files | grep -E "(public/uploads|public/images/background\.png\.20261302|\.backup$|\.20260)"` (8件全て tracked)
- `find frontend/src -type f \( -name "*.backup" -o -name "*.2026*" \)`
- `awk -F= '{print "key="$1" len="length($2)}' .env.local` (実値を読まず長さのみ)
- `node -e "..." with better-sqlite3` (テーブル/件数列挙)
- `file <PNG画像>` (画像種別確認)

## 章をまたぐ関連項目

- A-1-01 ⇔ D-1-01 (env example の乖離)
- A-1-02 ⇔ M-1-04 (パスワードポリシーの仕様決定)、F-2-01 (SESSION_SECRET 利用先)、F-3-01 (ENCRYPTION_KEY 利用先)
- A-1-03 ⇔ A-1-04 / A-1-05 / A-1-06 / A-1-07 / A-1-08(全て gitignore 不備に起因)、D-4-01(.dockerignore も同様)
- A-1-08 ⇔ B-14-01 (oneTimePassword 保存形式)、E-1-01 (schema.prisma 機密フィールド)、F-7-01 (audit-log の IP)、6章認証ライブラリ群
