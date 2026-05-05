# 脆弱性診断 机上デバッグ観点 計画書 ver 4.0（Claude Web版）

**対象リポジトリ:** `arte-web-site-project`（Next.js 16 + TypeScript + Prisma 7 + SQLite）
**作成日:** 2026-04-22
**版:** ver 4.0（ver 3.0 に対し、ChatGPT / Claude / Gemini による3者レビュー結果を反映）
**作成者:** Claude（Web版、Opus 4.7）
**位置付け:** ステップ5-2の修正アウトプット。ver 3.0 の草案を、マージされたレビュー結果（`merge_draft_review_result_by_claude.md`）に基づき修正した計画書。

---

## 版の変遷と変更概要

### ver 3.0 → ver 4.0 での主な変更点

**構造面:**
- 0章に **証拠区分**（Confirmed / Suspected / Environment-dependent）と **回答主体**（Code / Repo docs / Infra・運用担当 / 仕様決定者）の2タグを導入
- 0.4 の「6領域の位置付け」を **「7領域の位置付け」** に修正（1.G追加で実質7領域だった整合を是正）
- **1章冒頭に「画面群 × 脅威マトリクス」** を追加（ChatGPT指摘 1-3）
- **2章（優先度型）を3分割**：2.A 確認済み重大 / 2.B repoから強く疑われる / 2.C 環境依存の重要確認
- 3章（コード型）を **repo内限定** に整理、従来の 3.11（運用・インフラ設定）は4章へ移動
- 4章を **4分類**（4.A 仕様確定 / 4.B 本番環境構成 / 4.C 証跡取得 / 4.D 保留でもよい）に再編し、各項目に回答主体タグを付与
- 5章末尾に **「次版反映方針」** を追加（何を削る／4章へ移す／2章へ昇格するかを明示）

**内容面:**
- **IPA-2.4（フィッシング対策）** 観点を 1.A.13 として追加
- 別冊「安全なSQLの呼び出し方」の **静的/動的プレースホルダ区別・文字エンコーディング起因SQLi・数値リテラル** を 1.A.1 に追加
- **Node.js / TypeScript 固有観点** を 1.F に追加：Prototype Pollution、ReDoS、Mass Assignment、ランタイム型検証、HPP（HTTPパラメータ汚染）
- **React 19 / Next.js 16 / Prisma 7 固有観点** を 1.F に追加：taint API、Server Actions `bind()` 誤用、PPR、Edge Runtime 制約、`fetch` cache挙動、Route Handler 存在確認
- **WebサーバOS・コンテナ層** を 1.B/1.C に追加：Alpine musl libc、init プロセス、FS書き込み領域、apk署名検証、Docker Content Trust
- **ネットワーク層** を 1.D に追加：HTTP/2 Rapid Reset、DNS リバインディング、Subdomain Takeover、Host ヘッダインジェクション、Slowloris、TLS詳細項目
- **DB層** を 1.E に追加：Prisma `where` 型安全性、SQLite WAL/journal、`X-Forwarded-For` 検証、Down migration、クエリタイムアウト
- **サプライチェーン** を 1.F/新規観点 に追加：postinstall 検査、SBOM、Lockfile Poisoning、SRI
- **悪意ある第三者観点** を新章 1.H として追加：Credential Stuffing、ATO連鎖、Insider Threat、Enumeration、CSP Bypass
- **画面キャプチャからの具体リスク** を 2章および該当観点に反映：CL-24-②/CL-25-② ショルダーハッキング、CL-10 一括削除、CL-17a カテゴリチップ整合性、CL-01 外部リンク `rel`、CL-29 2FA無効化、監査ログ削除/エクスポート
- **情報露出・エラー管理** を 1.A.14 として独立項目化
- `dangerouslySetInnerHTML` 等の **フレームワーク迂回経路** を 1.A.5 および 3.10 で強調

**実効性面:**
- 各観点を可能な範囲で **「確認対象 / 危険と判断する条件 / 除外条件 / 確認証跡」** の4要素で整理
- 観点ごとに **タグ**：`[コード確認]` `[ドキュメント確認]` `[運用確認]`
- 観点ごとに **起点脅威** を付与（3章の各ファイルに、主要な脅威を1つ付記）
- 重複許容ルールの限定：同じ論点の重複は1章と2章の二箇所まで、3章と4章では再掲しない

---

## 目次

