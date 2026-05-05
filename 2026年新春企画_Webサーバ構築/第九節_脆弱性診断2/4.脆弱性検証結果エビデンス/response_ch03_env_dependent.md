# 回答ファイル: 第3章 本番環境依存論点(2.C / Environment-dependent)

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第3章 本番環境依存論点
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

> **指示書 2.2 / 4.1 / 第3章方針に従い、本章は全項目「判定不能(本番環境設定に依存)」として記録する。Claude Code はリポジトリ内のドキュメント・設定ファイルの有無のみ確認可能で、実環境の構成は判定不能。**

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| C-1-01 | High | 2.C.2 | `frontend/docker/`, `docker/` | `Dockerfile.prod`, `docker-compose.prod.yml`, `.dockerignore` | 本番環境構成のリバースプロキシ・TLS終端・WAF・CDN の配置前提のドキュメント化 | (本番環境設定依存のため回答不能 — 運用担当ヒアリング必要)<br><br>**確認できる事実(リポジトリ内)**:<br>- `Dockerfile.prod`: 非rootユーザー(uid 1001 nextjs)+ multi-stage(D-3-02)。リバースプロキシ前提の `HOSTNAME="0.0.0.0"` / `EXPOSE 3000` 設定<br>- `docker-compose.prod.yml`: minimal(D-3-01 で NG)、reverse proxy / TLS / WAF / CDN の設定なし<br>- `.dockerignore`: `.env*` 単体や `*.db` の除外不備(D-4-01 で NG)<br>- `docs/architecture/` ディレクトリ: **空**(L-5-01 で確認、運用ドキュメント未記述) | 判定不能 | **判定根拠**:<br>- 本番デプロイ環境(Nginx? Traefik? Cloudflare? 等)の選定は運用判断<br>- リポジトリ内に本番アーキテクチャドキュメントが**全くない**(L-5-01)→ 運用担当ヒアリング必須<br>- 関連: D-3-01 / D-3-02 / D-4-01 / L-5-01。 |
| C-2-01 | High | 2.C.3 | (リポジトリ外) | - | 本番環境のリバースプロキシ、TLS終端、CDN/WAF採用、HSTS preload登録予定 | (本番環境設定依存のため回答不能 — 運用担当ヒアリング必要)<br><br>**確認できる事実(リポジトリ内)**:<br>- `next.config.ts:86`: HSTS ヘッダ `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` 設定済み(本番 HTTPS 化後に有効化される設計)<br>- HSTS preload 運用注意のコメント [next.config.ts:78-86] あり(削除困難・サブドメイン HTTPS 化の事前確認等を文書化) | 判定不能 | **判定根拠**:<br>- TLS 設定・CDN/WAF 採用は本番環境の運用判断<br>- HSTS preload リスト登録は手動申請(`https://hstspreload.org/`)が必要<br>- 関連: D-2-01 / 既存§13 / L-5-01。 |
| C-3-01 | High | 2.C.4 | (リポジトリ外) | - | DDoS/DoS多層防御の現状。アプリ層以外の防御策 | (本番環境設定依存のため回答不能 — 運用担当ヒアリング必要)<br><br>**確認できる事実(リポジトリ内のアプリ層防御)**:<br>- アカウントロック(指数バックオフ、login/actions.ts:36-42)<br>- レート制限(IP+ユーザー名+2FA、F-5-01 / G-2-01)<br>- contact form のメールアドレス別レート制限(G-6-01)<br>**ネットワーク層・インフラ層の DDoS 対策(Cloudflare DDoS Protection / AWS Shield 等)は本番環境依存** | 判定不能 | **判定根拠**:<br>- アプリ層防御は実装済(F-5-01 / G-2-01 で確認)<br>- ネットワーク・トランスポート層の防御は本番環境の運用判断<br>- 関連: F-5-01 / G-2-01 / G-6-01 / 既存§8.1。 |
| C-4-01 | High | 2.C.6 | (リポジトリ外) | - | Fail2ban 等のホストレベル侵入検知の導入予定 | (本番環境設定依存のため回答不能 — 運用担当ヒアリング必要)<br><br>**リポジトリ内に Fail2ban / OSSEC / Wazuh 等の設定ファイルなし**(grep で `fail2ban`/`auditd` 等のヒット 0件) | 判定不能 | **判定根拠**:<br>- ホストレベル侵入検知はホスト OS の運用範疇<br>- 関連: L-5-01 (運用ドキュメント未記述)。 |
| C-5-01 | High | 2.C.7 | (リポジトリ外) | - | ホストOSハードニング方針(UFW/iptables, SSH設定, unattended-upgrades, AppArmor, auditd) | (本番環境設定依存のため回答不能 — 運用担当ヒアリング + CIS Benchmark 照合必要)<br><br>**リポジトリ内**: ホスト OS 設定ファイルなし(Dockerfile.prod は alpine ベースのコンテナ内のみ) | 判定不能 | **判定根拠**:<br>- ホスト OS のセキュリティ設定は運用範疇<br>- 関連: L-5-01。 |
| C-6-01 | Medium | 2.C.8 | `docker/`, `frontend/` | `docker-compose.prod.yml`, `prisma.config.ts`, `.env.local.example` | 本番構成でのDBファイル配置、パーミッション、バックアップ設計、PostgreSQL移行計画 | (本番環境設定依存のため回答不能 — 運用担当ヒアリング必要)<br><br>**確認できる事実**:<br>- `docker-compose.prod.yml`: ボリューム未定義(D-3-01)、DB ファイル永続化なし<br>- `frontend/src/lib/prisma.ts:9`: DB url `'file:./dev.db'` ハードコード(F-9-01 で NG 寄り)<br>- バックアップ設計: 既存§18 で「未実装」と指摘、E-3-01 で同様<br>- PostgreSQL 移行: M-1-09 仕様確定待ち | 判定不能 | **判定根拠**:<br>- 本番 DB 設計は仕様(SQLite 継続 vs PostgreSQL 移行)+運用(配置/パーミッション/バックアップ)判断<br>- 関連: D-3-01 / F-9-01 / E-3-01 / M-1-09 / 既存§18。 |

