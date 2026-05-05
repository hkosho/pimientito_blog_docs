# 回答ファイル: 第9章 スクリプト・既存レポート確認

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第9章 スクリプト・既存レポート確認(3.6 / 3.7)
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| I-1-01 | High | 3.6 | `frontend/scripts/` | `reset-password.ts` | パスワードリセット処理の引数取扱い、実行権限、パスワード生成方法、ログ出力内容 | (a) **引数取扱い:** 引数なし。`usernames = ['***MASKED***', '***MASKED***']` がハードコードされた固定2ユーザーに対してのみ処理する(`process.argv` 未参照)。<br>(b) **実行権限:** スクリプト内に実行権限の制御は無く、`prisma.user.update` を直接呼ぶ。`PrismaBetterSqlite3({ url: 'file:./dev.db' })` で **dev.db に直接書き込む** ため、運用上は dev.db に到達できる環境(コンテナ内 / 開発機)でしか実害はないが、ファイルパーミッションのみが防御線。<br>(c) **パスワード生成:** `crypto.randomBytes(16)` で 16 バイト(=128 bit エントロピー)の暗号学的乱数を取得し、56文字字種(紛らわしい文字 `0/O/1/I/l/o` を除外)から `bytes[i] % chars.length` で 16 文字生成。理論エントロピーは約 16×log₂56 ≈ 92.9 bit。**ただしモジュロバイアスあり**: 256 / 56 = 4 余り 32 のため、最初の 32 文字(`ABCDEFGHJKLMNPQRSTUVWXYZabcdefgh`)が他より僅かに出やすい(影響は実用上極小)。<br>(d) **ログ出力:** 生成した平文一時パスワードを `console.log("temp password: ${tempPassword}")` で **標準出力に出力**。 | 部分NG | **判定根拠**:<br>- 引数なし固定対象で危険な汎用化を回避している点は OK。<br>- 平文パスワードを stdout に出力するのは管理用途として一般的で、Pipe先がログファイルでなければ実害は少ない。が、CI/コンテナ運用で stdout がログ収集対象になる場合は **平文パスワードがログ基盤に流入** するリスクあり。<br>- モジュロバイアスは Low レベルの指摘で、現実的な攻撃成立条件はほぼない。<br>- 引用元: [reset-password.ts:9-15](frontend/scripts/reset-password.ts#L9-L15) (生成ロジック)、[reset-password.ts:35-37](frontend/scripts/reset-password.ts#L35-L37) (出力)、[reset-password.ts:18-19](frontend/scripts/reset-password.ts#L18-L19) (固定対象)。<br>- bcrypt rounds は 12(`bcrypt.hash(tempPassword, 12)` [reset-password.ts:21](frontend/scripts/reset-password.ts#L21))で `BCRYPT_ROUNDS = 12` 統一方針(既存レポート §4.4)と一致。<br>- 関連: 既存レポート `security-report.md` §6.2 (初期パスワードのレスポンス送信)。 |
| I-2-01 | High | 3.6 | `frontend/scripts/` | `list-users.ts` | 出力内容に機密情報(ハッシュ/メール等)が含まれるか | `prisma.user.findMany` の `select` で以下フィールドのみ取得し JSON.stringify で stdout に出力:<br>- `id`, `username`, `email`, `role`, `isLocked`, `mustChangePassword`, `twoFactorEnabled`, `loginAttempts`<br>**`passwordHash` / `totpSecret` / `recoveryCodesHash` / `oneTimePassword` / `otpExpiresAt` / `lockedUntil` 等の機密フィールドは `select` に含まれず、出力されない**。 | 部分NG | **判定根拠**:<br>- パスワードハッシュ・TOTPシークレット・リカバリコードハッシュ・OTP は出力されず、機密フィールド漏洩リスクは限定的(OK寄り)。<br>- ただし `email`(個人情報)・`username`(認証要素の片方)は出力される。stdout がログ収集される運用や、出力結果がチャット/メール経由で共有された場合は **個人情報漏洩** となり得る。<br>- スクリプト自体に出力先制限・マスキング・出力先確認プロンプトは無い。<br>- 引用元: [list-users.ts:8-19](frontend/scripts/list-users.ts#L8-L19) (select 句)、[list-users.ts:20](frontend/scripts/list-users.ts#L20) (出力)。<br>- DTO の安全側 `safeUserSelect` (`src/lib/dto.ts`、既存レポート §6.5)とは別途 `select` を直書きしているため、将来の機密フィールド追加時に同期忘れリスクあり(コードレベル指摘)。 |
| I-3-01 | High | 3.7 / 4.C.33 | `security-report/` | `security-report.md`, `security-review-response.md`, `security-test-results.md` | 既存レポートの指摘事項と対応状況。今回の診断範囲との重複洗い出し | (a) **`security-report.md` (2026-02-14、第3回修正後)**:<br>初回31件検出 → 修正済み 26件、条件付き受容 3件、残存 5件(LOW 2件 + INFO 3件)。21章構成で SQLi / XSS / CSRF / 認証 / 認可 / 情報漏洩 / アップロード / DoS / トラバーサル / 機密情報 / クリックジャッキング / HTTP ヘッダ / TLS / DB / 依存パッケージ / Docker / ログ監査 / バックアップ / ポート / スパム / 2FA を網羅。<br>(b) **`security-review-response.md` (2026-02-14)**:<br>Gemini + ChatGPT による第2次レビューへの回答書。Priority 1 (Critical) 4件・Priority 2 (High) 〜 を「✅対応済み / ⚠️低減済み / 📋条件付き受容 / 🔲本番前必須」のラベルで整理。<br>(c) **`security-test-results.md` (2026-02-14)**:<br>動的テストの実行結果記録。ユーザー列挙 / レート制限 / CSRF / セッション固定 / CSP / アップロードヘッダ / DTO漏洩 / 監査ログ改ざん の 8 検証項目について `scripts/test-csrf.sh` `scripts/test-session-fixation.sh` の出力ログを含め記録。<br><br>(d) **本チェックリストとの重複範囲**:<br>- A-1-01〜08 (1章) ⊃ §10 (機密情報) §18 (バックアップ) §14 (DB)<br>- B-3-01/02 (middleware) ⊃ §5.1<br>- B-4-01 / F-2-01 (session) ⊃ §4.1, §4.2<br>- B-5-01 / F-3-01 (crypto) ⊃ §4 全般<br>- B-5-02 (TOTP再利用) ⊃ §4.6 (📋条件付き受容)<br>- B-6-01 / F-5-01 (rate-limit) ⊃ §4.3, §8.3<br>- B-7-01/02 / G-4-01 (画像アップロード) ⊃ §7 全般<br>- B-8-01 / G-2-01 (セッション固定) ⊃ §4.x + `security-test-results.md` §4<br>- B-9-01 (CSRF) ⊃ §3 + `security-test-results.md` §3<br>- B-11-01 (CSP) ⊃ §12<br>- B-12-01 / F-7-01 (audit-log) ⊃ §17<br>- B-14-01 (oneTimePassword) ⊃ §4.6<br>- D-3-02 / 16章 (Docker) ⊃ §16<br>- D-5-01 (依存) ⊃ §15<br>- E-2-01 / L-1-01 (bcrypt) ⊃ §4.4 (12 rounds 統一) | OK | **判定根拠**:<br>- 既存レポート3本ともリポジトリに存在(`ls -la security-report/` で確認、最終更新 2026-02-14)。<br>- 既存レポートで「修正済み」と記載されている項目について、本回答では各章で **現状コードを再確認** し、5.3 ルールに基づき初出箇所で詳細引用、以降は「既存レポート §X 参照」で簡略化する。<br>- 既存レポートの最終更新は 2026-02-14、現コミット `1e75c36`(2026-05-03 時点) との差分は本作業中に判明したものを各章備考に明記する方針。<br>- 関連: 5.3 章をまたいだ重複処理の回避、7.3 既存レポートとの重複。 |

---

## 章サマリ

- **回答済項目数:** 3件
  - Critical: 0件
  - High: 3件
  - Medium: 0件
  - Low: 0件
- **OK 件数:** 1件 (I-3-01)
- **NG 件数:** 0件
- **部分NG 件数:** 2件 (I-1-01, I-2-01)
- **判定不能 件数:** 0件
- **判定不能の主な理由:** 該当なし(本章はリポジトリ内のスクリプトおよびレポートを直接参照可能)

## 参照したファイル一覧

- `frontend/scripts/reset-password.ts` (全42行)
- `frontend/scripts/list-users.ts` (全23行)
- `security-report/security-report.md` (全923行、要約読込)
- `security-report/security-review-response.md` (冒頭80行、構造把握用)
- `security-report/security-test-results.md` (冒頭80行、構造把握用)

## 実行した grep / コマンド一覧

- `ls -la /srv/git/arte-web-site-project/frontend/scripts/`
- `ls -la /srv/git/arte-web-site-project/security-report/`
- `git rev-parse HEAD` → `1e75c36647736d45ad49f25a4448b51ffd9ec109`

## 章をまたぐ関連項目

- I-1-01 ⇔ E-2-01 / L-1-01 (bcrypt rounds=12 の統一性)
- I-2-01 ⇔ §6.5 既存レポート (DTO 適用)、F-9-01 (Prisma 使用パターン)
- I-3-01 は本作業全体への前提情報。各章の「既存レポート §X 参照」記述の根拠となる