- [0. 前提情報と略記](#0-前提情報と略記)
  - [0.1 優先度定義](#01-優先度定義)
  - [0.2 証拠区分（新設）](#02-証拠区分新設)
  - [0.3 回答主体タグ（新設）](#03-回答主体タグ新設)
  - [0.4 略記](#04-略記)
  - [0.5 本リポジトリの技術的前提（Confirmed のみ）](#05-本リポジトリの技術的前提confirmed-のみ)
  - [0.6 7領域の位置付け](#06-7領域の位置付け)
  - [0.7 「机上デバッグ」の範囲定義（新設）](#07-机上デバッグの範囲定義新設)
- [1. 網羅型観点リスト](#1-網羅型観点リスト)
  - [1.0 画面群 × 脅威マトリクス（新設）](#10-画面群--脅威マトリクス新設)
  - [1.A IPA 11脆弱性 × リポジトリ構成 × ウェブ健康診断13項目](#1a-ipa-11脆弱性--リポジトリ構成--ウェブ健康診断13項目)
  - [1.B インフラセキュリティ（コンテナ／オーケストレーション層）](#1b-インフラセキュリティコンテナオーケストレーション層)
  - [1.C サーバセキュリティ（コンテナ内部OS／ランタイム層）](#1c-サーバセキュリティコンテナ内部osランタイム層)
  - [1.D ネットワークセキュリティ](#1d-ネットワークセキュリティ)
  - [1.E データベースセキュリティ](#1e-データベースセキュリティ)
  - [1.F フレームワーク／言語ランタイムセキュリティ](#1f-フレームワーク言語ランタイムセキュリティ)
  - [1.G 運用・インフラ防御層（条件付き観点）](#1g-運用インフラ防御層条件付き観点)
  - [1.H 悪意ある第三者からの攻撃観点（新設）](#1h-悪意ある第三者からの攻撃観点新設)
- [2. 優先度型観点リスト（本リポジトリ固有リスク）](#2-優先度型観点リスト本リポジトリ固有リスク)
  - [2.A 確認済みの重大論点（Confirmed）](#2a-確認済みの重大論点confirmed)
  - [2.B repoから強く疑われる論点（Suspected）](#2b-repoから強く疑われる論点suspected)
  - [2.C 本番環境依存の重要確認（Environment-dependent）](#2c-本番環境依存の重要確認environment-dependent)
- [3. コード型：確認必須ファイルリスト（repo内限定）](#3-コード型確認必須ファイルリストrepo内限定)
- [4. 未確認事項・ヒアリング候補（4分類・回答主体付き）](#4-未確認事項ヒアリング候補4分類回答主体付き)
- [5. 本草案の限界と次版反映方針](#5-本草案の限界と次版反映方針)
- [版履歴まとめ](#版履歴まとめ)

---

## 0. 前提情報と略記

### 0.1 優先度定義

| 優先度 | 定義 |
|--------|------|
| **Critical** | 被害発生時の影響が甚大で、かつ **証拠区分 Confirmed**（既に問題の兆候が確認できる）であるもの。即時対応が必要。 |
| **High** | 能動的攻撃で直接被害につながる脆弱性（IPA危険度「高」相当）。実装確認を最優先で行うべきもの。Suspected であっても実装次第で Critical に昇格する可能性がある。 |
| **Medium** | 受動的攻撃、または限定的影響の脆弱性（IPA危険度「中」相当）。実装確認が必要だが、Critical/High の後でもよいもの。 |
| **Low** | 攻撃成功確率が低い、または影響が軽微なもの（IPA危険度「低」相当）。運用上の注意レベル。 |

**補足:** 優先度は「影響度」を示す。修正コストとの2軸評価は、机上デバッグ結果を踏まえ次ステップで行う（レビュー指摘 13-2 対応）。

### 0.2 証拠区分（新設）

レビュー指摘（ChatGPT 0-2, 2-2）を踏まえ、各観点・論点に以下の証拠区分タグを付与する。優先度と併記することで、「重大ではあるが未確認」と「重大かつ既に兆候あり」を区別する。

| タグ | 意味 |
|------|------|
| **`Confirmed`** | 技術スタック調査レポート、画面分析資料、リポジトリツリー情報等から、**既に問題の兆候または設計上の制約が明示されている** もの。 |
| **`Suspected`** | 上記資料から **強く疑われる** が、実装を読まないと確定しないもの。 |
| **`Environment-dependent`** | 本番環境の設定・運用手順に依存するもの。repo 単体では確認不能。 |

### 0.3 回答主体タグ（新設）

レビュー指摘（ChatGPT 4-2, Claude 12-3）を踏まえ、各ヒアリング項目および未確認事項に、以下の回答主体タグを付与する。ステップ6以降の skill.md 作成時に、Claude Code で回答可能な項目と人間へのヒアリングが必要な項目を分離するために用いる。

| タグ | 意味 |
|------|------|
| **`[Code]`** | リポジトリ内のソースコードを Claude Code が読めば確定できる項目。 |
| **`[Repo docs]`** | リポジトリ内の `docs/` 配下ドキュメントで確定できる項目。 |
| **`[Infra/運用]`** | 本番環境構成・運用手順の担当者へのヒアリングが必要な項目。 |
| **`[仕様決定]`** | 仕様決定者・プロダクトオーナーの判断が必要な項目。 |

### 0.4 略記

- **IPA-N.N**: 「安全なウェブサイトの作り方 改訂第7版」の章番号（例：IPA-1.1 = SQLインジェクション、IPA-2.4 = フィッシング詐欺を助長しないための対策）
- **WKD-X**: 「ウェブ健康診断仕様」の診断項目記号（例：WKD-A = SQLインジェクション）
- **CL-NN**: 画面分析資料の画面ID（例：CL-05 = Contact画面）
- **対策ID**: IPAチェックリスト上のID（例：1-(i)-a = SQLインジェクションのプレースホルダ実装）
- **SQL別冊-N.N**: IPA別冊「安全なSQLの呼び出し方」の章番号

### 0.5 本リポジトリの技術的前提（Confirmed のみ）

レビュー指摘（ChatGPT 0-2）を踏まえ、本節には **Confirmed のみ** を記載する。未確認事項は4章へ移動。

- フロントエンド＋バックエンドが Next.js（16.1.6）の Server Actions / Route Handler で一体化 `Confirmed`
- 専用のバックエンドAPIサーバは存在しない（`backend/` はREADMEのみ） `Confirmed`
- Prisma 7.4 + SQLite（better-sqlite3 アダプタ） `Confirmed`
- 認証：iron-session（8系、Cookie名 `staff_session`、maxAge 8時間、httpOnly/secure/sameSite=lax）+ bcryptjs + otplib（TOTP 2FA） `Confirmed`
- レート制限：インメモリ（Mapベース、単一プロセス前提） `Confirmed`
- セキュリティヘッダ：X-Frame-Options（SAMEORIGIN）、X-Content-Type-Options（nosniff）、Referrer-Policy、HSTS（max-age=63072000; includeSubDomains; preload）、Permissions-Policy、CSP（公開/staff分離）、`poweredByHeader: false` `Confirmed`
- `frontend/.env.local` がリポジトリ内に存在（`ENCRYPTION_KEY` / `SESSION_SECRET` / `SEED_ADMIN_PASSWORD` が平文格納）`Confirmed`
- `frontend/dev.db` がリポジトリ内に存在 `Confirmed`
- バックアップ系ファイルが残存（`*.backup`, `Footer.tsx.20260212`, `background.png.20261302` 等）`Confirmed`
- Docker 本番イメージは `node:20-alpine` ベース、実行ユーザー `nextjs`（UID 1001、非root）、Next.js standalone モード `Confirmed`
- リバースプロキシ設定はリポジトリ内に存在しない `Confirmed`（有無自体は `Environment-dependent`）
- `scripts/test-csrf.sh`、`scripts/test-session-fixation.sh` が存在 `Confirmed`

**未確認事項は 4章 に移動。**

### 0.6 7領域の位置付け

レビュー指摘（ChatGPT 0-1）を踏まえ、「6領域」を **「7領域」** に修正し、各章の役割を一文で固定する。

```
┌──────────────────────────────────────────────────────────┐
│ 1.F フレームワーク／言語ランタイム                       │
│     (Next.js/React/Prisma/iron-session/Node.js/TS)       │
├──────────────────────────────────────────────────────────┤
│ 1.A アプリケーション（IPA 11脆弱性 × WKD 13項目）        │
├──────────────────────────────────────────────────────────┤
│ 1.E データベース（SQLite/PostgreSQL、データ暗号化等）    │
├──────────────────────────────────────────────────────────┤
│ 1.C コンテナ内部サーバ（Alpine・Node.jsランタイム）      │
├──────────────────────────────────────────────────────────┤
│ 1.B インフラ（Docker・コンテナ・secrets管理）            │
├──────────────────────────────────────────────────────────┤
│ 1.G ホストOS・運用防御（条件付き観点）                   │
├──────────────────────────────────────────────────────────┤
│ 1.D ネットワーク（TLS・DNS・リバースプロキシ・WAF）      │
└──────────────────────────────────────────────────────────┘

    横断章：
    1.H 悪意ある第三者からの攻撃観点（ver 4.0 新設）
```

**各章の役割（固定）:**
- **1.A**: IPA起点のアプリ脆弱性
- **1.B〜1.F**: 技術レイヤ補完（repo内根拠を持つ項目）
- **1.G**: repo外根拠が必要な項目（本番ホストOS・運用防御。本番運用がLinux系ホストOSである場合の条件付き観点）
- **1.H**: 攻撃者視点で横断的に整理（公開面・認証面・権限管理面・情報漏洩面）

**1.C（コンテナ内）と 1.G（ホストOS）の違い（レビュー指摘 B〜G-3 対応）:**
- **1.C**: Dockerコンテナ内部のAlpine Linux、Node.jsランタイム、コンテナ内ファイル権限 **のみに限定**
- **1.G**: Dockerを動かしているホストOSそのもののハードニング、侵入検知、自動ブロック、運用監視

### 0.7 「机上デバッグ」の範囲定義（新設）

レビュー指摘（Claude 12-4）を踏まえ、本草案が対象とする「机上デバッグ」の範囲を明示する。

| 範囲 | 内容 | 対応章 |
|------|------|--------|
| **机上デバッグの主対象（repo内）** | コードを読むだけで判断できる観点。リポジトリ内のファイル・設定・コメントから確定可能。 | 1.A / 1.B / 1.C / 1.D（repo内部分）/ 1.E / 1.F / 1.H、2.A、2.B、3章 |
| **机上デバッグの補助対象（仕様確認）** | 仕様の明確化が必要で、仕様決定者への確認が必要な観点。 | 4.A |
| **机上デバッグの範囲外（環境依存）** | 本番ホストOS・運用設定・外部サービス設定に依存する観点。本草案で観点として提示するが、ステップ8のClaude Code調査では対象外。運用担当者へのヒアリングで対応。 | 1.G、2.C、4.B、4.C |
| **今回は保留でもよい項目** | 将来的に重要だが現時点の机上デバッグでは優先度が低い項目。 | 4.D |

---

## 1. 網羅型観点リスト

**本章は辞書であり、実施順は2章に従う** （レビュー指摘 1-1 対応）。各観点は可能な範囲で **「確認対象 / 危険と判断する条件 / 除外条件 / 確認証跡」** の4要素で整理する。

### 1.0 画面群 × 脅威マトリクス（新設）

レビュー指摘（ChatGPT 1-3, A-2）を踏まえ、画面キャプチャの画面群ごとに、主要脅威・主確認ファイル・対応WKD項目を一望できるマトリクスを冒頭に置く。

| 画面群 | 主要画面 | 主要操作 | 主脅威 | 対応WKD | 主確認ファイル |
|---|---|---|---|---|---|
| **公開サイト** | CL-01〜04 | 閲覧 | XSS（ブログ本文）、オープンリダイレクト、クローラ耐性、外部リンク `rel` | WKD-B, M | `src/app/page.tsx`, `src/app/blog/**`, `src/app/news/**`, `src/app/about/**` |
| **Contact（公開入力）** | CL-05 | 送信 | XSS、CSRF、SQLi、メールヘッダインジェクション、ReDoS、スパム、レート制限 | WKD-A, B, C, F | `src/app/contact/**` |
| **Staff Login / 2FA** | CL-06, CL-07 | 認証 | セッション管理（固定化、再生成）、認証失敗制御、ユーザ列挙、2FAバイパス、Credential Stuffing | WKD-J, K | `src/app/staff/login/**`, `src/lib/auth.ts`, `src/lib/session.ts`, `src/middleware.ts` |
| **Staff Dashboard** | CL-08, CL-09 | 閲覧 | ロール別UI分岐漏れ、キャッシュによるクロスユーザ漏洩 | WKD-L | `src/app/staff/page.tsx`, `src/app/staff/StaffShell.tsx`, `src/app/staff/layout.tsx` |
| **Staff CRUD** | CL-10〜21 | 作成・更新・削除・一括削除 | 認可（水平/垂直）、CSRF、XSS（管理画面反射）、IDOR、ファイルアップロード、一括操作の上限 | WKD-C, L | `src/app/staff/blog/**`, `src/app/staff/news/**`, `src/app/staff/categories/**`, `src/app/staff/contacts/**` |
| **User管理（admin限定）** | CL-22〜26 | ユーザ作成/削除、パスワード発行、2FA設定、初期PW表示 | admin認可、Mass Assignment、自己削除防止、ショルダーハッキング（初期PW/OTP画面表示）、承認フロー | WKD-J, L | `src/app/staff/users/**`, `src/lib/auth.ts` |
| **Audit Logs（admin限定）** | CL-27, CL-28 | 検索・フィルタ | admin認可、ログ改ざん耐性、フィルタ入力XSS、IP信頼性（`X-Forwarded-For`） | WKD-L | `src/app/staff/audit-logs/**`, `src/lib/audit-log.ts` |
| **セキュリティ設定** | CL-29 | パスワード変更、2FA有効/無効化 | 重要操作再認証、2FA無効化の単一ボタン化、変更通知 | WKD-J | `src/app/staff/settings/**` |

---

### 1.A IPA 11脆弱性 × リポジトリ構成 × ウェブ健康診断13項目

IPA資料の11脆弱性を軸に、各脆弱性について「本リポジトリで該当しうる箇所」「確認方法」「参照資料」を網羅的に列挙する。ver 4.0 では IPA第2章由来の観点（IPA-Oper）および WKD 検出パターン粒度での落とし込みを強化（レビュー指摘 A-1, 3-4 対応）。

**凡例:**
- **[IPA-Appl]**: IPA第1章由来（アプリケーション層の実装）
- **[IPA-Oper]**: IPA第2章由来（運用・組織対応）

---

#### 1.A.1 SQLインジェクション（IPA-1.1 / WKD-A）

##### 観点A.1.1 Prismaクエリでの生SQL使用 [IPA-Appl] [コード確認]

- **確認対象**: `src/` 配下の `$queryRaw`, `$executeRaw`, `$queryRawUnsafe`, `$executeRawUnsafe` の使用箇所。
- **危険と判断する条件**:
  - `$queryRawUnsafe` / `$executeRawUnsafe` にユーザ入力を含む文字列が渡されている（動的プレースホルダ以下の安全性）
  - `$queryRaw` / `$executeRaw` でタグ付きテンプレートリテラル形式ではなく、文字列連結で組み立てられたSQL文が渡されている
- **除外条件**:
  - `$queryRaw` のタグ付きテンプレートリテラル形式（`` $queryRaw`SELECT ... WHERE id = ${id}` ``）は、パラメータ化される（SQL別冊-3.2.1 静的プレースホルダ相当）
- **確認証跡**: grep結果（該当ファイルパスと行番号）、および該当箇所が Confirmed / Suspected のいずれか。
- **該当コード候補**: `src/lib/prisma.ts`、`src/app/staff/**/actions.ts`、`src/lib/audit-log.ts`
- **参照**: IPA-1.1 根本的解決 1-(i)-a、SQL別冊 3.2.1（静的プレースホルダ）、3.2.2（動的プレースホルダ）
- **優先度**: High

##### 観点A.1.2 検索・フィルタ機能での動的WHERE構築 [IPA-Appl] [コード確認]

- **確認対象**: ブログ一覧のカテゴリフィルタ（CL-04）、監査ログのフィルタ（CL-27、CL-28）のクエリ構築箇所。
- **危険と判断する条件**:
  - `where: { id: { in: userInputArray } }` で配列の長さ制限がない（巨大配列による DoS の可能性）
  - `contains` 演算子に `%` `_` のエスケープがない（SQLiteでは `LIKE` のメタ文字扱いだが、PostgreSQL移行時に `mode: 'insensitive'` と組み合わさって挙動変動）
  - `OR` 条件をユーザ入力の数に応じて動的に増やしている（クエリプラン変動、メモリ消費）
- **除外条件**: Prisma の型安全な演算子のみで構築され、ユーザ入力は Zod 等でバリデーション済み
- **確認証跡**: 各 `actions.ts` の where 構築箇所の抜粋
- **該当コード候補**: `src/app/staff/audit-logs/page.tsx`、`src/app/staff/audit-logs/AuditLogsTable.tsx`、`src/app/blog/page.tsx`、`src/app/blog/MobileFilterMenu.tsx`
- **参照**: IPA-1.1、WKD-A 検出パターン1〜3、SQL別冊 2.5（SQLインジェクションの原因）
- **優先度**: Medium

##### 観点A.1.3 ORDER BY / LIMIT / OFFSET のパラメータ化 [IPA-Appl] [コード確認]

- **確認対象**: Prismaの `orderBy`、`take`、`skip` にユーザ入力を渡している箇所。
- **危険と判断する条件**: 列名を文字列で動的に指定しており、ホワイトリスト検証がない
- **該当コード候補**: 一覧表示系の `*Table.tsx`、`actions.ts`
- **優先度**: Low

##### 観点A.1.4 エラーメッセージの外部露出 [IPA-Appl] [コード確認]

- **確認対象**: `try/catch` 内での `error.message` の使い方、Next.js の `error.tsx` / `not-found.tsx`。
- **危険と判断する条件**:
  - Prisma のエラー（例：`PrismaClientKnownRequestError`）をそのままレスポンスに含めている
  - SQLite のエラーメッセージ（例：`SQLITE_CONSTRAINT`）がそのまま露出している
- **除外条件**: 本番ビルド時に Next.js がスタックトレースを自動マスクしている（要確認）
- **該当コード候補**: 各 `actions.ts`、`src/app/**/error.tsx`（存在する場合）
- **参照**: IPA-1.1 保険的対策 1-(iii)
- **優先度**: Medium

##### 観点A.1.5 DBアカウント権限（PostgreSQL移行時） [IPA-Appl] [仕様決定]

- **確認対象**: `prisma.config.ts`、`.env.local.example` の接続文字列。本番運用時のDB権限設計方針。
- **危険と判断する条件**: PostgreSQL移行後、アプリケーション用アカウントに DDL 権限（`CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`）が付与される設計になっている
- **参照**: IPA-1.1 保険的対策 1-(iv)
- **優先度**: Low（現時点・SQLite運用前提）

##### 観点A.1.6 文字エンコーディング起因のSQLi（将来観点） [IPA-Appl] [仕様決定]

レビュー指摘（Claude 3-2）対応で追加。

- **確認対象**: PostgreSQL移行時の `client_encoding` 設定（SQL別冊 A.2, A.3 参照）
- **危険と判断する条件**: Shift_JIS 等のマルチバイトエンコーディングが使用される場合、バックスラッシュのエスケープに起因する SQL インジェクションが発生しうる
- **除外条件**: SQLite は UTF-8 既定のため現時点は低リスク。PostgreSQL移行時に `client_encoding=UTF8` が明示されれば問題なし
- **優先度**: Low（現時点）

##### 観点A.1.7 数値リテラルに対するインジェクション [IPA-Appl] [コード確認]

レビュー指摘（Claude 3-2）対応で追加。

- **確認対象**: TypeScript のランタイムで `string` として渡ってきた「数値」を、`parseInt` / Zod等でのバリデーションなしに Prisma のフィルタに直接使っている経路
- **危険と判断する条件**: `where: { id: userInput }` の形で、`userInput` が文字列のまま Prisma に渡されている箇所（Prisma が自動で型チェックするため例外発生）
- **除外条件**: Zod スキーマで `z.coerce.number()` または `z.number()` による型強制があること
- **参照**: SQL別冊 2.5.2（数値リテラルに対するSQLインジェクション）
- **優先度**: Medium

---

#### 1.A.2 OSコマンド・インジェクション（IPA-1.2 / WKD-D）

##### 観点A.2.1 シェル起動系APIの使用有無 [IPA-Appl] [コード確認]

- **確認対象**: `src/` 配下を `child_process`, `exec(`, `execSync(`, `spawn(`, `spawnSync(`, `execFile(` で全文検索
- **危険と判断する条件**: `exec` / `execSync` にユーザ入力を含む文字列が渡されている
- **除外条件**: `spawn` / `execFile` で `shell: true` オプションなし、かつ引数配列でユーザ入力を分離している
- **該当コード候補**: `src/**/*.ts`、`scripts/*.ts`（`list-users.ts`, `reset-password.ts`）
- **参照**: IPA-1.2 根本的解決 2-(i)
- **優先度**: High（未確認のため最優先で存在確認）

##### 観点A.2.2 ファイル名を使った外部コマンド呼び出し [IPA-Appl] [コード確認]

- **確認対象**: `src/app/staff/blog/actions.ts` の画像処理フロー
- **危険と判断する条件**: sharp ではなく、外部コマンド（ImageMagick等）をファイル名付きで呼んでいる
- **除外条件**: sharp の Node.js API のみ使用
- **優先度**: Medium

---

#### 1.A.3 パス名パラメータの未チェック／ディレクトリ・トラバーサル（IPA-1.3 / WKD-G）

##### 観点A.3.1 ブログ画像アップロード時の保存先パス組み立て [IPA-Appl] [コード確認]

- **確認対象**: `src/app/staff/blog/actions.ts` の `path.join` や文字列連結によるパス組み立て
- **危険と判断する条件**:
  - ユーザ指定のファイル名（originalname）をそのまま使用
  - `../` 等の相対パス要素が削除されない
  - 保存先が `public/uploads/blog/` から外れる可能性（`path.resolve` での境界チェックなし）
- **除外条件**: タイムスタンプ + ランダム文字列で生成、元ファイル名は使わない、かつ `path.resolve` で保存先ディレクトリ内に収まることを検証
- **確認証跡**: `public/uploads/blog/` に存在するファイル名（`1770992461005.png`, `1770994779680.png`）からタイムスタンプ生成と推測
- **該当コード候補**: `src/app/staff/blog/actions.ts`、`src/app/staff/blog/BlogForm.tsx`
- **参照**: IPA-1.3 根本的解決 3-(i)-a, 3-(i)-b
- **優先度**: High

##### 観点A.3.2 画像表示パスのパラメータ化 [IPA-Appl] [コード確認]

- **確認対象**: `src/app/api/` や `route.ts` にファイル配信ハンドラが存在するか
- **除外条件**: Next.js の `public/` 配下の静的配信のみ
- **優先度**: Low

##### 観点A.3.3 ウェブサーバ内のファイルアクセス権限 [IPA-Appl] [コード確認]

- **確認対象**: `frontend/docker/Dockerfile.prod`、`docker/docker-compose.prod.yml` のボリューム設定、`public/uploads/` のパーミッション
- **参照**: IPA-1.3 保険的対策 3-(ii)
- **優先度**: Medium

---

#### 1.A.4 セッション管理の不備（IPA-1.4 / WKD-K）

##### 観点A.4.1 セッションIDの推測困難性 [IPA-Appl] [コード確認]

- **確認対象**: `src/lib/session.ts` の iron-session 設定、`.env.local` の `SESSION_SECRET` の長さ
- **危険と判断する条件**: `SESSION_SECRET` が32文字未満、または推測可能な値（`SEED_ADMIN_PASSWORD` 等の弱いパスワード）
- **参照**: IPA-1.4 根本的解決 4-(i)
- **優先度**: Critical `Confirmed`（`.env.local` のGit混入のため、仮に強度が十分でも既に漏洩済み扱い）

##### 観点A.4.2 セッションIDのURLパラメータ格納 [IPA-Appl] [コード確認]

- **確認対象**: `src/middleware.ts`、`src/app/staff/login/actions.ts` のリダイレクト実装
- **除外条件**: iron-session は Cookie ベースのため通常該当せず
- **優先度**: Low

##### 観点A.4.3 Secure属性 [IPA-Appl] [コード確認]

- **確認対象**: `src/lib/session.ts` の `cookieOptions.secure` の実装
- **危険と判断する条件**: `secure: false` で固定、または `NODE_ENV` の判定が間違っている
- **除外条件**: `secure: process.env.NODE_ENV === 'production'` のような条件分岐が正しく実装されている（技術レポートで記載あり `Confirmed`）
- **参照**: IPA-1.4 根本的解決 4-(iii)
- **優先度**: High

##### 観点A.4.4 ログイン成功後のセッション再生成（Session Fixation対策） [IPA-Appl] [コード確認]

- **確認対象**: `src/app/staff/login/actions.ts` のログイン処理、および `scripts/test-session-fixation.sh` の内容とテスト結果
- **危険と判断する条件**:
  - ログイン成功時に `session.destroy()` → 新規発行 → `session.save()` の順序が守られていない
  - iron-session のCookie値が変化しない（固定化を許容する実装）
- **除外条件**: `scripts/test-session-fixation.sh` が Pass しており、セッションCookie値の変化を検証している
- **参照**: IPA-1.4 根本的解決 4-(iv)-a、WKD-K 検出パターン1
- **優先度**: High

##### 観点A.4.5 ログイン後の秘密情報発行（保険的対策） [IPA-Appl] [コード確認]

レビュー指摘（Claude 3-5）対応で追加。

- **確認対象**: Next.js Server Actions のハッシュID（`Next-Action` ヘッダ）相当が、IPA-1.4 保険的対策 4-(iv)-b（既存セッションIDとは別の秘密情報をページ遷移ごとに確認）を代替しているか
- **危険と判断する条件**: Server Actions 機構への依存のみで、CSRF保護と秘密情報発行が混同されている
- **参照**: IPA-1.4 保険的対策 4-(iv)-b
- **優先度**: Medium

##### 観点A.4.6 HttpOnly属性 [IPA-Appl] [コード確認]

- **確認対象**: `src/lib/session.ts`
- **除外条件**: iron-session のデフォルト挙動 `httpOnly: true`（技術レポートで記載あり `Confirmed`）
- **参照**: IPA-1.5 保険的対策 5-(ix)
- **優先度**: Low

##### 観点A.4.7 SameSite属性 [IPA-Appl] [コード確認]

- **確認対象**: `src/lib/session.ts`、および全 `actions.ts` がServer Action（POST相当）で実装されているか
- **除外条件**: `sameSite: 'lax'` 設定（技術レポートで記載あり `Confirmed`）+ 状態変更操作がすべてPOST経由
- **優先度**: Medium

##### 観点A.4.8 セッション有効期限 [IPA-Appl] [コード確認]

- **確認対象**: `src/lib/session.ts`、`src/middleware.ts` でのセッション更新ロジック
- **危険と判断する条件**: アイドルタイムアウトとアブソリュートタイムアウトの区別がない
- **参照**: IPA-1.4 保険的対策 4-(vi)
- **優先度**: Medium

##### 観点A.4.9 言語・ミドルウェアのセッション管理機構使用 [IPA-Appl] [コード確認]

- **確認対象**: `src/lib/recovery-codes.ts`、`src/lib/auth.ts`（iron-session 以外にセッション類似の識別子を扱っていないか）
- **参照**: WKD-K 検出パターン2
- **優先度**: Medium

##### 観点A.4.10 2FAトークンの再利用防止 [IPA-Appl] [コード確認]

- **確認対象**: `src/lib/auth.ts`、`src/app/staff/login/actions.ts` のTOTP検証処理
- **危険と判断する条件**: 使用済みトークン追跡が完全にインメモリ Map のみで、プロセス再起動時に状態喪失
- **攻撃前提**: プロセス再起動直後の時間窓を捉えた能動的攻撃者
- **優先度**: Medium

---

#### 1.A.5 クロスサイト・スクリプティング（IPA-1.5 / WKD-B）

##### 観点A.5.1 Reactのデフォルトエスケープと迂回経路 [IPA-Appl] [コード確認]

レビュー指摘（Gemini 2.3, 3.1）対応で強化。

- **確認対象**: `src/` 配下を `dangerouslySetInnerHTML`, `innerHTML`, `document.write` で全文検索
- **危険と判断する条件**:
  - `dangerouslySetInnerHTML` にユーザ入力（ブログ本文、ニュース本文、Contactメッセージ等）が直接流れている
  - サニタイズライブラリ（`isomorphic-dompurify`等）を経由していない
- **除外条件**: 入力がサーバ側で HTMLサニタイズ済み、かつ構文解析木ベースのホワイトリスト処理がなされている
- **確認証跡**: `dangerouslySetInnerHTML` の使用箇所リストと、その入力ソース
- **該当コード候補**: `src/app/blog/[id]/page.tsx`、`src/app/news/[id]/page.tsx`、ブログ/ニュース本文レンダリング処理全般
- **参照**: IPA-1.5 根本的解決 5-(i), 5-(vi), 5-(vii)
- **優先度**: High

##### 観点A.5.2 href/src属性へのユーザ入力出力 [IPA-Appl] [コード確認]

- **確認対象**: ブログ/ニュースの入力フォームに対するバリデーション処理、および表示側での `href` 組み立て
- **危険と判断する条件**: `javascript:` / `data:` / `vbscript:` スキームを許容
- **除外条件**: `http://` `https://` のみホワイトリスト
- **参照**: IPA-1.5 根本的解決 5-(ii)、WKD-B 検出パターン4
- **優先度**: Medium

##### 観点A.5.3 `<script>` 要素の動的生成 [IPA-Appl] [コード確認]

- **確認対象**: `src/` 配下の `<Script>` / `<script>` 使用箇所
- **参照**: IPA-1.5 根本的解決 5-(iii)
- **優先度**: Low

##### 観点A.5.4 スタイルシート取り込みの制限 [IPA-Appl] [コード確認]

- **確認対象**: `next.config.ts` のCSP設定の `style-src` ディレクティブ
- **危険と判断する条件**: `style-src *` や `'unsafe-inline'` のみで、nonce/hash ベースでない
- **参照**: IPA-1.5 根本的解決 5-(iv)
- **優先度**: Medium

##### 観点A.5.5 Content-Type文字コード指定 [IPA-Appl] [コード確認]

- **確認対象**: `src/app/**/route.ts`（存在する場合）、カスタムレスポンスヘッダ生成
- **除外条件**: Next.js のデフォルト `text/html; charset=utf-8`
- **参照**: IPA-1.5 根本的解決 5-(viii)
- **優先度**: Low

##### 観点A.5.6 HttpOnly Cookie属性 [IPA-Appl] [コード確認]

- 観点A.4.6 と同じ。参照のみ。
- **優先度**: Low

##### 観点A.5.7 CSPの厳格性 [IPA-Appl] [コード確認]

- **確認対象**: `next.config.ts` のCSPヘッダ定義
- **危険と判断する条件**:
  - `script-src` に `'unsafe-inline'` を含む（実質CSP無効化）
  - `script-src` に JSONPエンドポイントを提供するドメインを許可（Bypass 可能）
  - nonce ベースの場合、nonce 値の予測可能性
  - `require-trusted-types-for 'script'` が未設定
- **参照**: IPA-1.5 保険的対策 5-(x)
- **優先度**: High

##### 観点A.5.8 TRACEメソッドの無効化 [IPA-Oper] [運用確認]

- **確認対象**: リバースプロキシ設定
- **優先度**: Medium `Environment-dependent`

##### 観点A.5.9 入力内容確認画面・管理画面でのエスケープ [IPA-Appl] [コード確認]

- **確認対象**: Contact送信処理のフロー、お問い合わせ管理（CL-10）一覧テーブル表示時のエスケープ
- **該当コード候補**: `src/app/contact/actions.ts`、`src/app/staff/contacts/ContactTable.tsx`
- **優先度**: High

##### 観点A.5.10 エラーページでの入力値反射 [IPA-Appl] [コード確認]

- **確認対象**: `src/app/**/error.tsx`、`actions.ts` のエラー返却
- **参照**: WKD-B 対象画面「エラー」
- **優先度**: Medium

##### 観点A.5.11 HTMLテキスト入力を許可する場合の対策 [IPA-Appl] [仕様決定] [コード確認]

レビュー指摘（Claude 3-3）対応で追加。

- **確認対象**: ブログ本文・ニュース本文の入力形式（Markdown か、HTML か、リッチエディタ経由か）
- **危険と判断する条件**:
  - HTML直接入力を許容しており、構文解析木からのスクリプト除去が未実装
  - Markdownレンダリング時に `html: true` でインラインHTMLを許容しつつサニタイズ未実施
- **参照**: IPA-1.5 根本的解決 5-(vi), 5-(vii)
- **優先度**: High

---

#### 1.A.6 CSRF（IPA-1.6 / WKD-C）

##### 観点A.6.1 Next.js Server Actions の CSRF 保護 [IPA-Appl] [コード確認]

レビュー指摘（Gemini 2.4, Claude 3-4）対応で強化。

- **確認対象**: `next.config.ts` のServer Actions関連設定、`scripts/test-csrf.sh` の内容とテスト結果
- **危険と判断する条件（WKD-C 4条件に対応）**:
  - **条件1**: トークン等のパラメータが存在しない（Server Actions のハッシュIDに依存）
  - **条件2**: トークンを削除しても処理が実行される
  - **条件3**: トークン文字列の推測が可能（`experimental.serverActions.encryptionKey` 未設定で鍵がデプロイごとに変動）
  - **条件4**: 別ユーザのトークンが使用できる
- **除外条件**:
  - `Origin` / `Referer` ヘッダチェックが有効
  - `sameSite=lax` Cookie と POST経由の状態変更操作
  - `scripts/test-csrf.sh` が Pass
- **参照**: IPA-1.6 根本的解決 6-(i)-a〜c、WKD-C 検出パターン1
- **優先度**: High

##### 観点A.6.2 特定副作用を持つ操作の洗い出し [IPA-Appl] [コード確認]

- **確認対象**: WKD-C の対象機能「パスワード変更、DB更新、メール送信」に対応する本リポジトリの操作
  - パスワード変更（CL-29、CL-25-③）
  - ブログ/ニュース/カテゴリ/ユーザ CRUD（CL-11〜CL-26）
  - お問い合わせ削除（CL-10、特に **一括削除**）
  - ログアウト
  - 2FA有効化/無効化（CL-29）
- **優先度**: High

##### 観点A.6.3 重要操作後の通知メール [IPA-Appl] [コード確認]

- **確認対象**: `src/app/staff/users/actions.ts`、`src/app/staff/settings/actions.ts`
- **危険と判断する条件**: パスワード変更、2FA設定変更、管理者権限での操作（他ユーザのパスワード発行等）の際、対象ユーザに通知メールを送る設計がない
- **参照**: IPA-1.6 保険的対策 6-(ii)
- **優先度**: Medium

##### 観点A.6.4 一括操作時のCSRF／認可／DoS [IPA-Appl] [コード確認]

レビュー指摘（Claude 11-2）対応で追加。

- **確認対象**: CL-10（お問い合わせ管理）の「選択チェックボックス」による一括削除機能
- **危険と判断する条件**:
  - 一括操作時のトランザクション性が欠落（一部失敗時の部分コミット）
  - CSRFトークンが単一操作ごとではなくリクエスト単位
  - 削除対象ID配列の上限値がない（1リクエストで10万件指定による DoS）
- **該当コード候補**: `src/app/staff/contacts/actions.ts`、`src/app/staff/contacts/ContactTable.tsx`
- **優先度**: High

##### 観点A.6.5 二重送信防止 [IPA-Appl] [コード確認]

- **確認対象**: フォーム送信ボタンのdisabled制御、サーバ側のidempotencyキー管理
- **優先度**: Low

---

#### 1.A.7 HTTPヘッダ・インジェクション（IPA-1.7 / WKD-I）

##### 観点A.7.1 リダイレクト先URLへの入力反映 [IPA-Appl] [コード確認]

- **確認対象**: `src/middleware.ts`、各 `actions.ts` のリダイレクト処理
- **危険と判断する条件**: `redirect()` 呼び出し時のURL引数にユーザ入力が直接入っている
- **除外条件**: Next.js の `redirect()` / `NextResponse.redirect()` のみ使用、かつ引数は固定文字列
- **参照**: IPA-1.7 根本的解決 7-(i)-a、WKD-I 検出パターン1, 2
- **優先度**: Medium

##### 観点A.7.2 Set-Cookieヘッダへの入力反映 [IPA-Appl] [コード確認]

- **確認対象**: `src/` 配下を `Set-Cookie`, `res.setHeader`, `cookies().set` で全文検索
- **除外条件**: iron-session 経由のみ
- **優先度**: Low

##### 観点A.7.3 カスタムレスポンスヘッダ [IPA-Appl] [コード確認]

- **確認対象**: リカバリコードダウンロード（CL-24-④）で `Content-Disposition: attachment; filename=...` を動的生成している可能性
- **危険と判断する条件**: ユーザ名やリカバリコード自体をヘッダ値に含め、改行コード検証なし
- **該当コード候補**: `src/app/staff/settings/actions.ts`、リカバリコード生成・ダウンロード処理
- **優先度**: Medium

##### 観点A.7.4 Host ヘッダインジェクション [IPA-Appl] [コード確認]

レビュー指摘（Claude 7-4）対応で追加。

- **確認対象**: 「管理者によるパスワード発行」フロー（CL-25-①〜④）で絶対URLを生成している箇所
- **危険と判断する条件**: `request.headers.get('host')` を信頼して絶対URLを生成している
- **除外条件**: 絶対URLは固定の `NEXT_PUBLIC_SITE_URL` 等から生成、または相対URLのみ使用
- **参照**: IPA-1.7（拡張観点）
- **優先度**: Medium

---

#### 1.A.8 メールヘッダ・インジェクション（IPA-1.8 / WKD-F）

##### 観点A.8.1 メール送信機能の有無と実装 [IPA-Appl] [コード確認]

- **確認対象**: `frontend/package.json`、`src/` 配下を `sendMail`, `nodemailer`, `SMTP`, `SES`, `SendGrid` で検索
- **危険と判断する条件**: メール送信が実装されており、かつ以下のいずれか
  - 外部入力（名前、メールアドレス、件名）が `To`, `From`, `Subject` ヘッダに直接含まれる
  - メールヘッダ固定・本文のみ外部入力のパターンになっていない（IPA-1.8 根本的解決 8-(i)-a 対応）
  - メール送信用API（Node.jsの環境や言語に用意されているもの）を使わず、SMTPコマンドを手組みしている
- **除外条件**:
  - メール送信未実装
  - メールヘッダを固定値にして、外部入力はすべて本文に出力（IPA-1.8 根本的解決 8-(i)-a）
  - メール送信用API使用（IPA-1.8 根本的解決 8-(i)-b）
- **参照**: IPA-1.8 全般、WKD-F 全般
- **優先度**: High（メール送信が実装されている場合）/ Low（未実装）

##### 観点A.8.2 お問い合わせフォームからの自動返信/管理者通知 [IPA-Appl] [コード確認]

- **確認対象**: `src/app/contact/actions.ts` のメール送信処理の有無
- **優先度**: High（同上）

##### 観点A.8.3 HTMLで宛先を指定しない [IPA-Appl] [コード確認]

- **確認対象**: メール送信フォームで `<input type="hidden" name="to" value="...">` のように宛先をHTMLに埋め込んでいないか
- **参照**: IPA-1.8 根本的解決 8-(ii)
- **優先度**: Medium

---

#### 1.A.9 クリックジャッキング（IPA-1.9）

##### 観点A.9.1 X-Frame-Options / frame-ancestors [IPA-Appl] [コード確認]

- **確認対象**: `next.config.ts` のヘッダ設定、CSPの内容
- **除外条件**: `X-Frame-Options: SAMEORIGIN` 設定済み `Confirmed`、CSPの `frame-ancestors` も設定済みであればさらに良い
- **参照**: IPA-1.9 根本的解決 9-(i)-a
- **優先度**: Medium

##### 観点A.9.2 重要操作での再認証 [IPA-Appl] [コード確認]

レビュー指摘（Claude 11-5）対応で強化。

- **確認対象**:
  - パスワード変更（CL-29）: 現在のパスワード入力必須か
  - **2FA無効化（CL-29）**: 再認証（パスワード + 現在の2FAコード）が必須か、1ボタンで無効化可能か
  - 他ユーザ削除（CL-26）: 管理者パスワード再入力があるか
- **危険と判断する条件**: CL-29 の 2FA「無効化」ボタンが1クリックで実行される設計
- **参照**: IPA-1.9 根本的解決 9-(i)-b
- **優先度**: High

##### 観点A.9.3 マウスのみで完結しない操作 [IPA-Appl] [コード確認]

- **確認対象**: 削除確認ダイアログ（CL-14, CL-17b, CL-21, CL-26-①）の UI 実装
- **参照**: IPA-1.9 保険的対策 9-(ii)
- **優先度**: Low

---

#### 1.A.10 バッファオーバーフロー（IPA-1.10）

##### 観点A.10.1 使用言語のメモリ安全性 [IPA-Appl] [コード確認]

- **確認対象**: `pnpm-lock.yaml` 内のバージョン確認、Trivyスキャン等での依存関係CVE確認
- **除外条件**: TypeScript/Node.js は直接メモリアクセスをしない（IPA-1.10 根本的解決 10-(i)-a 該当）
- **危険と判断する条件**: ネイティブバインディング（better-sqlite3, sharp）に既知脆弱性
- **参照**: IPA-1.10 根本的解決 10-(ii)
- **優先度**: Medium

---

#### 1.A.11 アクセス制御や認可制御の欠落（IPA-1.11 / WKD-L）

WKD-L の3検出パターンを独立観点として分離（レビュー指摘 3-4 対応）。

##### 観点A.11.1 認証機能の網羅 [IPA-Appl] [コード確認]

- **確認対象**: `src/middleware.ts` の `matcher` 設定、および `src/app/staff/` 配下の全ディレクトリ
- **危険と判断する条件**: middleware のパスマッチング設定が不完全で、一部 `/staff/*` 配下が未保護
- **参照**: IPA-1.11 根本的解決 11-(i)
- **優先度**: High

##### 観点A.11.2 ロールベース認可（admin/staff） [IPA-Appl] [コード確認]

- **確認対象**: `src/lib/auth.ts` の `requireAdmin` 実装、および各 admin 専用画面の `page.tsx` / `actions.ts` での呼び出し
- **対象画面**: CL-08（管理者ダッシュボード）、CL-22a/b、CL-23〜26、CL-27、CL-28
- **危険と判断する条件**: admin限定機能（ユーザ管理、監査ログ）が `requireAdmin()` で保護されていない
- **参照**: IPA-1.11 根本的解決 11-(ii)、WKD-L 検出パターン1（URL操作による権限外ページアクセス）
- **優先度**: Critical

##### 観点A.11.3 IDOR（Insecure Direct Object Reference） [IPA-Appl] [仕様決定] [コード確認]

レビュー指摘（ChatGPT A-3）対応で判定を二段化。

- **【設計確認】**: Blog、News等の記事に対し、author単位の権限分離が要件か、staff全員が全記事を編集可能な設計か
- **【実装確認】**: author単位の権限分離が要件の場合に、実装されていなければ脆弱性
- **確認対象**: Blog モデルに `authorId` 相当のフィールドがあるか（Prisma schema）、`actions.ts` の更新時に自分の記事かの確認があるか
- **参照**: WKD-L 検出パターン2（ID類の改変）
- **該当コード候補**: `frontend/prisma/schema.prisma`、`src/app/staff/blog/actions.ts`
- **優先度**: Medium（設計次第）

##### 観点A.11.4 自分自身の削除/ロール変更の禁止 [IPA-Appl] [コード確認]

- **確認対象**: `src/app/staff/users/actions.ts` の削除・ロール変更処理
- **危険と判断する条件**: UIレベルでは制御されているが、サーバ側でのガードがない、または最後の管理者を削除可能
- **除外条件**: CL-22a で「ログイン中ユーザーの行には削除操作が存在しない」と記載あり `Confirmed`。サーバ側でも同様のチェックがあること
- **優先度**: High

##### 観点A.11.5 Server Actions の直接呼び出し保護 [IPA-Appl] [コード確認]

- **確認対象**: 全 `actions.ts` の関数先頭で `requireAuth()` / `requireAdmin()` が呼ばれているか
- **危険と判断する条件**: middleware のみに頼った認可で、Server Action 内では認証チェック未実施
- **優先度**: Critical

##### 観点A.11.6 監査ログ閲覧権限 [IPA-Appl] [コード確認]

- **確認対象**: `src/app/staff/audit-logs/page.tsx`
- **除外条件**: `requireAdmin()` 呼び出しあり
- **優先度**: High

##### 観点A.11.7 hidden/Cookie の権限クラス改変 [IPA-Appl] [コード確認]

レビュー指摘（Claude 3-4）対応で追加。

- **確認対象**: hidden input、Cookie に現在の権限（`role=admin` 等）を格納している箇所
- **危険と判断する条件**: クライアント側で書き換え可能な権限情報をサーバ側で信頼している
- **除外条件**: 権限情報は iron-session の暗号化Cookie内のみに保持
- **参照**: WKD-L 検出パターン3
- **優先度**: Medium

---

#### 1.A.12 ウェブ健康診断13項目のうち、IPA 11脆弱性対応外のもの

##### 観点A.12.1 ディレクトリ・リスティング（WKD-E） [IPA-Appl] [運用確認]

- **確認対象**: `public/uploads/blog/` へのブラウザアクセス時の挙動、Next.js の本番ビルド出力時のディレクトリ配信設定
- **優先度**: Medium

##### 観点A.12.2 意図しないリダイレクト（WKD-H） [IPA-Appl] [コード確認]

- **確認対象**: `src/middleware.ts`、`src/app/staff/login/actions.ts` のリダイレクト先決定ロジック
- **危険と判断する条件**: ログイン後のリダイレクト先パラメータ（`?redirect=/staff/...`）が外部ドメインも許容
- **除外条件**: リダイレクト先が `/` 始まりの内部パスのみホワイトリスト
- **優先度**: Medium

##### 観点A.12.3 認証（WKD-J） [IPA-Appl] [コード確認]

WKD-J の検出パターンを独立観点として分離（レビュー指摘 3-4, A-4 対応）。

- **J-1 パスワード最大文字数 8文字以上**: CL-25-③、CL-29 の「文字数制限の表示」で確認
- **J-2 パスワード文字種**: 数字のみ・英字のみ禁止のバリデーションがあるか
- **J-3 パスワード伏字**: CL-06, CL-25-③, CL-29 で `type="password"` + 表示切替アイコン `Confirmed`
- **J-4 認証エラーメッセージ**: 「ユーザIDが違います」「パスワードが違います」と区別していないか → ユーザ列挙攻撃対策
  - **【強化】**: アカウント作成時・パスワードリセット時（本システムでは管理者が発行）のメッセージも同等に確認
- **J-5 ログアウト機能**: 左サイドナビに存在 `Confirmed`。ログアウト後「戻る」ボタンでセッション再開できないか
- **J-6 アカウントロック**: `User.isLocked`, `lockedAt`, `lockedUntil`, `loginAttempts` フィールドあり `Confirmed`。10回連続失敗時の挙動
- **J-7 パスワードポリシー追加確認項目**（レビュー指摘 A-4）:
  - 初期PWの有効期限
  - one-time password の保存形式（平文/ハッシュ/暗号化）
  - パスワード変更時の現行PW必須有無
  - MFA無効化時の再認証有無
- **優先度**: High（全体として）

##### 観点A.12.4 クローラへの耐性（WKD-M） [IPA-Appl] [コード確認]

- **確認対象**: `src/app/**/page.tsx` のキャッシュ設定、`robots.ts` / `sitemap.ts` の内容
- **優先度**: Low

---

#### 1.A.13 フィッシング詐欺を助長しないための対策（IPA-2.4）

レビュー指摘（Claude 3-1, 10-5）対応で新規追加。

##### 観点A.13.1 運営者情報の明示 [IPA-Oper] [コード確認] [仕様決定]

- **確認対象**: About画面（CL-02）の項目（会社名、所在地、連絡先）
- **危険と判断する条件**: 運営者情報の記載がない、または代表挨拶のみで所在地等が不明
- **参照**: IPA-2.4
- **優先度**: Medium

##### 観点A.13.2 TLS証明書の種類 [IPA-Oper] [運用確認]

- **確認対象**: 本番環境で使用予定の証明書タイプ
- **危険と判断する条件**: DV証明書のみ（フィッシングサイトとの識別困難）
- **除外条件**: EV証明書または OV証明書の採用
- **優先度**: Low `Environment-dependent`

##### 観点A.13.3 メール認証（SPF / DKIM / DMARC） [IPA-Oper] [運用確認]

- **確認対象**: メール送信実装がある場合の SPF / DKIM / DMARC 設定
- **優先度**: Medium `Environment-dependent`（メール送信有無が先決）

##### 観点A.13.4 ログインURLの固定化 [IPA-Oper] [コード確認]

- **確認対象**: `src/middleware.ts`、`src/app/staff/login/actions.ts` のリダイレクト処理
- **危険と判断する条件**: 任意ドメインからのリダイレクトを許容するログイン URL
- **除外条件**: ログインURLが固定、かつリダイレクト先が内部パスのみ
- **優先度**: Medium

---

#### 1.A.14 情報露出・エラー管理（独立項目化）

レビュー指摘（Gemini 2.1）対応で新規追加。

##### 観点A.14.1 サーバ・フレームワークのバナー情報 [IPA-Appl] [コード確認]

- **確認対象**: HTTPレスポンスヘッダ
- **除外条件**:
  - `poweredByHeader: false` 設定 `Confirmed`（`X-Powered-By` 無効化）
  - リバースプロキシ側でも Nginx/Apache のバージョン情報非表示
- **優先度**: Medium

##### 観点A.14.2 本番環境のスタックトレース表示 [IPA-Appl] [コード確認]

- **確認対象**: `src/app/**/error.tsx` のエラーハンドリング、本番ビルド時の Next.js 挙動
- **危険と判断する条件**: 本番環境でクライアントにスタックトレースが表示される
- **除外条件**: カスタム `error.tsx` で汎用エラーメッセージに置換
- **優先度**: High

##### 観点A.14.3 開発用デバッグ出力の残存 [IPA-Appl] [コード確認]

- **確認対象**: `console.log`, `debugger`, `TODO`, `FIXME` の残存（3.10 grep観点で横断的に実施）
- **優先度**: Medium

---


### 1.B インフラセキュリティ（コンテナ／オーケストレーション層）

Docker Compose ベースの構成に対する、コンテナランタイム・イメージ・secrets管理等の観点。repo内根拠を持つ項目に限定（レビュー指摘 B〜G-1 対応）。

---

#### 観点B.1 コンテナランタイムの権限制限 [コード確認]

- **確認対象**: `docker/docker-compose.prod.yml` に以下が設定されているか
  - `cap_drop: [ALL]` + 必要最小の `cap_add`
  - `security_opt: [no-new-privileges:true]`
  - `read_only: true`（ルートファイルシステムを読み取り専用）
  - `tmpfs`（書き込みが必要な箇所の分離）
  - `user` 指定
- **危険と判断する条件**: 上記が未設定、または緩い設定
- **該当ファイル**: `docker/docker-compose.prod.yml`、`frontend/docker/Dockerfile.prod`
- **優先度**: Medium

#### 観点B.2 コンテナのファイルシステム書き込み領域の明確化 [コード確認]

レビュー指摘（Claude 6-3）対応で追加。

- **確認対象**: 本リポジトリで以下の書き込みが発生する箇所の分離設計
  - `public/uploads/blog/`（ブログ画像アップロード）
  - `dev.db`（SQLite、本番運用時の永続化先は未定義）
  - Next.js の `.next/cache/`（ランタイムキャッシュ）
- **危険と判断する条件**: `read_only: true` を指定せず、ルートFS全体が書き込み可能
- **除外条件**: 書き込み先を Volume マウントで分離、それ以外は read-only
- **該当ファイル**: `docker/docker-compose.prod.yml`
- **優先度**: Medium

#### 観点B.3 コンテナイメージの脆弱性スキャン [コード確認] [運用確認]

- **確認対象**: ベースイメージ（`node:20-alpine`）およびインストール済みパッケージのCVE
- **危険と判断する条件**: Trivy/Grype 等でのイメージスキャンが未実施、または重大CVE放置
- **該当ファイル**: `frontend/docker/Dockerfile.prod`、`frontend/docker/Dockerfile.dev`、CI/CD設定
- **優先度**: Medium

#### 観点B.4 イメージビルドの再現性と成果物の最小化 [コード確認]

- **確認対象**: `frontend/docker/Dockerfile.prod` の FROM 句、各ステージの COPY 内容、runner ステージの最終構成
- **危険と判断する条件**:
  - ベースイメージがタグのみ（`node:20-alpine`）で digest 未固定
  - runner ステージに不要なビルドツール（python3, make, g++）が残存
- **除外条件**: `FROM node:20-alpine@sha256:...` digest 固定、multi-stage で runner に最小構成のみ
- **該当ファイル**: `frontend/docker/Dockerfile.prod`
- **優先度**: Medium

#### 観点B.5 Alpine のパッケージ署名検証 [コード確認]

レビュー指摘（Claude 6-4）対応で追加。

- **確認対象**: `Dockerfile.prod` 内の `apk add` 時のバージョン固定、`/etc/apk/keys/` の署名検証
- **危険と判断する条件**:
  - `apk add package` でバージョン固定なし
  - パッケージ署名検証を無効化する `--allow-untrusted` の使用
- **除外条件**: `apk add --no-cache package=version` 形式
- **該当ファイル**: `frontend/docker/Dockerfile.prod`
- **優先度**: Medium

#### 観点B.6 Docker Content Trust / イメージ署名 [運用確認]

レビュー指摘（Claude 6-5）対応で追加。

- **確認対象**: ビルド済みイメージのレジストリ push 時の署名・検証運用
- **危険と判断する条件**: 署名なしイメージが本番レジストリへ push される
- **除外条件**: cosign / notation / Docker Content Trust による署名運用
- **優先度**: Medium `Environment-dependent`

#### 観点B.7 secrets管理 [運用確認]

- **確認対象**:
  - `docker/docker-compose.prod.yml` の環境変数関連設定
  - 本番運用ドキュメント（`docs/` 配下）の有無と内容
  - ビルド時と実行時で secrets の扱いが分離されているか
- **危険と判断する条件**: `docker-compose.prod.yml` に `env_file` 指定がなく、環境変数の注入方法が別途運用で管理される想定 `Confirmed`。運用手順が未定義の場合、平文コピーの誤運用リスク。
- **該当ファイル**: `docker/docker-compose.prod.yml`、`docs/architecture/`、`docs/process/`
- **優先度**: High `Environment-dependent`

#### 観点B.8 コンテナ間通信とネットワーク分離 [コード確認]

- **確認対象**: `docker-compose.prod.yml` の `networks`、`ports`、`expose` 設定
- **危険と判断する条件**: アプリコンテナの3000番が `ports` でホスト外部に直接公開されている（リバースプロキシ経由にすべき）
- **該当ファイル**: `docker/docker-compose.prod.yml`、`docker/docker-compose.dev.yml`
- **優先度**: Medium

#### 観点B.9 `.dockerignore` の妥当性 [コード確認]

- **確認対象**: `frontend/.dockerignore` の内容
- **危険と判断する条件**: 以下がignoreされていない
  - `.env*`
  - `.git`
  - `node_modules`
  - `*.backup`, `*.20*`（日付サフィックス系）
  - `*.db`, `dev.db`, `dev.db-journal`, `dev.db-wal`, `dev.db-shm`
  - `.next`（ビルド出力）
  - `docs/`、`README.md`
- **該当ファイル**: `frontend/.dockerignore`
- **優先度**: High

#### 観点B.10 コンテナのヘルスチェックとリスタートポリシー [コード確認]

- **確認対象**: `docker-compose.prod.yml` の `healthcheck`、`restart` ポリシー、アプリケーション側の `/health` エンドポイント有無
- **注記**: リスタート時にインメモリ状態（Rate Limit、TOTP使用済みトークン）が失われる副作用を考慮した設計か
- **該当ファイル**: `docker/docker-compose.prod.yml`
- **優先度**: Medium

#### 観点B.11 ログの永続化と集約 [運用確認]

- **確認対象**: `docker-compose.prod.yml` の `logging` 設定
- **優先度**: Low `Environment-dependent`

#### 観点B.12 開発環境と本番環境の分離 [コード確認]

- **確認対象**: 両 Compose ファイルの差分、本番でdev compose を使うリスク
- **優先度**: Low

---

### 1.C サーバセキュリティ（コンテナ内部OS／ランタイム層）

**コンテナ内部** のAlpine Linux、Node.jsランタイム、ファイルシステムに関する観点（ホストOSは 1.G へ分離）。

---

#### 観点C.1 ベースイメージのハードニング [コード確認]

- **確認対象**: `Dockerfile.prod`、`Dockerfile.dev` の RUN 命令、ベースイメージタグの鮮度
- **危険と判断する条件**: Alpine Linux のセキュリティアップデートが反映されていない
- **除外条件**: `apk update && apk upgrade` がビルド時に実施、ベースイメージの digest 固定
- **該当ファイル**: `frontend/docker/Dockerfile.prod`、`frontend/docker/Dockerfile.dev`
- **優先度**: Medium

#### 観点C.2 非rootユーザ実行と権限分離 [コード確認]

- **確認対象**:
  - `Dockerfile.prod` の `USER nextjs` 指示の位置
  - `chown`、`chmod` 設定
  - `public/uploads/blog/` への書き込み権限
- **除外条件**: `実行ユーザー: nextjs（UID 1001、非root）` `Confirmed`
- **該当ファイル**: `frontend/docker/Dockerfile.prod`
- **優先度**: High

#### 観点C.3 Node.jsランタイムのセキュリティオプション [コード確認]

- **確認対象**: `Dockerfile.prod` の `ENV NODE_OPTIONS`、`package.json` の scripts
- **確認候補オプション**:
  - `--disable-proto=delete`（`Object.prototype` への細工防止、Prototype Pollution 対策）
  - `--force-node-api-uncaught-exceptions-policy=true`
  - `--experimental-permission`（Node.js 20+、ファイル・ネットワーク権限制限）
- **該当ファイル**: `frontend/docker/Dockerfile.prod`、`frontend/package.json`
- **優先度**: Medium

#### 観点C.4 Alpine Linux の musl libc 固有リスク [コード確認]

レビュー指摘（Claude 6-1）対応で追加。

- **確認対象**: 使用中の musl libc バージョンと既知CVE、Node.jsネイティブアドオン（better-sqlite3、sharp）の Alpine 向けビルド状況
- **危険と判断する条件**: musl libc 固有の脆弱性（過去例: CVE-2019-14697）が未パッチ、またはネイティブアドオンが Alpine でテストされていない
- **優先度**: Medium

#### 観点C.5 ファイルシステム権限 [コード確認]

- **確認対象**: 以下のパスの権限設計
  - `public/uploads/blog/`（書き込み可、nextjs所有）
  - `dev.db`（開発環境のみ、コンテナ外からのアクセス制限）
  - `.next/cache/`（Next.jsビルドキャッシュ）
  - `node_modules/`（読み取り専用）
- **優先度**: Medium

#### 観点C.6 不要なバイナリ・パッケージの除去 [コード確認]

- **確認対象**: `Dockerfile.prod` のrunnerステージ定義、`RUN apk add` の後に `rm -rf /var/cache/apk/*` があるか
- **該当ファイル**: `frontend/docker/Dockerfile.prod`
- **優先度**: Medium

#### 観点C.7 プロセス管理（init process、ゾンビプロセス対策） [コード確認]

レビュー指摘（Claude 6-2）対応で優先度 Low → Medium に引き上げ。

- **確認対象**: `Dockerfile.prod` の ENTRYPOINT/CMD、docker-compose の `init: true`、`dumb-init` / `tini` の使用
- **危険と判断する条件**: `node server.js` を PID 1 で直接起動、`init: true` 未設定
- **想定される影響**: SIGTERM / SIGINT の取りこぼしによるゼロダウンタイムデプロイの失敗、ゾンビプロセスの蓄積
- **優先度**: Medium

#### 観点C.8 タイムゾーンとロケール [コード確認]

- **確認対象**: `Dockerfile` の `ENV TZ=UTC`、`date` コマンド出力
- **優先度**: Low

---

### 1.D ネットワークセキュリティ

TLS、DNS、リバースプロキシ、WAFなどのネットワーク層の観点。リバースプロキシ設定はリポジトリ内に存在しない `Confirmed` ため、多くの観点が `Environment-dependent`。

---

#### 観点D.1 TLS/SSL設定（通信の暗号化） [運用確認]

レビュー指摘（Claude 7-6）対応で具体項目を掘り下げ。

- **確認対象**:
  - 対応プロトコル（TLS 1.2以上、1.3推奨、1.0/1.1 完全無効化）
  - 対応暗号スイート（`ECDHE-ECDSA-AES128-GCM-SHA256` 等のPFS対応のみ、`RC4`, `3DES` 完全排除）
  - OCSP Stapling、Session Resumption の有効化
  - Certificate Transparency（CT）対応
  - HSTS preload 登録（D.2 と連携）
  - 証明書の鍵長（RSA 2048以上、ECDSA P-256 推奨）
  - SAN（Subject Alternative Name）の適切性
- **危険と判断する条件**: TLS 1.0/1.1 の有効化、弱い暗号スイート許容、DV証明書のみ
- **参照**: IPA-2.3
- **優先度**: High `Environment-dependent`

#### 観点D.2 HSTSの設定 [コード確認]

- **確認対象**: `next.config.ts` のHSTSヘッダ定義
- **除外条件**: `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` 設定済み `Confirmed`
- **注記**: `includeSubDomains` によりサブドメインも対象となるため、サブドメインでHTTP運用があれば問題
- **該当ファイル**: `frontend/next.config.ts`
- **優先度**: Medium

#### 観点D.3 リバースプロキシの必要性と設計 [運用確認]

- **確認対象**: 本番構成でのリバースプロキシ配置、`X-Forwarded-For`, `X-Forwarded-Proto`, `X-Real-IP` の扱い
- **危険と判断する条件**: Next.js サーバを直接外部公開（TLS終端、ヘッダ上書き防止、レート制限が不在）
- **優先度**: High `Environment-dependent`

#### 観点D.4 ポート公開範囲 [コード確認] [運用確認]

- **確認対象**: `docker-compose.prod.yml` の `ports`（ホストバインド）と `expose`（内部公開のみ）の使い分け、ホストファイアウォール設定
- **危険と判断する条件**: アプリケーションコンテナの3000番ポートがホスト外部に直接公開
- **該当ファイル**: `docker/docker-compose.prod.yml`
- **優先度**: High

#### 観点D.5 DNS設定 [運用確認]

- **確認対象**: DNSSEC、CAAレコード、SPF/DKIM/DMARC、DNSキャッシュポイズニング対策
- **参照**: IPA-2.2
- **優先度**: Low `Environment-dependent`

#### 観点D.6 WAF（Web Application Firewall） [運用確認]

- **確認対象**: ModSecurity、AWS WAF、Cloudflare、Imperva 等の配置方針
- **参照**: IPA-2.6
- **優先度**: Medium `Environment-dependent`

#### 観点D.7 Rate Limiting の多層化 [運用確認]

- **確認対象**: リバースプロキシ/CDN/WAF 側のレート制限設定
- **注記**: アプリケーション層のインメモリ Rate Limit（観点2.A.5）は水平スケール時に破綻
- **優先度**: Medium

#### 観点D.8 CORS設定とオリジン制御 [コード確認]

- **確認対象**: `next.config.ts` の `headers()` でのCORS設定、`src/app/**/route.ts` の `Access-Control-*` ヘッダ
- **注記**: Route Handler の存在確認が先決（観点F.7 参照）
- **優先度**: Medium（Route Handler 存在時）

#### 観点D.9 公開エンドポイントのDDoS対策 [コード確認] [運用確認]

- **確認対象**: `src/app/contact/actions.ts`、`src/lib/rate-limit.ts`、CDN/WAF設定
- **危険と判断する条件**: Contact（`/contact`）が認証なしで外部POSTを受け付け、アプリ層のインメモリRate Limit のみで防御
- **該当ファイル**: `src/app/contact/actions.ts`、`src/lib/rate-limit.ts`
- **優先度**: High

#### 観点D.10 HTTP/2 Rapid Reset 攻撃（CVE-2023-44487） [運用確認]

レビュー指摘（Claude 7-1）対応で追加。

- **確認対象**: Node.js のバージョン（20系でのパッチ適用）、リバースプロキシ側のHTTP/2設定、CDN/WAFでのmitigation
- **危険と判断する条件**: HTTP/2 有効化済みかつパッチ未適用
- **優先度**: Medium `Environment-dependent`

#### 観点D.11 DNS リバインディング攻撃 [コード確認] [運用確認]

レビュー指摘（Claude 7-2）対応で追加。

- **確認対象**: `Host` ヘッダの検証（`next.config.ts` の `allowedHosts` 相当設定、または nginx 側の `server_name` 厳格化）
- **危険と判断する条件**: 任意の `Host` ヘッダを受け入れる設定
- **優先度**: Medium

#### 観点D.12 Subdomain Takeover [運用確認]

レビュー指摘（Claude 7-3）対応で追加。

- **確認対象**: サブドメイン運用方針、CDN解除時のDNSレコード削除プロセス、iron-session Cookie の `Domain` 属性指定有無
- **危険と判断する条件**: `Domain=example.com` でCookieが親ドメイン全体に共有され、子ドメイン奪取でセッションCookieが窃取されるリスク
- **除外条件**: Cookie の `Domain` 属性を指定せず、登録ドメインのみに限定
- **優先度**: Medium `Environment-dependent`

#### 観点D.13 Slowloris 等の低速DoS [運用確認]

レビュー指摘（Claude 7-5）対応で追加。

- **確認対象**: Next.jsのstandaloneモードでの `server.js` の挙動（`server.headersTimeout`, `server.requestTimeout`）、リバースプロキシ側のタイムアウト・keepalive 制限
- **優先度**: Medium `Environment-dependent`

#### 観点D.14 内部ネットワークからの攻撃面 [仕様決定]

- **確認対象**: 将来のPostgreSQL移行時の接続設定
- **優先度**: Low（現時点）

---

### 1.E データベースセキュリティ

SQLite/Prismaを中心に、将来のPostgreSQL移行も見据えたDB層の観点。

---

#### 観点E.1 データ暗号化（at-rest） [運用確認]

- **確認対象**:
  - `prisma.config.ts` の接続オプション（SQLCipher使用時は `?cipher=...`）
  - 本番でSQLiteファイルを配置するボリュームの暗号化状況
  - PostgreSQL移行時のTDE方針
- **該当ファイル**: `frontend/prisma.config.ts`、`docker/docker-compose.prod.yml`
- **優先度**: Medium `Environment-dependent`

#### 観点E.2 SQLite ジャーナル・WAL ファイルの扱い [コード確認]

レビュー指摘（Claude 8-2）対応で追加。

- **確認対象**:
  - `.gitignore` / `.dockerignore` の `dev.db*`（ワイルドカード）設定
  - SQLCipher使用時の補助ファイル（`dev.db-journal`, `dev.db-wal`, `dev.db-shm`）暗号化挙動
  - バックアップ取得時の補助ファイル整合性
- **危険と判断する条件**: 本体 `dev.db` のみ ignore され、WAL/journal が Git管理される
- **優先度**: Medium

#### 観点E.3 接続暗号化（in-transit） [仕様決定]

- **確認対象**: `.env.local.example` の `DATABASE_URL` 形式、移行計画ドキュメント
- **参照**: IPA-2.3
- **優先度**: Low（現時点・SQLite）

#### 観点E.4 DBアカウント権限（最小権限原則） [仕様決定]

- **確認対象**: PostgreSQL移行時の権限設計
- **参照**: IPA-1.1 保険的対策 1-(iv)
- **優先度**: Medium

#### 観点E.5 機密カラムのDB内保護 [コード確認]

- **確認対象**: 以下のカラムの保存形式
  - `users.passwordHash`（bcryptハッシュ `Confirmed`）
  - `users.twoFactorSecret`（AES-256-GCMで暗号化済みと記載あり `Confirmed`）
  - `users.recoveryCodesHash`（bcryptハッシュ配列をJSON保存 `Confirmed`）
  - `users.oneTimePassword`（**暗号化/ハッシュ化されているか要確認** `Suspected`）
  - `audit_logs.ipAddress`, `audit_logs.userAgent`（個人情報該当性）
- **危険と判断する条件**: `oneTimePassword` が平文保存
- **該当ファイル**: `frontend/prisma/schema.prisma`、`src/lib/crypto.ts`、`src/lib/auth.ts`
- **優先度**: High

#### 観点E.6 バックアップファイルの管理 [コード確認] [運用確認]

- **確認対象**: バックアップ運用ドキュメント、`.gitignore`、`.dockerignore`、`frontend/dev.db` の現状
- **該当ファイル**: `frontend/.gitignore`、`frontend/.dockerignore`、`frontend/dev.db`
- **優先度**: Medium

#### 観点E.7 Prismaのクエリログ・エラーログ [コード確認]

- **確認対象**: `src/lib/prisma.ts` の PrismaClient 初期化オプション、環境変数での制御
- **危険と判断する条件**: 本番環境で `log: ['query']` 等が有効で、`$queryRaw` のパラメータが平文出力される
- **該当ファイル**: `frontend/src/lib/prisma.ts`
- **優先度**: High

#### 観点E.8 Prismaの `where` 条件における型安全性 [コード確認]

レビュー指摘（Claude 8-1）対応で追加。

- **確認対象**:
  - `in` 演算子への配列注入（`where: { id: { in: userInputArray } }` の長さ制限）
  - `contains` 演算子の挙動差（SQLite → PostgreSQL 移行時）
  - `OR` 条件の動的構築（クエリプラン変動、メモリ消費）
- **危険と判断する条件**: ユーザ入力配列に長さ制限なし、または `OR` の数が動的に増える
- **優先度**: Medium

#### 観点E.9 `X-Forwarded-For` の検証と IP スプーフィング [コード確認]

レビュー指摘（Claude 8-3）対応で追加。

- **確認対象**: `src/lib/audit-log.ts` での IP 取得ロジック
- **危険と判断する条件**:
  - `X-Forwarded-For` を無条件信頼（リバースプロキシ未設定時、攻撃者任意の値）
  - 複数IPがカンマ区切りで含まれる場合の順序（左から右）とどのIPを信用するかの定義がない
- **除外条件**: リバースプロキシ経由を前提に、最右のproxy直前のIPのみ信頼、または `request.socket.remoteAddress` を使用
- **該当ファイル**: `src/lib/audit-log.ts`
- **優先度**: High

#### 観点E.10 マイグレーション時のデータ整合性（Down migration 欠落） [コード確認]

レビュー指摘（Claude 8-4）対応で強化。

- **確認対象**:
  - マイグレーション `20260214020000_replace_admin_with_user/migration.sql` の内容
  - 旧 `admin` テーブルのデータ移行・削除状況
  - Prisma Migrate は Rollback（Down migration）を標準サポートしないため、マイグレーション失敗時の復元手順
- **危険と判断する条件**: マイグレーション失敗時の復元手順が未定義、ステージングでのリハーサルなし
- **該当ファイル**: `frontend/prisma/migrations/20260214020000_replace_admin_with_user/migration.sql`
- **優先度**: Medium

#### 観点E.11 Prismaクエリのタイムアウトとサーキットブレーカー [コード確認]

レビュー指摘（Claude 8-5）対応で追加。

- **確認対象**: 長時間クエリの検知・強制停止、接続プール枯渇時の挙動
- **危険と判断する条件**: better-sqlite3 は同期APIのため、`Promise.race` でのタイムアウト実装が困難で DoS 耐性が低い
- **優先度**: Low（可用性観点）

#### 観点E.12 SQLiteの同時書き込み制約 [コード確認]

- **確認対象**: better-sqlite3 の初期化オプション、`PRAGMA journal_mode=WAL`、`busy_timeout` 設定
- **該当ファイル**: `src/lib/prisma.ts`、`frontend/prisma.config.ts`
- **優先度**: Low（可用性観点）

#### 観点E.13 個人情報の保持期間とパージ [仕様決定]

- **確認対象**: `contacts` テーブル、`audit_logs` テーブルの保持期間設計
- **優先度**: Low

---

### 1.F フレームワーク／言語ランタイムセキュリティ

Next.js 16、React 19、Prisma 7、iron-session、otplib、bcryptjs、および **Node.js / TypeScript 言語固有の観点** を扱う。ver 4.0 では Node.js / TypeScript 固有観点を大幅強化（レビュー指摘 4-1〜4-5 対応）。

レビュー指摘（ChatGPT F-1）対応で、F章を2グループに分離：
- **F-a: repo内で判定できる事項**
- **F-b: バージョン/CVE照合が必要な事項**

---

#### F-a: repo内で判定できる事項

##### 観点F.2 Server Components / Client Components の境界 [コード確認]

- **確認対象**:
  - `'use client'` 宣言のあるファイルで `process.env.XXX_SECRET` 等のシークレット参照がないか
  - `src/lib/session.ts`, `src/lib/crypto.ts`, `src/lib/auth.ts` 等の機密モジュールに `import 'server-only'` 宣言があるか
  - Server Component から Client Component への props 受け渡しで、不要な内部データ（userモデル全体）が含まれていないか
- **危険と判断する条件**: シークレットがクライアントバンドルに混入
- **該当ファイル**: `src/lib/` 全体、`src/app/**/*.tsx` の先頭行
- **優先度**: High（F.2 / F.9 はレビュー指摘 F-2 対応で 2章に昇格候補）

##### 観点F.3 Next.js Middleware の適用範囲と制約 [コード確認]

レビュー指摘（Claude 5-5）対応で強化。

- **確認対象**:
  - `src/middleware.ts` の `matcher` 配列
  - `src/app/staff/` 配下のディレクトリ構造との突合
  - Middleware 内部で `src/lib/crypto.ts` のAES-256-GCM関数が呼ばれていないか（呼ばれるとEdge Runtime で落ちる）
  - iron-session の Edge Runtime 対応状況
- **危険と判断する条件**: `matcher` 設定の誤りで保護漏れ
- **該当ファイル**: `frontend/src/middleware.ts`
- **優先度**: Critical

##### 観点F.5 Route Handler vs Server Action の使い分け [コード確認]

レビュー指摘（Claude 5-7）対応で強化。

- **【先行タスク】**: `src/app/**/route.ts` の **存在確認自体** を最優先タスクとする
- **確認対象**: 各Route Handler内での認証チェック、CORS設定
- **危険と判断する条件**: Route Handler が存在し、Server Action と同等の認証・認可がない
- **除外条件**: Route Handler が存在しない（Server Actions のみで完結）
- **該当ファイル**: `src/app/` 配下
- **優先度**: High（存在確認後に Medium 以下に調整）

##### 観点F.6 Next.js Cache の情報漏洩 [コード確認]

レビュー指摘（Claude 5-4, 5-6）対応で強化。

- **確認対象**:
  - `src/app/staff/**/*.tsx` での `fetch` 呼び出しの `cache` オプション
  - `page.tsx` の `export const dynamic = 'force-dynamic'` 宣言の有無
  - Server Action内での `revalidatePath` 呼び出し箇所
  - `next.config.ts` の `experimental.ppr` 設定確認
  - Next.js 16 での `fetch` デフォルトキャッシュ挙動（14以前は `force-cache`、15以降は `no-store`）
- **危険と判断する条件**:
  - `/staff/*` 配下で PPR が有効化
  - 認証ユーザ固有データが別ユーザにキャッシュ配信される
- **該当ファイル**: `src/app/staff/**/page.tsx`、各 `actions.ts`、`next.config.ts`
- **優先度**: High

##### 観点F.7 Prisma のコネクション管理 [コード確認]

- **確認対象**: `src/lib/prisma.ts` の `globalThis.prisma` パターン実装
- **該当ファイル**: `frontend/src/lib/prisma.ts`
- **優先度**: Medium

##### 観点F.8 iron-session の鍵ローテーション設計 [コード確認]

- **確認対象**: `src/lib/session.ts` の `password` 指定方法（文字列か配列か）
- **危険と判断する条件**: `password` が単一文字列で、鍵ローテーション時に全セッションが無効化される設計
- **除外条件**: 配列形式（`[{ id: 1, password: 'old' }, { id: 2, password: 'new' }]`）
- **該当ファイル**: `frontend/src/lib/session.ts`
- **優先度**: Medium

##### 観点F.9 React 19 の Hydration とデータリーク [コード確認]

- **確認対象**:
  - Server Component → Client Component のprops受け渡し箇所
  - Prisma クエリで `select` を明示しているか、モデル全体を返していないか
  - `src/lib/dto.ts` でのデータ変換層の有無
- **危険と判断する条件**: `passwordHash`, `twoFactorSecret`, `email` 等が不必要にクライアントに送られる
- **該当ファイル**: `src/lib/dto.ts`、各 `*Form.tsx`、`*Table.tsx`
- **優先度**: High

##### 観点F.10 React 19 の taint API 活用 [コード確認]

レビュー指摘（Claude 5-1）対応で追加。

- **確認対象**:
  - `src/lib/auth.ts` / `src/lib/session.ts` での `taintObjectReference` / `taintUniqueValue` 使用有無
  - Prisma モデルのうち `User.passwordHash`, `User.twoFactorSecret` 等を返す箇所での適用
- **危険と判断する条件**: 機密データに taint 未適用
- **優先度**: Medium

##### 観点F.11 Server Actions の `bind()` 誤用と encryption key [コード確認]

レビュー指摘（Claude 5-2）対応で追加。

- **確認対象**:
  - `.bind(null, ...)` パターンの `src/` 配下検索
  - `next.config.ts` の `experimental.serverActions.encryptionKey` 設定の有無
- **危険と判断する条件**:
  - `.bind(null, secretValue)` で機密値をバインド（クライアントに漏洩）
  - `encryptionKey` が未設定で、デプロイごとにServer Action のハッシュID が変動
- **該当ファイル**: `src/` 配下、`frontend/next.config.ts`
- **優先度**: High

##### 観点F.12 otplib のTOTP実装詳細 [コード確認]

レビュー指摘（ChatGPT F-3）対応で実害条件を追記。

- **確認対象**: `src/lib/auth.ts`、`src/app/staff/settings/actions.ts` のotplib設定
  - `secret` の長さ（160bit / 20byte / base32 32文字以上）
  - `algorithm`（HMAC-SHA1 が RFC6238 標準だが、SHA256/SHA512 も選択可）
  - `digits`（6桁が標準）
  - `step`（時間ステップ、30秒が標準）
  - `window`（許容ドリフト、`[前1, 後1]` = ±30秒が標準）
- **危険と判断する条件**:
  - secret 強度不足 → 総当たり耐性低下
  - window 過大 → リプレイ許容拡大
  - step 過大 → 有効時間過長
  - リカバリコード使用時の監査ログ記録がない
- **該当ファイル**: `frontend/src/lib/auth.ts`
- **優先度**: High

##### 観点F.13 bcryptjs のコストパラメータ [コード確認]

- **確認対象**: パスワードハッシュ化箇所（`bcrypt.hash(password, ??)`）の第二引数
- **危険と判断する条件**: saltRounds が10未満（2026年時点では12以上推奨）
- **該当ファイル**: `src/lib/auth.ts`、`src/app/staff/login/actions.ts`、`src/lib/recovery-codes.ts`、`frontend/prisma/seed.ts`
- **優先度**: Medium

##### 観点F.14 型安全性の誤用（TypeScript）と ランタイム型検証 [コード確認]

レビュー指摘（Claude 4-4）対応で強化。

- **確認対象**:
  - `src/` 配下を `as any`, `@ts-ignore`, `@ts-expect-error`, `as unknown as` で全文検索
  - `tsconfig.json` の `strict` 設定
  - `JSON.parse(request.body)` の戻り値に型アサーションのみで `User` 型等を付与していないか
  - `src/lib/dto.ts` が Zod 等で、全ての外部入力経路（Server Actions 引数、Route Handler、URL SearchParams）に適用されているか
- **危険と判断する条件**:
  - 型アサーションのみで外部入力を Prisma に流している
  - Zod バリデーションが一部の入力経路にしか適用されていない
- **除外条件**: Zod / Valibot 等でのランタイムバリデーションが全入力経路で徹底
- **該当ファイル**: `src/**/*.ts`、`src/**/*.tsx`、`frontend/tsconfig.json`、`src/lib/dto.ts`
- **優先度**: High

##### 観点F.15 Server Actions のペイロードサイズ制限 [コード確認]

- **確認対象**: `next.config.ts` の `experimental.serverActions.bodySizeLimit` 設定、ブログ画像の想定最大サイズ
- **該当ファイル**: `frontend/next.config.ts`、`src/app/staff/blog/BlogForm.tsx`
- **優先度**: Medium

##### 観点F.16 Tailwind CSS v4 の動的class生成 [コード確認]

- **確認対象**: `src/` 配下の `` className={`bg-${userInput}`} `` パターン検索
- **優先度**: Low

##### 観点F.17 sharp のリソース制限 [コード確認]

- **確認対象**: ブログ画像処理箇所の sharp 使用方法
- **危険と判断する条件**: `sharp.limitInputPixels()` 未設定、並列処理の抑制なし（image bomb DoS）
- **該当ファイル**: `src/app/staff/blog/actions.ts`
- **優先度**: Medium

##### 観点F.18 React 19 のuse hook とデータフェッチ [コード確認]

- **確認対象**: `src/` 配下の `use(` パターン、エラーバウンダリの実装
- **優先度**: Low

##### 観点F.19 Prototype Pollution [コード確認]

レビュー指摘（Claude 4-1）対応で新規追加。

- **確認対象**:
  - Contact フォーム・ブログ編集などで受け取った JSON を `Object.assign` / スプレッド演算子 / 手書き再帰マージで展開している箇所
  - `src/lib/dto.ts` での Zod バリデーションで `.strict()` / `.passthrough(false)` が使われているか
- **危険と判断する条件**:
  - `__proto__`, `constructor.prototype`, `prototype` をキーとして受け取り、そのまま展開
  - Prisma の `where` / `data` に直接ユーザ入力オブジェクトを流す実装（Mass Assignment と連鎖）
- **除外条件**: Zod strict スキーマで未知キーを拒否
- **該当コード候補**: `src/lib/dto.ts`、各 `actions.ts`
- **参照**: CWE-1321
- **優先度**: High

##### 観点F.20 ReDoS（Regular Expression Denial of Service） [コード確認]

レビュー指摘（Claude 4-2）対応で新規追加。

- **確認対象**: `src/lib/dto.ts`、各フォームのバリデーション正規表現
- **危険と判断する条件**:
  - `(a+)+b` 形式・`(a|a)+` 形式のネストした量指定子
  - メール正規表現の入れ子構造
- **除外条件**: Zod の組み込みバリデータ（`.email()` 等）への統一、または `safe-regex` パッケージによる静的解析
- **特に重要**: Contact フォーム（CL-05）は認証不要の外部POST可能エンドポイント
- **参照**: CWE-1333
- **優先度**: High

##### 観点F.21 Mass Assignment（ORM の property injection） [コード確認]

レビュー指摘（Claude 4-3）対応で新規追加。

- **確認対象**: 全 `actions.ts` の `prisma.*.create / update / upsert` 呼び出し箇所
- **危険と判断する条件**: `prisma.user.update({ data: formData })` のように、`data` に入力オブジェクトを直接流し、クライアント側から `role: 'admin'` 等の意図しないフィールドを注入される可能性
- **除外条件**: `data` への引数が明示的にフィールドを列挙（`{ name: input.name, email: input.email }`）、`...input` スプレッドで丸ごと渡していない
- **特に重要**:
  - `src/app/staff/users/actions.ts`（ロール昇格の危険）
  - `src/app/staff/settings/actions.ts`（他ユーザIDへのすり替え）
- **参照**: CWE-915
- **優先度**: Critical（実装次第）

##### 観点F.22 HTTP パラメータ汚染（HPP） [コード確認]

レビュー指摘（Claude 4-5）対応で新規追加。

- **確認対象**:
  - `src/app/blog/page.tsx`（カテゴリフィルタ `searchParams`）
  - `src/app/staff/audit-logs/page.tsx`（アクションフィルタ）
- **危険と判断する条件**: `searchParams.get()` と `searchParams.getAll()` の使い分けが不正確で、同一キーが複数回出現した際にバリデーション回避やDBクエリ異常
- **除外条件**: Zod スキーマでの型強制
- **優先度**: Medium

##### 観点F.23 Prisma 7 の新機能・変更点 [コード確認]

レビュー指摘（Claude 5-3）対応で追加。

- **確認対象**:
  - `@prisma/adapter-better-sqlite3` の接続管理（Prisma 6 以前との差異）
  - `prisma.config.ts`（新方式）と従来の `schema.prisma` の `datasource` ブロックの併用時の優先順位
  - Prisma 7 のログレベルのデフォルト変更
- **優先度**: Medium

---

#### F-b: バージョン/CVE照合が必要な事項

##### 観点F.1 Next.js 16 の既知脆弱性 [コード確認]

- **確認対象**:
  - Next.js GitHub Security Advisories
  - `pnpm audit`、`pnpm outdated`
  - `pnpm-lock.yaml` のバージョン確認
- **該当ファイル**: `frontend/package.json`、`frontend/pnpm-lock.yaml`
- **優先度**: High

##### 観点F.4 Next.js Image Optimization の脆弱性 [コード確認]

- **確認対象**: `next.config.ts` の `images.remotePatterns`, `images.domains` 設定、`next/image` の使用箇所
- **危険と判断する条件**: 外部URLに対し `remotePatterns` 制限が緩い（過去SSRF脆弱性）
- **該当ファイル**: `frontend/next.config.ts`、`src/app/**/*.tsx`
- **優先度**: Medium

##### 観点F.24 npm/pnpm postinstall スクリプトの検査 [コード確認]

レビュー指摘（Claude 9-1）対応で追加。

- **確認対象**:
  - `pnpm install --ignore-scripts` による postinstall スクリプトの実行抑制
  - `pnpm audit` / `pnpm audit --audit-level=high` の CI 組込み
  - `pnpm.onlyBuiltDependencies` 設定
- **優先度**: Medium

##### 観点F.25 SBOM（Software Bill of Materials）生成 [運用確認]

レビュー指摘（Claude 9-2）対応で追加。

- **確認対象**: 依存ライブラリのリスト（SPDX / CycloneDX 形式）生成、`syft` / `grype` の使用
- **優先度**: Low `Environment-dependent`

##### 観点F.26 Lockfile Poisoning とレビュー観点 [コード確認]

レビュー指摘（Claude 9-3）対応で追加。

- **確認対象**:
  - `pnpm install --frozen-lockfile` の CI での強制
  - lockfile 変更を含む PR でのレビュー要件
- **優先度**: Medium

##### 観点F.27 Subresource Integrity (SRI) [コード確認]

レビュー指摘（Claude 9-4）対応で追加。

- **確認対象**: 外部CDNから読み込むスクリプト・スタイルシートに対する `integrity` 属性、`next.config.ts` の `experimental.sri`
- **除外条件**: 外部リソースを読み込んでいない
- **優先度**: Low

---

### 1.G 運用・インフラ防御層（条件付き観点）

**ホストOSがLinux系（Ubuntu Server 等）である場合の条件付き観点**。本リポジトリからは本番ホストOSが確定できない。本章は `Environment-dependent` な項目で構成され、机上デバッグ（ステップ8のClaude Code調査）の対象外とし、運用担当者へのヒアリング候補として整理（レビュー指摘 B〜G-2, 12-4 対応）。

Ubuntu固有の確認コマンドやファイルは 4.B（本番環境構成の確認項目）に集約。ここでは観点の骨子のみ提示する。

---

#### 観点G.1 ホストOSの基本ハードニング [運用確認]

- **確認対象**: 最小限のパッケージインストール、`/etc/sysctl.conf` のカーネルパラメータ、swap 暗号化、`/tmp` の `noexec`
- **参照**: CIS Linux Benchmark、IPA-2.1
- **優先度**: High `Environment-dependent`

#### 観点G.2 ファイアウォール（UFW / iptables / nftables） [運用確認]

- **確認対象**: INPUT 最小許可ルール、Docker との共存（`DOCKER-USER` チェイン）
- **参照**: IPA-2.1
- **優先度**: High `Environment-dependent`

#### 観点G.3 侵入検知・自動ブロック（Fail2ban） [運用確認]

- **確認対象**: `sshd` jail、カスタム jail（`/staff/login` 失敗、Contact スパム検知）、アプリ層ロックアウトとの二重化
- **参照**: IPA-2.5、CWE-307
- **優先度**: High `Environment-dependent`

#### 観点G.4 SSH ハードニング [運用確認]

- **確認対象**: `PasswordAuthentication no`、`PermitRootLogin no`、`AllowUsers`、2FA
- **優先度**: High `Environment-dependent`

#### 観点G.5 自動セキュリティアップデート（unattended-upgrades） [運用確認]

- **確認対象**: `unattended-upgrades` 設定、カーネル Livepatch
- **参照**: IPA-2.1
- **優先度**: High `Environment-dependent`

#### 観点G.6 Mandatory Access Control（AppArmor / SELinux） [運用確認]

- **確認対象**: `aa-status`、Docker コンテナのプロファイル適用
- **優先度**: Medium `Environment-dependent`

#### 観点G.7 監査サブシステム（auditd） [運用確認]

- **優先度**: Medium `Environment-dependent`

#### 観点G.8 ファイル整合性監視（AIDE / Tripwire） [運用確認]

- **優先度**: Medium `Environment-dependent`

#### 観点G.9 セキュリティ診断ツール（Lynis / chkrootkit / rkhunter） [運用確認]

- **優先度**: Medium `Environment-dependent`

#### 観点G.10 ログ集約と異常検知（SIEM観点） [運用確認]

- **確認対象**: ELK / Loki / CloudWatch / Datadog 等、ホスト・コンテナ・アプリ3層のログ集約
- **優先度**: Medium `Environment-dependent`

#### 観点G.11 DDoS / DoS 対策の多層化 [運用確認]

- **確認対象**: L3/L4/L7 各層の対策
- **参照**: IPA-2.6
- **優先度**: High `Environment-dependent`

#### 観点G.12 Bot・スクレイパー対策 [コード確認] [運用確認]

- **確認対象**: `src/app/contact/actions.ts`、`src/app/staff/login/actions.ts`、`src/app/robots.ts`、CAPTCHA、honeypot
- **優先度**: Medium

#### 観点G.13 インシデント対応・フォレンジック準備 [運用確認]

- **確認対象**: IRP、連絡体制、証拠保全、シークレットローテーション手順、通知義務（72時間以内）
- **優先度**: Medium `Environment-dependent`

#### 観点G.14 時刻同期（NTP / chrony） [運用確認]

- **確認対象**: `systemd-timesyncd` または `chrony`
- **優先度**: Medium `Environment-dependent`

---

### 1.H 悪意ある第三者からの攻撃観点（新設）

レビュー指摘（Claude 10-1〜10-6）対応で、攻撃者視点の横断的観点を独立章として新設。

---

#### 観点H.1 Credential Stuffing（パスワードリスト攻撃） [コード確認]

- **確認対象**:
  - 漏洩パスワードDB（Have I Been Pwned API 等）とのチェック（ユーザ登録時・変更時）
  - IPベースのレート制限に加え、**同一IPから複数ユーザへのログイン試行** の検知
- **危険と判断する条件**: ユーザ別のアカウントロックのみで、IP別の横断的検知なし
- **該当コード候補**: `src/lib/rate-limit.ts`、`src/app/staff/login/actions.ts`
- **優先度**: Medium

#### 観点H.2 Account Takeover（ATO）連鎖 [コード確認]

- **確認対象**:
  - 管理者によるワンタイムパスワード発行（CL-25）が、管理者権限奪取時に全ユーザへの侵害経路になる可能性
  - リカバリコードの再生成機能があれば、2FAバイパス経路となる
  - セッション Hijack + 2FA スキップ：ログイン後のセッションを奪えば2FAを再要求しない設計か
- **危険と判断する条件**: リカバリコード再生成が2FAバイパス経路、セッション窃取で永続的なアクセス
- **優先度**: High

#### 観点H.3 内部脅威（Insider Threat） [コード確認] [仕様決定]

- **確認対象**:
  - 退職者のアカウント無効化タイミングと、セッションの強制無効化
  - 管理者権限の最小化（IPA-1.11 根本的解決 11-(ii) の運用面）
  - 監査ログの **管理者自身による改ざん** の検知
  - 権限変更（admin昇格等）の承認フロー
- **危険と判断する条件**: admin が無制限に権限操作可能、承認フローなし
- **優先度**: Medium

#### 観点H.4 Enumeration（列挙）攻撃 [コード確認]

- **確認対象**:
  - **ユーザ列挙**: ログインエラーメッセージ、パスワードリセット時のメッセージ、Contact フォームからのメール送信可否（重複送信時のメッセージ）
  - **URL列挙**: `/staff/*` 配下のパスが 401/403/404 で区別できる挙動による存在推定
  - **機能列挙**: ロール別にUI要素が変わる場合、HTMLコメントや未使用JSに admin専用情報が残存するか
- **危険と判断する条件**: エラーメッセージやHTTPステータスで存在有無が判別可能
- **優先度**: Medium

#### 観点H.5 CSP Bypass 手法への対策深化 [コード確認]

- **確認対象**: `next.config.ts` のCSP定義
- **危険と判断する条件**:
  - `'unsafe-inline'` を含む（実質CSP無効化）
  - `script-src` に許可したドメイン（CDNやGoogle系）が JSONP endpoint を提供
  - `nonce` ベースの場合、nonce値の予測困難性
  - Trusted Types（`require-trusted-types-for 'script'`）未導入
- **優先度**: High

#### 観点H.6 フィッシング攻撃（IPA-2.4 の能動的攻撃視点） [運用確認] [仕様決定]

- 観点 1.A.13 と連動。攻撃者視点では、フィッシングサイトとの識別困難性が能動的攻撃の代表格。
- **優先度**: Medium

#### 観点H.7 ショルダーハッキング / スクリーンショット漏洩 [コード確認]

レビュー指摘（Claude 11-1）対応で追加。

- **確認対象**: CL-24-② 初期パスワード画面、CL-25-② ワンタイムパスワード画面
- **危険と判断する条件**:
  - 画面の物理的表示期間が長く、自動非表示機能なし
  - CSSの `@media print` 非表示、`-webkit-user-select: none` 未実装
  - `Cache-Control: no-store, no-cache` 未設定、`autocomplete="off"` 属性なし
  - 管理者画面の「ユーザ作成完了」画面が別セッションでも同内容を表示（遷移直後のリロード耐性）
- **該当コード候補**: `src/app/staff/users/**`
- **優先度**: High

#### 観点H.8 タブナビング（window.opener 経由） [コード確認]

レビュー指摘（Claude 11-4）対応で追加。

- **確認対象**: CL-01（告知テキスト）、ブログ本文内の外部リンクに `rel="noopener noreferrer"` が付与されているか、`target="_blank"` の使用有無
- **危険と判断する条件**: `target="_blank"` 使用時に `rel` 属性欠落
- **除外条件**: 外部リンクに `rel="noopener noreferrer"` 付与
- **優先度**: Medium

---


## 2. 優先度型観点リスト（本リポジトリ固有リスク）

レビュー指摘（ChatGPT 2-1, 2-3）を踏まえ、本章を **3分割** して再編する。

- **2.A 確認済みの重大論点（`Confirmed`）**: 技術レポート・画面分析資料から既に問題の兆候が確認されたもの
- **2.B repoから強く疑われる論点（`Suspected`）**: 実装を読まないと確定しないが、設計上のリスクが強く疑われるもの
- **2.C 本番環境依存の重要確認（`Environment-dependent`）**: 本番運用時に確認が必要だが、repo 単体では判定できないもの

各項目に **攻撃の前提条件**（レビュー指摘 13-3 対応）を付記する。

---

### 2.A 確認済みの重大論点（`Confirmed`）

---

#### 2.A.1 【Critical】`Confirmed` `.env.local` のGitリポジトリ混入

- **証拠**: 技術レポートの「8. 環境変数の管理方法」で明記。`ENCRYPTION_KEY`、`SESSION_SECRET`、`SEED_ADMIN_PASSWORD` が平文で格納。
- **攻撃の前提**: Git閲覧権限保持者（低ハードル）
- **想定される影響**:
  - `SESSION_SECRET` 漏洩 → セッション偽造、任意ユーザへのなりすまし
  - `ENCRYPTION_KEY` 漏洩 → DBに保存されたTOTPシークレットの復号 → 2FA完全無効化
  - Git履歴に残っているため、現時点で `.env.local` を削除しても履歴から復元可能
- **確認方法**:
  1. `.gitignore` に `.env.local` が含まれているか
  2. `git log --all --full-history -- frontend/.env.local` でコミット履歴確認
  3. 過去のコミットでシークレットが露出した時点で、実質「漏洩済み」と扱い、全シークレットのローテーションを判断
- **該当コード**: `frontend/.env.local`、`frontend/.gitignore`、リポジトリ全体のGit履歴
- **参照**: IPA-2.5（パスワード管理）、CWE-798

---

#### 2.A.2 【Medium】`Confirmed` バックアップファイル・不要ファイルの混入

- **証拠**: リポジトリツリーに以下が確認されている
  - `src/app/favicon.ico.backup`
  - `src/app/globals.css.backup`
  - `src/app/page.tsx.backup`
  - `src/components/layouts/Footer.tsx.20260212`
  - `src/components/layouts/Header.tsx.20260206`
  - `public/images/background.png.20261302`（日付形式が異常）
- **攻撃の前提**: URLを推測した外部攻撃者（`public/` 配下は低ハードル）
- **想定される影響**:
  - バックアップファイル内に旧API キーやデバッグコードが残存するリスク
  - 公開ディレクトリ（`public/images/`）配下にあるため、URLを推測されれば外部からダウンロード可能
- **確認方法**: 各バックアップファイルの内容を確認、削除または `.gitignore` 化の判断。**`public/` 配下のバックアップは特に優先的に削除**（レビュー指摘 3-3 対応）

---

#### 2.A.3 【Medium】`Confirmed` `dev.db` のリポジトリ混入

- **証拠**: リポジトリツリーに `frontend/dev.db` が存在
- **攻撃の前提**: Git閲覧権限保持者
- **想定される影響**:
  - シードデータ以外に、開発中に入力したテストデータ（メールアドレス等）が履歴に残存
  - ユーザパスワード（bcryptハッシュ）が履歴に残れば、オフラインブルートフォース対象
- **確認方法**: `.gitignore` の `*.db` / `dev.db` の有無、`git log --all --full-history -- frontend/dev.db`
- **該当ファイル**: `frontend/dev.db`、`frontend/.gitignore`
- **関連**: 観点 E.2（SQLite ジャーナル・WAL ファイルの扱い）

---

#### 2.A.4 【Low】`Confirmed` public/uploads/blog/ の既存ファイルの素性

- **証拠**: `public/uploads/blog/1770992461005.png`, `1770994779680.png` の2件が存在（開発中のテストアップロード物と推測）
- **確認観点**: 個人情報や機密情報が含まれていないか、削除対象とすべきか

---

### 2.B repoから強く疑われる論点（`Suspected`）

---

#### 2.B.1 【Critical】`Suspected` Server Actions での認可チェックの徹底

- **疑い根拠**: Next.js App RouterのServer Actionsは、middleware.ts による認証だけでは不十分。Server Actionは独立したエンドポイントとして直接POSTされうるため、関数内部で毎回 `requireAuth()` / `requireAdmin()` を呼ぶ必要がある。
- **攻撃の前提**: 認証済み一般staff（中ハードル）
- **想定される影響**: 一般ユーザ（staff）が管理者専用操作（ユーザ作成、監査ログ閲覧、他ユーザのパスワード発行等）を実行可能になる可能性
- **確認方法**: 以下の全 `actions.ts` の先頭で、適切な認証・認可関数が呼ばれているか機械的に確認
  - `src/app/contact/actions.ts` → 認証不要（公開機能）
  - `src/app/staff/login/actions.ts` → 認証不要（ログイン自体）
  - `src/app/staff/blog/actions.ts` → staff以上
  - `src/app/staff/news/actions.ts` → staff以上
  - `src/app/staff/categories/actions.ts` → staff以上
  - `src/app/staff/contacts/actions.ts` → staff以上
  - `src/app/staff/settings/actions.ts` → staff（自分の設定のみ）
  - `src/app/staff/users/actions.ts` → **admin必須**
- **参照**: IPA-1.11、WKD-L、観点A.11.5

---

#### 2.B.2 【Critical】`Suspected` Mass Assignment によるロール昇格

レビュー指摘（Claude 4-3）対応で新規追加。

- **疑い根拠**: Prismaの `prisma.user.update({ data: formData })` や `prisma.blog.create({ data: input })` のように、ユーザ入力オブジェクトを直接 `data` に流す実装があると、クライアント側から `role: 'admin'` などの意図しないフィールドを注入される可能性
- **攻撃の前提**: 認証済み一般staff（中ハードル）
- **想定される影響**:
  - 一般staff が自己アカウントを admin に昇格
  - 他ユーザIDへのすり替えによる情報漏洩
- **確認方法**: 全 `actions.ts` の `prisma.*.create / update / upsert` 呼び出し箇所で、`data` への引数が明示的にフィールドを列挙しているか、`...input` スプレッドで丸ごと渡していないか
- **特に重要**:
  - `src/app/staff/users/actions.ts`（ロール昇格の危険）
  - `src/app/staff/settings/actions.ts`（他ユーザIDへのすり替え）
- **参照**: CWE-915、観点F.21

---

#### 2.B.3 【Critical】`Suspected` Next.js Middleware の認証カバレッジ

- **疑い根拠**: `src/middleware.ts` の `matcher` 設定の誤りで、認証が必要なパスが保護されない可能性。また Middleware は Edge Runtime で実行され、独自のAES-256-GCM実装が呼ばれると落ちる可能性。
- **攻撃の前提**: 未認証の外部攻撃者
- **確認方法**: `src/middleware.ts` の `matcher` 配列、`src/app/staff/` 配下のディレクトリ構造との突合、Edge Runtime 対応の確認
- **該当ファイル**: `frontend/src/middleware.ts`
- **参照**: 観点F.3

---

#### 2.B.4 【High】`Suspected` iron-session 設定の妥当性

- **疑い根拠**: iron-sessionの設定は全セッション制御の中核。`cookieOptions` の個別属性、パスワード長（32文字以上必須）、ローテーション設計が適切か。
- **攻撃の前提**: セッション窃取を試みる攻撃者（Confirmed の `.env.local` 混入により既に漏洩済みの可能性）
- **確認方法**:
  1. `src/lib/session.ts` で以下を確認：
     - `password` の長さと出所（環境変数か）
     - `cookieName: 'staff_session'` `Confirmed`
     - `cookieOptions.httpOnly: true` `Confirmed`
     - `cookieOptions.secure: process.env.NODE_ENV === 'production'` の判定
     - `cookieOptions.sameSite: 'lax'` `Confirmed`
     - `cookieOptions.maxAge: 60 * 60 * 8` (= 28800) `Confirmed`
  2. `password` を配列形式にしてローテーション可能設計になっているか
- **該当コード**: `src/lib/session.ts`

---

#### 2.B.5 【High】`Suspected` TOTP実装の詳細

- **疑い根拠**: otplib を使用し、TOTPシークレットは AES-256-GCM で暗号化保存。2FA関連の独自実装が多い。
- **攻撃の前提**: 漏洩したセッションまたはパスワードを持つ攻撃者、プロセス再起動時の窓を狙う攻撃者
- **確認観点**:
  - **2.B.5.1** TOTPシークレットの生成に `crypto.randomBytes` 相当の安全な乱数源を使用
  - **2.B.5.2** AES-256-GCM の IV（Nonce）を毎回ランダム生成
  - **2.B.5.3** AES-GCM の認証タグ（auth tag）を検証
  - **2.B.5.4** TOTP検証時のタイミング攻撃対策（定数時間比較）
  - **2.B.5.5** 60秒再利用防止のMapが、プロセス再起動で失われる影響
  - **2.B.5.6** リカバリコードの生成エントロピー、ハッシュ化、使用済みフラグの管理
  - **2.B.5.7** QRコード生成時、`otpauth://` URIにユーザ識別子がURLエンコードされて含まれるが、XSSやCRLFの混入リスク
- **該当コード**: `src/lib/auth.ts`, `src/lib/crypto.ts`, `src/lib/recovery-codes.ts`, `src/app/staff/login/actions.ts`, `src/app/staff/settings/actions.ts`

---

#### 2.B.6 【High】`Suspected` レート制限のインメモリ実装の限界

- **疑い根拠**: 技術レポートで「インメモリ（Mapベース）、単一プロセス構成でのみ有効」と明記 `Confirmed`。ただし実装詳細は `Suspected`。
- **攻撃の前提**: プロセス再起動を誘発できる攻撃者、または複数プロセスに分散してアクセスできる攻撃者
- **想定される影響**:
  - プロセス再起動でレート制限がリセット → ブルートフォース攻撃のウィンドウ拡大
  - 水平スケール時は各プロセスが独立したカウンタを持つため、実質的な制限回数が N倍
- **確認方法**:
  1. `src/lib/rate-limit.ts` のデータ構造と有効期限処理
  2. `docker-compose.prod.yml` の `deploy.replicas` やリスタートポリシー
  3. ロックアウトの状態をDBに永続化する設計になっているか（`User.loginAttempts`, `lockedUntil` を活用しているか）
- **該当コード**: `src/lib/rate-limit.ts`、`src/app/staff/login/actions.ts`
- **参照**: IPA-2.5

---

#### 2.B.7 【High】`Suspected` ブログ画像アップロードの検証

- **疑い根拠**: 本リポジトリで唯一のファイルアップロード機能（CL-16）。`public/uploads/blog/` 配下にタイムスタンプ風のファイル名で保存。
- **攻撃の前提**: 認証済み staff（アップロード権限を持つ）
- **確認観点**:
  - **2.B.7.1** 許可される拡張子・MIMEタイプ
  - **2.B.7.2** サーバ側での実ファイル内容検証（マジックナンバーチェック）
  - **2.B.7.3** ファイルサイズ上限（DoS対策）
  - **2.B.7.4** ファイル名生成ロジック（ユーザ入力のファイル名を使っていないか、パスの混入がないか）
  - **2.B.7.5** 保存先パスのトラバーサル防止（`path.join` + `path.resolve` で境界外チェック）
  - **2.B.7.6** sharpでの画像リサイズ処理時の例外ハンドリング（画像爆弾対策）
  - **2.B.7.7** アップロードされた画像がHTMLとして解釈されるリスク（`.html` が拡張子偽装で混入しないか）
  - **2.B.7.8** 静的配信パスのキャッシュヘッダ、Content-Typeの明示
- **該当コード**: `src/app/staff/blog/BlogForm.tsx`、`src/app/staff/blog/actions.ts`
- **参照**: IPA-1.3、CWE-434

---

#### 2.B.8 【High】`Suspected` ログイン時セッション再生成（Session Fixation）

- **疑い根拠**: リポジトリに `scripts/test-session-fixation.sh` が既に存在することから、開発側も重要視している `Confirmed`。ただしテスト結果・実装詳細は `Suspected`。
- **攻撃の前提**: 被害者に特定のセッションIDを注入できる攻撃者
- **確認方法**:
  1. `src/app/staff/login/actions.ts` でログイン成功時、`session.destroy()` → 新しいセッション値の設定 → `session.save()` の順序
  2. `scripts/test-session-fixation.sh` のテスト内容と結果
- **該当コード**: `src/app/staff/login/actions.ts`、`scripts/test-session-fixation.sh`

---

#### 2.B.9 【High】`Suspected` CSRF保護の実効性

- **疑い根拠**: Next.js Server ActionsのCSRF保護は、Next.js フレームワーク側の Origin ヘッダ検証と `sameSite=lax` Cookie に依存。`scripts/test-csrf.sh` が存在する `Confirmed` ため既にテスト実施済みと推測されるが、保護の具体的メカニズムは `Suspected`。
- **攻撃の前提**: 認証済みユーザを罠ページに誘導できる攻撃者
- **確認方法（WKD-C 4条件対応）**:
  1. `next.config.ts` の `experimental.serverActions.allowedOrigins` 設定
  2. Server Actions のリクエストヘッダ（Next-Action ヘッダ等）の検証ロジック
  3. `scripts/test-csrf.sh` のテストケース
  4. iron-session の `sameSite=lax` による保護範囲と、全ての状態変更操作がPOST経由か
- **該当コード**: `next.config.ts`、全 `actions.ts`、`scripts/test-csrf.sh`
- **参照**: IPA-1.6、観点A.6.1

---

#### 2.B.10 【High】`Suspected` Contact画面（公開フォーム）の入力検証

- **疑い根拠**: 認証不要で外部からPOST可能な唯一の入力フォーム（CL-05）。攻撃者にとって最もアクセスしやすいエンドポイント。
- **攻撃の前提**: 未認証の外部攻撃者（ハードル最低）
- **確認観点**:
  - **2.B.10.1** スキーマバリデーション（Zod等）の有無と厳格さ
  - **2.B.10.2** XSS：本文にHTMLタグを含む入力が管理画面（CL-10）でエスケープ
  - **2.B.10.3** SQLインジェクション（Rawクエリ使用有無）
  - **2.B.10.4** メールヘッダインジェクション（メール送信実装がある場合）
  - **2.B.10.5** レート制限（スパム対策）
  - **2.B.10.6** CAPTCHA/honeypotの有無
  - **2.B.10.7** ログへの全文記録によるログインジェクションリスク
  - **2.B.10.8** ReDoS（観点F.20 と連動）
  - **2.B.10.9** Prototype Pollution（観点F.19 と連動）
- **該当コード**: `src/app/contact/page.tsx`、`src/app/contact/actions.ts`、`src/lib/dto.ts`

---

#### 2.B.11 【High】`Suspected` CSPヘッダの内容

- **疑い根拠**: 技術レポートで「公開ページ用 / staff用 で分離」との記載あり `Confirmed`。Next.js 16のApp Router + Server Actionsは `'unsafe-inline'` や `'unsafe-eval'` を要求するケースがあるため、現実的な厳格度は `Suspected`。
- **確認方法**: `next.config.ts` のCSP定義
  - `script-src`: `'self'` か `'unsafe-inline'` 許容か、nonce ベースか
  - `style-src`: Tailwind CSSとの整合
  - `img-src`: ブログ画像（同一オリジン）のみか外部許可か
  - `connect-src`: Server Actions のPOST先
  - `frame-ancestors`: `'none'` か `'self'` か
  - `form-action`: フォーム送信先の制限
  - `upgrade-insecure-requests`: 本番環境での有効化
  - `require-trusted-types-for 'script'` の導入有無
- **該当コード**: `next.config.ts`
- **参照**: 観点A.5.7、H.5

---

#### 2.B.12 【High】`Suspected` 監査ログの完全性

- **疑い根拠**: 技術レポートで `AuditLog` モデルに「ユーザーID・アクション・リソース・IPアドレス・User-Agent」を記録とあり `Confirmed`。ただし記録漏れやログ改ざん耐性は `Suspected`。
- **確認観点**:
  - **2.B.12.1** 記録対象イベントの網羅
  - **2.B.12.2** ログ改ざん防止（admin権限でもログ削除できないか）
  - **2.B.12.3** IPアドレス取得元（`X-Forwarded-For` ヘッダの信頼性、観点E.9 と連動）
  - **2.B.12.4** User-Agent の長さ制限（ログ肥大化攻撃対策）
  - **2.B.12.5** ログ書き込み失敗時の動作
  - **2.B.12.6** 監査ログ自体のアクセス履歴の記録
  - **2.B.12.7** 監査ログの削除・エクスポート機能の有無（WORM原則、レビュー指摘 11-6 対応）
- **該当コード**: `src/lib/audit-log.ts`、各 `actions.ts` でのログ書き込み箇所、`src/app/staff/audit-logs/*`

---

#### 2.B.13 【High】`Suspected` Server/Client 境界でのシークレット漏洩

- **疑い根拠**: Next.js App Router + Server Actions + React 19 の組み合わせで、`'use client'` 宣言したモジュールからセッション情報・環境変数を参照するとクライアントバンドルに混入する。
- **攻撃の前提**: 未認証の外部攻撃者（公開バンドルの検査）
- **確認方法**:
  1. `src/lib/session.ts`, `src/lib/crypto.ts`, `src/lib/auth.ts` 等に `import 'server-only'` が宣言されているか
  2. Client Component から直接これらのモジュールを import している箇所がないか
  3. ビルド後の `.next/static/chunks/` で機密情報の文字列が含まれていないか（grep検査）
- **該当コード**: `src/lib/*.ts`、`src/app/**/*.tsx`（`'use client'` 宣言あり）

---

#### 2.B.14 【High】`Suspected` 機密カラムのDB保存形式

- **疑い根拠**: 技術レポートでいくつかの機密フィールドが列挙されているが、各々の保存形式（平文/ハッシュ/暗号化）の明確化が必要
- **特に重要**: `users.oneTimePassword`（発行後短時間で使用される想定だが、DB保存時に暗号化されているか、平文か）
- **危険と判断する条件**: `oneTimePassword` が平文保存 → 発見時 Critical に昇格
- **確認方法**: `frontend/prisma/schema.prisma` の型定義、`src/lib/crypto.ts` / `src/lib/auth.ts` / `src/app/staff/users/actions.ts` での保存前の処理
- **該当ファイル**: 上記3ファイル
- **参照**: IPA-2.5、観点E.5

---

#### 2.B.15 【High】`Suspected` ショルダーハッキング / 初期PW・OTP画面表示

レビュー指摘（Claude 11-1）対応で追加。

- **疑い根拠**: 画面分析資料より、初期パスワード（CL-24-②）とワンタイムパスワード（CL-25-②）が画面上に表示される設計 `Confirmed`。ただし保護機構は `Suspected`。
- **攻撃の前提**: 画面を覗き見できる内部者、印刷・スクリーンショット履歴を入手できる攻撃者
- **確認観点**:
  - 画面の物理的表示期間、自動非表示機能
  - CSSの `@media print` 非表示、`-webkit-user-select: none`
  - `Cache-Control: no-store, no-cache`、`autocomplete="off"` 属性
  - 管理者画面の「ユーザ作成完了」画面が別セッションでも同内容を表示（リロード耐性）
- **該当コード**: `src/app/staff/users/**`
- **参照**: 観点H.7

---

#### 2.B.16 【Medium】`Suspected` 2FA 無効化フローの保護

レビュー指摘（Claude 11-5）対応で追加。

- **疑い根拠**: 画面分析資料 CL-29 に「有効化または無効化ボタン」とあり `Confirmed`。2FA無効化が1ボタンで可能な設計の可能性。
- **攻撃の前提**: セッション窃取済み攻撃者
- **確認方法**:
  - 無効化フローに再認証ステップ（パスワード + 現在の2FAコード）があるか
  - 無効化実行時の監査ログ記録と、対象ユーザへのメール通知
- **該当コード**: `src/app/staff/settings/actions.ts`
- **参照**: 観点A.9.2

---

#### 2.B.17 【Medium】`Suspected` 一括削除機能のCSRF／認可／DoS

レビュー指摘（Claude 11-2）対応で追加。

- **疑い根拠**: 画面分析資料 CL-10 に「選択チェックボックス」による一括削除機能が示唆 `Suspected`
- **攻撃の前提**: 認証済み staff
- **確認方法**:
  - 一括操作時のトランザクション性
  - CSRF トークン検証（1操作1トークンか、リクエスト単位か）
  - 削除対象ID配列の上限値
- **該当コード**: `src/app/staff/contacts/actions.ts`
- **参照**: 観点A.6.4

---

#### 2.B.18 【Medium】`Suspected` CL-17a カテゴリチップの整合性

レビュー指摘（Claude 11-3）対応で追加。

- **疑い根拠**: CL-17a で既選択カテゴリがチップとして表示され削除可能 `Confirmed`。サーバ側での整合性チェックは `Suspected`。
- **確認方法**: 削除されたチップのカテゴリが本当にユーザのアクセス可能範囲か（他ユーザの記事のカテゴリを操作できないか）
- **該当コード**: `src/app/staff/blog/actions.ts`

---

#### 2.B.19 【Medium】`Suspected` 外部リンクの `rel` 属性

レビュー指摘（Claude 11-4）対応で追加。

- **疑い根拠**: CL-01 の「告知テキスト（他ページへのリンクを含む）」について、リンクの種類（内部/外部）と `target="_blank"` の使用有無 `Suspected`
- **確認方法**:
  - 外部リンクに `rel="noopener noreferrer"` が付与されているか
  - ブログ本文内のリンクが同様の扱いになっているか
- **参照**: 観点H.8

---

#### 2.B.20 【Medium】`Suspected` 依存関係の脆弱性（SCA観点）

- **疑い根拠**: 主要な依存ライブラリのバージョンが最新に近いが、個別に既知CVEがないか確認が必要
- **確認方法**: `pnpm audit`、Trivyスキャン、GitHub Dependabot Alerts
- **該当ファイル**: `frontend/package.json`, `frontend/pnpm-lock.yaml`
- **参照**: 観点F.1

---

#### 2.B.21 【Medium】`Suspected` Next.js キャッシュによるクロスユーザ情報漏洩

- **疑い根拠**: Next.js 16 の App Router ではデフォルトで静的レンダリング/キャッシュが有効化される場合がある。`/staff/*` 配下の管理画面で、認証ユーザ固有データが別ユーザにキャッシュ配信されるリスク `Suspected`
- **危険と判断する条件（発見時 Critical に昇格）**: ユーザ情報が他者に表示される実装が見つかった場合
- **確認方法**:
  1. `src/app/staff/**/page.tsx` で `export const dynamic = 'force-dynamic'` が宣言されているか
  2. Server Component内のPrismaクエリがユーザごとに異なる結果を返す場合、キャッシュ無効化が正しく設定されているか
- **該当ファイル**: `src/app/staff/**/page.tsx` 全般
- **参照**: 観点F.6

---

### 2.C 本番環境依存の重要確認（`Environment-dependent`）

本節は運用担当者へのヒアリング対象（4.B / 4.C と連携）。

---

#### 2.C.1 【High】`Environment-dependent` 本番環境の secrets 注入方法

- **確認対象**:
  - `docker/docker-compose.prod.yml` の環境変数関連設定
  - 本番運用ドキュメント（`docs/architecture/`, `docs/process/`）
  - Docker secrets / Kubernetes Secrets / 外部 Vault 等の採用有無
- **該当ファイル**: `docker/docker-compose.prod.yml`、`docs/`
- **参照**: 観点B.7

---

#### 2.C.2 【High】`Environment-dependent` Docker本番構成のハードニング

- **確認観点**:
  - コンテナ内から不要なシステムコール・権限を制限（`read_only`, `cap_drop` 等）
  - リバースプロキシ（nginx等）の配置方針と、TLS終端、HTTPヘッダの最終設定者
  - ベースイメージのCVE
  - `.dockerignore` の内容
  - `docker-compose.prod.yml` のネットワーク分離、ボリューム設計
- **該当ファイル**: `frontend/docker/Dockerfile.prod`、`docker/docker-compose.prod.yml`、`frontend/.dockerignore`
- **参照**: 観点B.1〜B.12

---

#### 2.C.3 【High】`Environment-dependent` リバースプロキシ・TLS・ネットワーク層の欠落

- **確認対象**:
  - リバースプロキシの配置（nginx、ALB、CloudFront 等）
  - TLS終端の配置
  - CDN / WAF の採用方針
  - HSTS preload 登録の予定
- **該当ファイル**: （リポジトリ外）
- **参照**: IPA-2.3、IPA-2.6、観点D.1〜D.14

---

#### 2.C.4 【High】`Environment-dependent` DDoS / DoS 攻撃の多層防御

| 層 | 現状 | 補足 |
|----|------|------|
| L7 アプリケーション（アカウントロック） | 実装あり（`User.isLocked`）`Confirmed` | ユーザ別のロックのみで別アカウント試行には無力 |
| L7 アプリケーション（Rate Limit） | インメモリ実装（2.B.6）`Confirmed` | プロセス再起動で消失 |
| L7 Contact フォームのスパム対策 | 未確認 `Suspected` | CAPTCHA の有無 |
| L7 リバースプロキシ | 未実装 `Confirmed` | nginx `limit_req` 等の標準対策なし |
| L7 CDN/WAF | 未確認 `Environment-dependent` | 大規模DDoS時に防御層なし |
| L4 SYN flood対策 | ホストOS依存 `Environment-dependent` | `tcp_syncookies` 等 |
| L3 IPレベルバン | 未実装 `Environment-dependent` | Fail2ban 未導入（1.G.3） |
| L3 クラウドDDoS | 未確認 `Environment-dependent` | AWS Shield/GCP Cloud Armor 等 |

- **参照**: IPA-2.6、観点G.11

---

#### 2.C.5 【High】`Environment-dependent` 本番ホスト環境・運用構成の未定義

- **確認対象**: 本番ホストOS、DB、リバースプロキシ、secrets、ログ集約、監視、バックアップ、CI/CD
- **該当ファイル**: `docs/architecture/`、`docs/process/`

---

#### 2.C.6 【High】`Environment-dependent` Fail2ban 等ホストレベル侵入検知の未導入

- **確認対象**: 本番運用ドキュメントにFail2banの導入手順があるか、アプリ層のログを Fail2ban で解析する連携設計
- **該当ファイル**: （リポジトリ外）
- **参照**: IPA-2.5、IPA-2.6、観点G.3

---

#### 2.C.7 【High】`Environment-dependent` ホストOS ハードニング方針の未定義

- **確認対象**: UFW/iptables、SSH、unattended-upgrades、AppArmor、auditd、NTP/chrony、ユーザ管理
- **該当ファイル**: （リポジトリ外）
- **参照**: CIS Linux Benchmark、IPA-2.1、観点G.1

---

#### 2.C.8 【Medium】`Environment-dependent` データベースの永続化とバックアップ

- **確認観点**:
  - 本番構成でのDBファイル配置（コンテナ内か永続ボリュームか）
  - DBファイルのパーミッション
  - バックアップ設計（インシデント時の復元可否）
  - PostgreSQL移行時の接続文字列、パスワード管理
- **該当ファイル**: `prisma.config.ts`、`docker/docker-compose.prod.yml`、`frontend/.env.local.example`

---

## 3. コード型：確認必須ファイルリスト（repo内限定）

レビュー指摘（ChatGPT 3-1, 3-2, 3-3）を踏まえ、本章は **repo内限定** とする。repo外の運用・インフラ設定は4章（ヒアリング候補）へ移動。

各ファイルに **起点脅威**（レビュー指摘 3-2 対応）を1つ付記する。

確認目的は以下のいずれか：
- **(S)** 機密情報のハードコーディング検査
- **(V)** 脆弱な関数・APIの使用検査
- **(A)** 認証・認可・セッション実装の確認
- **(I)** 入力検証・出力エスケープの確認
- **(C)** セキュリティ設定・ポリシーの確認
- **(F)** フレームワーク固有の挙動・設定
- **(N)** ネットワーク・インフラ層の設定

---

### 3.0 公開配信領域の不要ファイル（最優先・新設）

レビュー指摘（ChatGPT 3-3）対応で最上段に配置。`public/` 配下の残骸は外部攻撃者からURL推測でアクセス可能なため最優先。

| 優先度 | パス | 確認目的 | 起点脅威 | 主な確認点 |
|--------|------|----------|----------|------------|
| High | `frontend/public/images/background.png.20261302` | S | 情報漏洩 | 公開配信中、日付形式異常、不要性、削除判断 |
| Low | `frontend/public/uploads/blog/1770992461005.png` | S | 情報漏洩 | テストアップロードファイル、機密性 |
| Low | `frontend/public/uploads/blog/1770994779680.png` | S | 情報漏洩 | 同上 |

---

### 3.1 設定・環境ファイル（最優先）

| 優先度 | パス | 確認目的 | 起点脅威 | 主な確認点 |
|--------|------|----------|----------|------------|
| Critical | `frontend/.env.local` | S | 機密情報Git混入 | シークレットの格納有無、Git混入の現状 |
| Critical | `frontend/.gitignore` | S, C | 機密情報Git混入 | `.env*`、`*.db`、`.backup` 等の除外設定 |
| Critical | `frontend/.env.local.example` | S, C | 機密情報Git混入 | プレースホルダが適切か、実値が混入していないか |
| High | `frontend/next.config.ts` | C, F, N | CSP/セキュリティヘッダ | CSP内容（公開/staff分離）、`experimental.serverActions.allowedOrigins`、`bodySizeLimit`、`encryptionKey`、`images.remotePatterns`、`experimental.ppr`、`poweredByHeader: false` |
| High | `frontend/prisma.config.ts` | S, C | DB接続機密 | DB接続文字列、環境変数参照、SQLite/PostgreSQL 切替 |
| High | `docker/docker-compose.prod.yml` | S, C, N | secrets管理/ネットワーク分離 | 環境変数注入方法、ネットワーク、ボリューム、restart policy、cap_drop、security_opt、ports/expose |
| High | `docker/docker-compose.dev.yml` | S, C | 開発用設定漏洩 | 開発用環境変数の扱い、ボリュームマウント |
| High | `frontend/docker/Dockerfile.prod` | C, N | コンテナハードニング | ベースイメージ固定(digest)、非rootユーザ、最小権限、multi-stage最適化、NODE_OPTIONS、init |
| High | `frontend/.dockerignore` | S, C | 機密情報イメージ混入 | `.env*`、`node_modules`、`.git`、`*.backup`、`dev.db*`（WAL含む）の除外 |
| Medium | `frontend/docker/Dockerfile.dev` | C | 開発用設定 | 開発用設定、ビルドツールの残存 |
| Medium | `frontend/package.json` | C, V | 依存管理/script | 依存バージョン、scriptsの危険なコマンド、`pnpm.onlyBuiltDependencies` |
| Medium | `frontend/pnpm-lock.yaml` | V | 依存CVE/Lockfile Poisoning | ロックされた依存バージョンの既知CVE、予期しない変更の検出 |
| Low | `frontend/eslint.config.mjs` | C | 静的解析ルール | セキュリティ関連ルール（`no-eval` 等）の有効化 |
| Low | `frontend/tsconfig.json` | C | 型安全性 | `strict` モード、型チェックの厳格度 |

---

### 3.2 Prisma（データ層）

| 優先度 | パス | 確認目的 | 起点脅威 | 主な確認点 |
|--------|------|----------|----------|------------|
| High | `frontend/prisma/schema.prisma` | C | データモデル設計 | 機密フィールドの型、`oneTimePassword` 等の保存形式、`authorId` 相当の有無（IDOR判定用） |
| High | `frontend/prisma/seed.ts` | S, V | 初期パスワードハードコーディング | 初期パスワードのハードコーディング、テストユーザの扱い、bcrypt saltRounds |
| Medium | `frontend/prisma/migrations/20260214094728_add_security_features/migration.sql` | C | セキュリティ機能DDL | セキュリティ機能追加時のカラム定義、制約 |
| Medium | `frontend/prisma/migrations/20260214010000_add_admin_auth/migration.sql` | C | 認証基盤DDL | 認証基盤導入時のDDL |
| Medium | `frontend/prisma/migrations/20260214020000_replace_admin_with_user/migration.sql` | C | テーブル構造変更 | 旧adminテーブルの残存データ懸念、Down migration欠落対応 |
| Low | その他マイグレーション（6件） | C | スキーマ変遷 | スキーマ変遷の把握 |

---

### 3.3 認証・セキュリティライブラリ層（src/lib/）

| 優先度 | パス | 確認目的 | 起点脅威 | 主な確認点 |
|--------|------|----------|----------|------------|
| Critical | `frontend/src/lib/auth.ts` | A, F | admin認可欠落 | `requireAuth()`、`requireAdmin()` の実装、セッション取得、ロール判定、`import 'server-only'` 宣言、otplib設定 |
| Critical | `frontend/src/lib/session.ts` | A, C, F | セッション固定化/Cookie属性 | iron-session設定（cookieOptions全属性、password長、rotation）、`import 'server-only'` 宣言 |
| Critical | `frontend/src/lib/crypto.ts` | V, S, F | 暗号化実装不備 | AES-256-GCMの実装（IV生成、auth tag検証、鍵管理）、`import 'server-only'` 宣言、Edge Runtime 非対応性 |
| High | `frontend/src/lib/rate-limit.ts` | A, V | ブルートフォース耐性 | レート制限のデータ構造、有効期限、競合状態、プロセス再起動時の挙動 |
| High | `frontend/src/lib/recovery-codes.ts` | A, V | 2FAバイパス | リカバリコード生成（エントロピー源、ハッシュ化）、検証（定数時間比較） |
| High | `frontend/src/lib/audit-log.ts` | A, V, N | ログ改ざん/IP信頼性 | ログ記録API、IP取得（`X-Forwarded-For` の信頼性）、書き込み失敗時の動作、User-Agent長さ制限 |
| High | `frontend/src/lib/dto.ts` | I | バリデーション欠落 | 入力バリデーションスキーマ（Zod等）の厳格度、`.strict()` 使用、全フォーム対応、ReDoS 対策、ランタイム型検証 |
| High | `frontend/src/lib/prisma.ts` | V, F | SQLi/ログ漏洩 | Prismaクライアント初期化、`$queryRaw` 使用の有無、`log` オプション、`globalThis` パターン |
| Critical | `frontend/src/middleware.ts` | A, F | 認証カバレッジ | `matcher` 設定、認証チェック漏れのパス、Edge Runtime 対応、cryptoモジュール呼び出しの有無 |

---

### 3.4 Server Actions（状態変更操作の要）

| 優先度 | パス | 確認目的 | 起点脅威 | 主な確認点 |
|--------|------|----------|----------|------------|
| Critical | `frontend/src/app/staff/users/actions.ts` | A, I, V | 権限昇格/Mass Assignment | `requireAdmin()` 呼び出し、自分自身の削除/降格防止、パスワード生成、入力検証、Mass Assignment 対策（`data` 引数の明示列挙） |
| Critical | `frontend/src/app/staff/login/actions.ts` | A, I, V | セッション固定化/ユーザ列挙 | ログイン処理、セッション再生成、レート制限連携、エラーメッセージ統一、ユーザ列挙対策、2FA検証 |
| Critical | `frontend/src/app/staff/settings/actions.ts` | A, I, V | 2FA無効化/他ユーザすり替え | 自分のパスワード変更、2FA有効/無効化、現在パスワード確認、再認証、Mass Assignment（他ユーザIDへのすり替え防止） |
| High | `frontend/src/app/staff/blog/actions.ts` | A, I, V | アップロード/IDOR | 画像アップロード、CRUD、IDOR検証、XSS、ファイル名サニゼーション、sharpリソース制限、Prototype Pollution |
| High | `frontend/src/app/staff/news/actions.ts` | A, I, V | XSS/CRUD | CRUD、XSS、公開日時のバリデーション |
| High | `frontend/src/app/staff/categories/actions.ts` | A, I, V | CRUD | CRUD、削除時の関連ブログ処理 |
| High | `frontend/src/app/staff/contacts/actions.ts` | A, V | 一括削除CSRF/DoS | 削除処理、一括削除のトランザクション/上限、取得時の認可 |
| High | `frontend/src/app/contact/actions.ts` | I, V, N | 公開入力面（XSS/SQLi/スパム/ReDoS） | **公開エンドポイント**、入力検証、メール送信時のヘッダインジェクション、レート制限、CAPTCHA、ReDoS |

---

### 3.5 ページ・フォームコンポーネント

| 優先度 | パス | 確認目的 | 起点脅威 | 主な確認点 |
|--------|------|----------|----------|------------|
| High | `frontend/src/app/staff/login/LoginForm.tsx` | I | クライアント入力 | パスワード入力type、autocomplete属性、エラー表示 |
| High | `frontend/src/app/staff/login/page.tsx` | A, F | 認証済時挙動 | 認証状態時のリダイレクト、URLパラメータ処理、dynamic設定 |
| High | `frontend/src/app/staff/blog/BlogForm.tsx` | I, F | ファイル入力/クライアント境界 | ファイル入力検証、クライアント側XSS、フォームaction、`'use client'` 宣言時のシークレット参照有無 |
| High | `frontend/src/app/staff/settings/PasswordChangeForm.tsx` | I | パスワード検証 | パスワード確認、強度表示、現在PW必須 |
| High | `frontend/src/app/staff/settings/TwoFactorSettings.tsx` | I, A | 2FA無効化/再認証 | QRコード表示、認証コード検証フロー、無効化時の再認証 |
| High | `frontend/src/app/staff/users/new/CreateUserForm.tsx` | I | ロール選択 | ロール選択、入力検証、Mass Assignment対策 |
| High | `frontend/src/app/staff/audit-logs/AuditLogsTable.tsx` | I, V | フィルタ入力XSS | フィルタ入力、XSS、ログ表示時のエスケープ、HPP対策 |
| High | `frontend/src/components/ui/PasswordInput.tsx` | I | パスワード表示切替 | パスワード表示切替、autocomplete属性 |
| Medium | `frontend/src/app/contact/page.tsx` | I | 公開フォーム | フォーム構造、クライアント側検証 |
| Medium | `frontend/src/app/blog/[id]/page.tsx` | V, F | ブログXSS/キャッシュ | ブログ本文レンダリング（XSS）、`dangerouslySetInnerHTML` 確認、キャッシュ設定、外部リンク `rel` |
| Medium | `frontend/src/app/news/[id]/page.tsx` | V, F | ニュースXSS | ニュース本文レンダリング（XSS）、キャッシュ設定 |
| Medium | `frontend/src/app/blog/page.tsx` | V, F | カテゴリフィルタHPP | カテゴリフィルタ、一覧レンダリング、キャッシュ、HPP対策 |
| Medium | `frontend/src/app/blog/MobileFilterMenu.tsx` | V | モバイルフィルタ | モバイル用フィルタUI、入力値の扱い |
| Medium | `frontend/src/app/staff/StaffShell.tsx` | A | サイドナビ制御 | サイドナビの表示制御、ロールによる分岐 |
| Medium | `frontend/src/app/staff/layout.tsx` | A, F | staffレイアウト | staff配下レイアウト、セッション取得、dynamic設定 |
| Medium | `frontend/src/app/staff/page.tsx` | A, F | ダッシュボード | ダッシュボード、ロール別表示、キャッシュ |
| Medium | `frontend/src/app/staff/users/page.tsx` | A, F | ユーザ一覧（admin） | admin認可、一括表示時のフィールド絞り込み |
| Medium | `frontend/src/app/staff/users/new/page.tsx` | A, F | ユーザ作成（admin） | admin認可、初期PW生成フロー、CL-24系画面 |
| Medium | 全 `*Table.tsx` | I, V | 一覧表示XSS/削除CSRF | 一覧表示時のエスケープ、削除操作のCSRF |
| Low | `frontend/src/app/layout.tsx` | C | グローバルレイアウト | グローバルレイアウト、meta設定 |
| Low | `frontend/src/app/page.tsx` | V | トップページ | トップページコンテンツ、告知テキスト外部リンク（CL-01） |
| Low | `frontend/src/app/robots.ts` | C | プレースホルダ残存 | `example.com` プレースホルダの残存確認 |
| Low | `frontend/src/app/sitemap.ts` | C | プレースホルダ残存 | 同上 |
| Low | `frontend/src/app/about/page.tsx` | C | 運営者情報明示 | 会社名・所在地・連絡先（IPA-2.4 フィッシング対策、CL-02） |

---

### 3.6 スクリプト類

| 優先度 | パス | 確認目的 | 起点脅威 | 主な確認点 |
|--------|------|----------|----------|------------|
| High | `frontend/scripts/reset-password.ts` | S, A | パスワードリセット | パスワードリセット処理、引数取扱い、実行権限 |
| High | `frontend/scripts/list-users.ts` | S | ユーザ一覧出力 | 出力内容に機密情報が含まれるか |
| High | `scripts/test-csrf.sh` | C | CSRFテスト内容 | CSRFテストのシナリオと期待結果 |
| High | `scripts/test-session-fixation.sh` | C | Session Fixationテスト内容 | セッション固定テストのシナリオと期待結果 |

---

### 3.7 既存のセキュリティレポート

| 優先度 | パス | 確認目的 | 起点脅威 | 主な確認点 |
|--------|------|----------|----------|------------|
| High | `security-report/security-report.md` | C | 既存レポート重複回避 | 既存の脆弱性レポートの内容、今回の診断範囲との重複 |
| High | `security-report/security-review-response.md` | C | 対応状況把握 | レビュー指摘と対応状況 |
| High | `security-report/security-test-results.md` | C | テスト実施状況 | テスト実施済み項目と結果 |

---

### 3.8 ドキュメント類

| 優先度 | パス | 確認目的 | 起点脅威 | 主な確認点 |
|--------|------|----------|----------|------------|
| Medium | `docs/architecture/` 配下 | C, N | 本番構成未定義 | 本番構成図、リバースプロキシ・TLS・secrets管理の方針 |
| Medium | `docs/process/` 配下 | C | 運用手順未定義 | 運用手順、secretsローテーション手順、デプロイ手順、インシデント対応手順 |
| Medium | `docs/failures/` 配下 | C | 過去事例 | 過去のインシデント・失敗事例の記録 |
| Low | `docs/ai-sessions/` 配下 | S | AI対話ログ | AIとの対話ログに機密情報が残っていないか |

---

### 3.9 バックアップ・不要ファイル（削除候補）

3.0 の `public/` 配下は最優先で別途掲載済み。`src/` 配下のバックアップは外部公開されないが、コード混乱・情報漏洩リスクがある。

| 優先度 | パス | 確認目的 | 起点脅威 | 主な確認点 |
|--------|------|----------|----------|------------|
| Medium | `frontend/src/app/favicon.ico.backup` | S, V | バックアップ残存 | バックアップの必要性、機密情報の残存 |
| Medium | `frontend/src/app/globals.css.backup` | S, V | バックアップ残存 | 同上 |
| Medium | `frontend/src/app/page.tsx.backup` | S, V | バックアップ残存 | 旧コード内の情報漏洩 |
| Medium | `frontend/src/components/layouts/Footer.tsx.20260212` | S, V | バックアップ残存 | 同上 |
| Medium | `frontend/src/components/layouts/Header.tsx.20260206` | S, V | バックアップ残存 | 同上 |
| Low | `frontend/dev.db` | S | 開発DB混入 | Git混入の履歴確認 |

---

### 3.10 コード全体への機械的検索（grep観点）

以下のパターンについて `src/` および設定ファイル全体を検索：

| 優先度 | 検索パターン | 検出時の意味 |
|--------|--------------|--------------|
| Critical | `password\s*[:=]\s*['"]` | パスワードのハードコーディング |
| Critical | `api[_-]?key\s*[:=]\s*['"]` | APIキーのハードコーディング |
| Critical | `secret\s*[:=]\s*['"][^$]` | シークレットのハードコーディング（環境変数参照除く） |
| Critical | `\$queryRawUnsafe`, `\$executeRawUnsafe` | Prismaの安全でない生SQL |
| High | `dangerouslySetInnerHTML` | XSS潜在箇所（レビュー指摘 Gemini 2.3 対応で強化） |
| High | `eval\s*\(`, `Function\s*\(` | 動的コード実行 |
| High | `child_process`, `exec\(`, `spawn\(` | OSコマンド実行 |
| High | `innerHTML\s*=`, `document\.write` | DOM経由のXSS |
| High | `http://` （httpsでない） | 平文通信URL |
| High | `as any`, `@ts-ignore`, `@ts-expect-error` | 型チェック迂回 |
| High | `'use client'` 宣言ファイルでの `process\.env\.` | クライアントバンドルへの環境変数混入 |
| High | `__proto__`, `constructor\.prototype` | Prototype Pollution 経路（ver 4.0 追加） |
| High | `\.bind\(null,` | Server Actions bind誤用（ver 4.0 追加） |
| High | `Object\.assign`, `\.\.\.input`, `\.\.\.formData` | Mass Assignment 経路（ver 4.0 追加） |
| Medium | `TODO`, `FIXME`, `XXX`, `HACK` | 未完了箇所 |
| Medium | `console\.log` | 本番環境での情報漏洩 |
| Medium | `localhost`, `127\.0\.0\.1`, `192\.168\.` | 開発用IPのハードコーディング |
| Medium | `@example\.com` | プレースホルダメールの残存 |
| Medium | `X-Forwarded-For` | IPアドレス取得の信頼性検証 |
| Medium | `target="_blank"` | 外部リンクの rel 属性確認（ver 4.0 追加、CL-01 対応） |
| Medium | `searchParams\.get\(`, `searchParams\.getAll\(` | HPP 対策確認（ver 4.0 追加） |
| Low | `debugger` | デバッグ文の残存 |

---

## 4. 未確認事項・ヒアリング候補（4分類・回答主体付き）

レビュー指摘（ChatGPT 4-1, 4-2 / Claude 12-3）対応で、**4分類** に再編し、各項目に **回答主体タグ** を付与する。ステップ6以降のskill.md化時、Claude Codeで回答可能な項目と人間へのヒアリングが必要な項目を分離するために用いる。

---

### 4.A 仕様確定が必要な項目

仕様決定者（プロダクトオーナー等）への確認が必要な項目。

| # | 項目 | 回答主体 |
|---|------|---------|
| 4.A.1 | ブログ記事のauthor単位の権限分離（staff全員が全記事を編集できる設計か、author別にスコープを持つか） | `[仕様決定]` |
| 4.A.2 | メール送信機能の実装方針（未実装のまま運用か、将来実装予定か） | `[仕様決定]` |
| 4.A.3 | ブログ本文・ニュース本文の入力形式（Markdown / HTML / リッチエディタ） | `[仕様決定]` |
| 4.A.4 | パスワードポリシー（最小文字数、使い回し禁止、有効期限、初期PW失効条件） | `[仕様決定]` |
| 4.A.5 | 初期パスワード・ワンタイムパスワード画面表示の自動非表示機能の要否 | `[仕様決定]` |
| 4.A.6 | 2FA無効化時の再認証・メール通知の要否 | `[仕样決定]` |
| 4.A.7 | 監査ログの削除・エクスポート機能の要否（WORM原則適用可否） | `[仕様決定]` |
| 4.A.8 | 運営者情報（会社名、所在地、連絡先）のサイト上への明示範囲（IPA-2.4） | `[仕様決定]` |
| 4.A.9 | 本番環境のDB選定（SQLiteのままか、PostgreSQL移行予定か） | `[仕様決定]` |
| 4.A.10 | 一括削除機能の上限値（CL-10の一括削除対象ID配列の最大件数） | `[仕様決定]` |
| 4.A.11 | 退職者アカウント無効化・権限変更承認フローの要否 | `[仕様決定]` |
| 4.A.12 | Credential Stuffing 対策としての漏洩パスワードDB（HIBP）連携の要否 | `[仕様決定]` |

---

### 4.B 本番環境構成の確認項目

本番運用担当者への確認が必要な項目。repo内では判定不能。

| # | 項目 | 回答主体 |
|---|------|---------|
| 4.B.1 | 本番ホストOSの種類・バージョン（Ubuntu Server か、他Linuxディストロか、バージョン） | `[Infra/運用]` |
| 4.B.2 | 本番環境の secrets 注入方法（Docker secrets / Kubernetes Secrets / Vault 等） | `[Infra/運用]` |
| 4.B.3 | 本番環境のリバースプロキシ構成（nginx、ALB、CloudFront 等） | `[Infra/運用]` |
| 4.B.4 | TLS終端の配置（Next.js直接 / リバースプロキシ / ALB 等） | `[Infra/運用]` |
| 4.B.5 | CDN / WAF の採用方針 | `[Infra/運用]` |
| 4.B.6 | HSTS preload 登録の予定 | `[Infra/運用]` |
| 4.B.7 | Fail2ban 導入有無と jail 設計（SSH、HTTP、カスタムjail） | `[Infra/運用]` |
| 4.B.8 | ホストファイアウォール設定（UFW / iptables / クラウドSG の使い分け） | `[Infra/運用]` |
| 4.B.9 | SSHハードニング実施状況（`PermitRootLogin`、鍵認証のみ、ポート変更、AllowUsers） | `[Infra/運用]` |
| 4.B.10 | unattended-upgrades の有効化 | `[Infra/運用]` |
| 4.B.11 | AppArmor プロファイルの適用状況 | `[Infra/運用]` |
| 4.B.12 | auditd 監査ルールの設定 | `[Infra/運用]` |
| 4.B.13 | ファイル整合性監視（AIDE / Tripwire）の導入 | `[Infra/運用]` |
| 4.B.14 | セキュリティ診断ツール（Lynis / chkrootkit / rkhunter）の定期実行 | `[Infra/運用]` |
| 4.B.15 | ログ集約基盤（ELK / Loki / CloudWatch / Datadog 等、3層ログ集約方針） | `[Infra/運用]` |
| 4.B.16 | DDoS対策の多層化（L3/L4/L7 各層、クラウドDDoS保護サービス） | `[Infra/運用]` |
| 4.B.17 | Bot対策（CAPTCHA、honeypot、Bot Management サービス） | `[Infra/運用]` |
| 4.B.18 | NTP/chrony による時刻同期 | `[Infra/運用]` |
| 4.B.19 | バックアップ戦略（RPO/RTO、3-2-1ルール、暗号化、復元テスト） | `[Infra/運用]` |
| 4.B.20 | CI/CDパイプラインのセキュリティ（secrets取扱い、自動スキャン、イメージ署名） | `[Infra/運用]` |
| 4.B.21 | 運用担当者アカウント管理（sudo権限、SSH鍵管理、離職時の失効手順） | `[Infra/運用]` |
| 4.B.22 | Docker Content Trust / イメージ署名運用 | `[Infra/運用]` |
| 4.B.23 | HTTP/2 Rapid Reset 攻撃（CVE-2023-44487）へのパッチ適用状況 | `[Infra/運用]` |
| 4.B.24 | サブドメイン運用方針、CDN解除時のDNSレコード削除プロセス | `[Infra/運用]` |
| 4.B.25 | TLS証明書の種類（DV/OV/EV）とフィッシング対策方針（IPA-2.4） | `[Infra/運用]` |
| 4.B.26 | メール送信時の SPF / DKIM / DMARC 設定（IPA-2.4） | `[Infra/運用]` |
| 4.B.27 | インシデント対応計画書（IRP）の整備状況と過去訓練の実施 | `[Infra/運用]` |
| 4.B.28 | 個人情報漏洩インシデント発生時の通知体制（個人情報保護委員会への72時間以内通知準備） | `[Infra/運用]` |

---

### 4.C 証跡取得が必要な項目

Claude Code がリポジトリ調査で回答可能な項目。ステップ8のClaude Code調査で重点的に取得する。

| # | 項目 | 回答主体 |
|---|------|---------|
| 4.C.1 | `src/lib/session.ts` の iron-session 設定の完全な内容（cookieOptions 全属性、password の出所と長さ、`password` が文字列か配列か） | `[Code]` |
| 4.C.2 | `src/lib/crypto.ts` の AES-256-GCM 実装（IV管理、auth tag検証、鍵導出、`import 'server-only'` 宣言） | `[Code]` |
| 4.C.3 | `src/lib/auth.ts` の `requireAuth`/`requireAdmin` の内部実装と、呼び出し忘れがあった場合の挙動 | `[Code]` |
| 4.C.4 | 全 `actions.ts` の先頭での認可チェック呼び出しの有無（機械的grep結果） | `[Code]` |
| 4.C.5 | `src/middleware.ts` の `matcher` 設定、Edge Runtime での iron-session 動作、cryptoモジュール呼び出しの有無 | `[Code]` |
| 4.C.6 | `next.config.ts` のCSP定義の詳細（公開用/staff用の具体内容、`'unsafe-inline'` 有無） | `[Code]` |
| 4.C.7 | `next.config.ts` の `serverActions.allowedOrigins`、`bodySizeLimit`、`encryptionKey`、`experimental.ppr` 設定 | `[Code]` |
| 4.C.8 | Prismaで `$queryRaw*` / `$executeRaw*` を使用している箇所の有無と内容 | `[Code]` |
| 4.C.9 | ブログ画像アップロードの保存パス組み立てロジックと sharp の設定 | `[Code]` |
| 4.C.10 | メール送信機能の実装状況（nodemailer等の使用有無） | `[Code]` |
| 4.C.11 | `src/lib/dto.ts` のバリデーションスキーマ内容（全フォーム分、`.strict()` 使用、ReDoS 対策） | `[Code]` |
| 4.C.12 | `src/lib/rate-limit.ts` のデータ構造と対象エンドポイント | `[Code]` |
| 4.C.13 | ログイン時のセッション再生成ロジック（`src/app/staff/login/actions.ts`） | `[Code]` |
| 4.C.14 | アカウントロック機構の具体的実装（試行回数、ロック期間、解除条件） | `[Code]` |
| 4.C.15 | `src/lib/prisma.ts` の `log` オプション設定と `globalThis` パターン | `[Code]` |
| 4.C.16 | `'use client'` 宣言モジュールでのシークレット参照の有無 | `[Code]` |
| 4.C.17 | `X-Forwarded-For` 等のプロキシヘッダの信頼性処理 | `[Code]` |
| 4.C.18 | bcryptjs の saltRounds 値 | `[Code]` |
| 4.C.19 | otplib の設定値（window, algorithm, digits, step） | `[Code]` |
| 4.C.20 | Prisma `where` / `data` での Mass Assignment 経路（`...input` スプレッド、`data: formData` の有無） | `[Code]` |
| 4.C.21 | Prototype Pollution 経路（`Object.assign`、スプレッド、再帰マージ） | `[Code]` |
| 4.C.22 | `dangerouslySetInnerHTML` の使用箇所と入力ソース | `[Code]` |
| 4.C.23 | Server Actions `.bind(null, ...)` パターンの使用有無 | `[Code]` |
| 4.C.24 | Route Handler（`app/**/route.ts`）の存在確認と認証チェック | `[Code]` |
| 4.C.25 | `users.oneTimePassword` カラムの保存形式（平文/ハッシュ/暗号化） | `[Code]` |
| 4.C.26 | React 19 taint API の使用有無（`src/lib/auth.ts` / `src/lib/session.ts`） | `[Code]` |
| 4.C.27 | `.env.local` のGit履歴確認（`git log --all --full-history -- frontend/.env.local`） | `[Code]` |
| 4.C.28 | `.gitignore` / `.dockerignore` の `dev.db*`（WAL/journal含む）の除外 | `[Code]` |
| 4.C.29 | `scripts/test-csrf.sh` の内容とシナリオ | `[Code]` |
| 4.C.30 | `scripts/test-session-fixation.sh` の内容とシナリオ | `[Code]` |
| 4.C.31 | `scripts/test-csrf.sh` の実施結果 | `[Repo docs]` |
| 4.C.32 | `scripts/test-session-fixation.sh` の実施結果 | `[Repo docs]` |
| 4.C.33 | `security-report/` 配下の既存レポートの指摘事項と対応状況 | `[Repo docs]` |
| 4.C.34 | `docs/architecture/` `docs/process/` 配下のドキュメント整備状況 | `[Repo docs]` |
| 4.C.35 | `pnpm audit` の実施結果 | `[Code]` |
| 4.C.36 | コンテナイメージの脆弱性スキャン実施有無 | `[Repo docs]` |
| 4.C.37 | 外部リンクの `rel="noopener noreferrer"` 付与状況（CL-01、ブログ本文） | `[Code]` |
| 4.C.38 | 監査ログの削除機能の有無（WORM原則） | `[Code]` |

---

### 4.D 今回は保留でもよい項目

将来的に重要だが、現時点の机上デバッグでは優先度が低い項目。次版以降で検討。

| # | 項目 | 回答主体 |
|---|------|---------|
| 4.D.1 | PostgreSQL移行時の `client_encoding` 設定（SQL別冊 A.2, A.3） | `[仕様決定]` |
| 4.D.2 | SBOM（Software Bill of Materials）生成運用 | `[Infra/運用]` |
| 4.D.3 | Subresource Integrity (SRI) の適用（外部CDN使用時のみ） | `[Code]` |
| 4.D.4 | Lockfile Poisoning 検知プロセス（`pnpm install --frozen-lockfile` のCI強制） | `[Infra/運用]` |
| 4.D.5 | Prisma 7 の新機能・変更点による設定差異の影響 | `[Code]` |
| 4.D.6 | Alpine Linux の musl libc 固有リスク | `[Infra/運用]` |
| 4.D.7 | Docker コンテナの `init: true` / dumb-init / tini の使用 | `[Code]` |
| 4.D.8 | Alpine のパッケージ署名検証（`apk add --no-cache package=version`） | `[Code]` |
| 4.D.9 | コンテナのファイルシステム書き込み領域の明確化（`read_only` + Volume） | `[Code]` |
| 4.D.10 | `$$ACTION_` ID 暗号化キー（`encryptionKey`）の設定 | `[Code]` |
| 4.D.11 | Prismaクエリのタイムアウトとサーキットブレーカー | `[Code]` |
| 4.D.12 | Prisma Migrate の Down migration 欠落対応（マイグレーション失敗時の復元手順） | `[Infra/運用]` |
| 4.D.13 | ログイン成功後の秘密情報発行（IPA-1.4 保険的対策 4-(iv)-b） | `[Code]` |

---

## 5. 本草案の限界と次版反映方針

### 5.1 ver 3.0 以前から継続する限界

- **SCA（Trivy）、SAST（Semgrep）での静的解析結果を含む予定だった観点が、現時点で空**: ステップ6以降でツール実行後に追補が必要
- **プライバシー・個人情報保護観点の一部欠落**: お問い合わせフォームの個人情報、監査ログのIP/UA保存期間（1.E.13 で部分対応）。通知体制は 4.B.28 で対応
- **Next.js 16 固有のCVE情報が具体的でない**: 観点F.1 で言及したが、具体的なCVE番号や影響バージョンの調査が未実施
- **開発者アカウント・GitHubセキュリティ**: リポジトリのブランチ保護、signed commits、Dependabot設定などのメタレベルの観点は 4.B.20 / 4.B.21 で部分対応
- **Secrets detection ツール（gitleaks, trufflehog）での履歴スキャン**: 観点として言及していないが、実施推奨

### 5.2 ver 4.0 で新たに対応した事項

レビュー結果を踏まえ、以下を ver 4.0 で改善した：

| 改善項目 | 対応箇所 |
|---|---|
| IPA-2.4（フィッシング対策）の追加 | 1.A.13、H.6 |
| 別冊「安全なSQLの呼び出し方」の観点深化 | 1.A.1.6、1.A.1.7 |
| Node.js / TypeScript 固有観点（Prototype Pollution、ReDoS、Mass Assignment、HPP、ランタイム型検証） | F.14、F.19、F.20、F.21、F.22 |
| React 19 / Next.js 16 / Prisma 7 固有観点 | F.3、F.5、F.6、F.10、F.11、F.23 |
| WebサーバOS・コンテナ層の深掘り | C.4、B.2、B.5、B.6、C.7（Medium格上げ） |
| ネットワーク層の深掘り | D.10、D.11、D.12、D.13、A.7.4 |
| DB層の深掘り | E.2、E.8、E.9、E.10、E.11 |
| サプライチェーン観点の追加 | F.24、F.25、F.26、F.27 |
| 悪意ある第三者からの攻撃観点（章新設） | 1.H 全体 |
| 画面分析資料からの具体リスク反映 | 2.B.15、2.B.16、2.B.17、2.B.18、2.B.19、H.7、H.8 |
| 情報露出・エラー管理の独立項目化 | 1.A.14 |
| 証拠区分（Confirmed/Suspected/Environment-dependent）の導入 | 0.2 |
| 回答主体タグの導入 | 0.3、4章 |
| 画面群 × 脅威マトリクスの新設 | 1.0 |
| 2章の3分割（2.A/2.B/2.C） | 2章 |
| 3章の repo内限定化 | 3章 |
| 4章の4分類化 | 4章 |
| 机上デバッグの範囲定義 | 0.7 |
| 7領域への修正 | 0.6 |
| 判定条件（確認対象/危険条件/除外条件/確認証跡）の4要素化 | 1章各観点 |
| 起点脅威の付与 | 3章 |

### 5.3 ver 4.0 で残る限界

- **運用防御層（1.G）の机上デバッグ範囲外指定**: ver 4.0 で 1.G を `Environment-dependent` 中心に整理したが、机上デバッグ（ステップ8のClaude Code調査）の対象外とした。そのため、1.G の大半の観点は本草案の修正では解消されず、4.B（ヒアリング項目）で別途対応する必要がある
- **観点間の依存関係・前後関係の明示**: レビュー指摘（13-1）の DAG 整理は未実施。2章の観点順序では一部依存があるが、可視化できていない
- **修正コストと影響度の2軸評価**: レビュー指摘（13-2）の `Trivial/Easy/Medium/Hard/Rearchitect` 軸は未導入。影響度のみの単軸評価
- **具体的なCVE番号・バージョン影響範囲の調査**: F.1、B.3等で言及したが、Web検索が必要。ステップ6以降で補完
- **フロントエンド（ブラウザ側）の深掘り**: Permissions Policy の個別ディレクティブ、Referrer Policy の適用範囲、Trusted Types（H.5で言及）の具体的導入方法は未深掘り

### 5.4 次版（ver 5.0）反映方針

レビュー指摘（ChatGPT 5-1）対応で、修正の方向性を明示する：

#### 次版で削るもの
- 1.G の細部（Fail2ban の jail 設計例、AIDE の監視対象リスト等）は運用側のチェックリストに分離し、本草案では観点タイトルのみに留める
- 版履歴の詳細（ver 1.0 → ver 2.0 → ver 3.0 の変遷説明）は簡素化

#### 次版で4章へ移すもの
- 1.G の全観点のうち、現状 `Environment-dependent` として残っているもの全てを 4.B への参照リンクに置き換える
- 観点B.6（Docker Content Trust）、観点D.5（DNS設定）等の純粋な運用観点は 4.B へ統合

#### 次版で2章へ昇格するもの
- 観点F.21（Mass Assignment）は、コード確認で発見された場合に即 Critical。現状は 2.B.2 に配置済みだが、3章での確認順序を最上位に
- 観点F.6 / 2.B.21（キャッシュによるクロスユーザ情報漏洩）は、発見時 Critical 相当のため、優先度表示を強調
- 観点E.5 / 2.B.14（`oneTimePassword` 保存形式）は、平文保存が確認された時点で Critical に昇格するフローを明示

### 5.5 既知の矛盾（次版で整理対象）

- **網羅型と優先度型の部分重複**: ver 4.0 では「重複許容ルールの限定（同じ論点の重複は1章と2章の二箇所まで、3章と4章では再掲しない）」を導入した。網羅型と優先度型の関係について、以下のいずれかを次版で決定する：
  - **案A（現行）**: 網羅型は「辞書」、優先度型は「実施順序」として役割分離
  - **案B**: 優先度型は網羅型の一部を本リポジトリ固有リスクとして再掲する補助と位置付け
  - **案C**: 優先度型を廃止し、網羅型に優先度を付与する方式に統合
- **観点H（悪意ある第三者）と 1.A〜1.F の重複**: ver 4.0 で攻撃者視点の横断章として新設したが、個別脆弱性（XSS、CSRF、認可）との重複がある。ver 5.0 で攻撃シナリオ軸として再整理するか検討

### 5.6 ステップ5-3 / ステップ6 への引き継ぎ事項

本草案から **Claude Code（ステップ8）へのヒアリング観点** として抽出すべき項目は、主に以下：

1. **4.C の全項目（38項目）**: Claude Code がリポジトリ調査で回答可能なため、skill.md の主要コンテンツ
2. **4.A の仕様確定項目**: Claude Code では確定できないが、skill.md に「仕様確認が必要」と記載しておくことで、調査結果レポートで仕様への疑問提示が可能
3. **2.A の Confirmed 事項**: 既に問題の兆候が確認されている項目として、skill.md の冒頭で「確認済み問題」を明示
4. **3章 の Critical / High ファイル**: 重点確認対象として skill.md で最優先に列挙
5. **3.0 の公開配信領域不要ファイル**: 最優先タスクとして skill.md に明記

**4.B / 4.D は skill.md の対象外**（運用担当者へのヒアリングまたは保留のため）。

---

## 版履歴まとめ

| 版 | 章立て | 観点数の目安 | 主な変更 |
|----|--------|--------------|---------|
| ver 1.0 | IPA 11脆弱性軸の網羅型、優先度型17項目、コード型 | 網羅型：約50、優先度型：17 | 初版 |
| ver 2.0 | 1章を6領域（A:IPA/B:インフラ/C:サーバ/D:ネットワーク/E:DB/F:フレームワーク）に拡張、優先度型22項目 | 網羅型：約100、優先度型：22 | 技術レイヤ拡張 |
| ver 3.0 | 1.G（運用・インフラ防御）を追加（7領域構成）、優先度型26項目、ヒアリング項目を53項目に拡充 | 網羅型：約114、優先度型：26、ヒアリング：53 | 運用防御層追加 |
| **ver 4.0** | 証拠区分・回答主体タグ導入、2章を3分割、1.A.13（フィッシング対策）/ 1.A.14（情報露出・エラー管理）/ 1.H（悪意ある第三者）新設、Node.js/TS固有観点・画面キャプチャ具体リスク・判定条件の4要素化 | 網羅型：約140、優先度型（3分割）：33、ヒアリング（4分類）：約80 | **3者レビュー結果反映** |

---

以上。