---

## 章サマリ

- **回答済項目数:** 6件
  - Critical: 0件
  - High: 5件 (C-1-01, C-2-01, C-3-01, C-4-01, C-5-01)
  - Medium: 1件 (C-6-01)
  - Low: 0件
- **OK 件数:** 0件
- **NG 件数:** 0件
- **部分NG 件数:** 0件
- **判定不能 件数:** 6件(全件)
- **判定不能の主な理由:**
  - 全 6 項目が **本番環境(リバースプロキシ / TLS / WAF / CDN / ホストOS / DB配置)** に依存する Env-dep カテゴリ。
  - リポジトリ単体では運用構成が判定できない(指示書 4.1 / 第3章方針通り)。
  - 各項目について、リポジトリ内で確認できる関連事実(設定ファイルの有無、関連実装の状況)は備考に明記。
  - 運用担当ヒアリング + CIS Benchmark 照合 + 動的検証(Nmap / OpenVAS 等)が後工程で必要。

## 参照したファイル一覧

(影響先の事実確認に使用、他章で詳細記載済み)

- (再利用) `frontend/docker/Dockerfile.prod` (D-3-02)
- (再利用) `docker/docker-compose.prod.yml` (D-3-01)
- (再利用) `frontend/.dockerignore` (D-4-01)
- (再利用) `frontend/next.config.ts` の HSTS 設定 (D-2-01)
- (再利用) `docs/architecture/` (L-5-01、空ディレクトリ確認)

## 実行した grep / コマンド一覧

(本章独自のコマンドはなし。第4章 / 第6章 / 第12章 で実施済の結果を参照)

## 章をまたぐ関連項目

- C-1-01 / C-2-01 / C-3-01 / C-4-01 / C-5-01 ⇔ L-5-01 (運用ドキュメント未記述)
- C-1-01 ⇔ D-3-01 / D-3-02 / D-4-01
- C-2-01 ⇔ D-2-01 (HSTS) / 既存§13
- C-3-01 ⇔ F-5-01 / G-2-01 / G-6-01 (アプリ層防御)
- C-6-01 ⇔ M-1-09 / F-9-01 / D-3-01 / E-3-01 / 既存§18
