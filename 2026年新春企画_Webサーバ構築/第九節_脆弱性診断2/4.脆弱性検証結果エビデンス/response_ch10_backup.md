# 回答ファイル: 第10章 バックアップ・不要ファイル確認

**対象チェックリスト:** `security-report/checklist_security_verification.md` ver 1.0
**回答対象:** 第10章 バックアップ・不要ファイル確認(3.9)
**回答日:** 2026-05-03
**回答者:** Claude Code
**参照コミット:** 1e75c36647736d45ad49f25a4448b51ffd9ec109
**処理対象:** 全重要度(Critical / High / Medium / Low)

---

## 回答マトリクス

| No. | 重要度 | 区分 | パス | ファイル名 | 質問(要約) | CC回答 | 評価(AI) | 判定根拠・備考 |
|---|---|---|---|---|---|---|---|---|
| J-1-01 | Medium | 3.9 | `frontend/src/app/`, `frontend/src/components/layouts/` | `*.backup`, `*.20260212`, `*.20260206` | 各ファイルと現行ファイルの差分、旧コードに残存する機密情報・APIキー・デバッグコード | 該当バックアップ 5 ファイル(A-1-03 / A-1-04 / A-1-05 で全件 Git 追跡対象として確認済):<br><br>**(a) `src/app/favicon.ico.backup`** (25,931 バイト): バイナリ ico、本作業では中身検査せず(セキュリティリスクは低い)<br><br>**(b) `src/app/globals.css.backup`** (488 バイト): Tailwind CSS の旧版。先頭20行確認、機密情報・API キーなし、純粋な CSS 変数定義のみ<br><br>**(c) `src/app/page.tsx.backup`** (3,117 バイト): トップページ旧版。先頭20行確認、「**有限会社〇〇〇**」というプレースホルダ会社名(現行は「有限会社アルテ」に置換済)、機密情報なし<br><br>**(d) `src/components/layouts/Footer.tsx.20260212`**: Footer 旧版。「**有限会社〇〇〇**」プレースホルダ、`target="_blank"` 3箇所(B-20-01 / K-3-06 で確認済、現行 `Footer.tsx` では使われていない)、機密情報なし<br><br>**(e) `src/components/layouts/Header.tsx.20260206`**: Header 旧版。`'use client'` コンポーネント、ナビ項目、機密情報なし<br><br>**機密情報 grep 結果**: `grep -EHni "key\|secret\|password\|token\|api[_-]?key" 各バックアップ` → React の `key={item.href}` 属性のみがヒット(偽陽性)。**実際の機密情報・API キー・デバッグコードは検出されず** ✅<br><br>**現行ファイルとの関係**: 現行 `Footer.tsx` / `Header.tsx` が同じディレクトリに存在し、最新版として運用中。バックアップは過去の作業時のスナップショットで、現行コードと並列に配置されている状態。 | 部分NG | **判定根拠**:<br>- **OK寄り(機密性)**: バックアップ内容に機密情報・API キー・デバッグコードは**含まれない** ✅<br>- **NG寄り(構造)**: ⚠️ ファイル自体が Git 追跡されており、`.gitignore` 不備(A-1-03)で除外対象に含まれていない<br>- **NG寄り(運用)**: 「有限会社〇〇〇」等のプレースホルダ文字列を含む旧コードが現行コードと併存 → 開発者の混乱の元、検索ヒットによる誤参照リスク<br>- **改善提案**:<br>  1. ⚠️ **これら 5 ファイルを削除**(`git rm` で履歴に残るが現状から除去)<br>  2. `.gitignore` に `*.backup` および `*.[12][09][0-9][0-9]*` パターン追加(A-1-03 改善)<br>  3. Git 履歴上は残るが、新規コミットで作業ツリーから消える<br>- 関連: A-1-03 / A-1-04 / A-1-05 / B-20-01 / K-3-06 / D-4-01 (.dockerignore も同様の不備)。 |

---

## 章サマリ

- **回答済項目数:** 1件
  - Critical: 0件
  - High: 0件
  - Medium: 1件
  - Low: 0件
- **OK 件数:** 0件
- **NG 件数:** 0件
- **部分NG 件数:** 1件 (J-1-01)
- **判定不能 件数:** 0件
- **判定不能の主な理由:** 該当なし

## 参照したファイル一覧

- `frontend/src/app/favicon.ico.backup`(サイズのみ確認、バイナリ)
- `frontend/src/app/globals.css.backup`(冒頭20行)
- `frontend/src/app/page.tsx.backup`(冒頭20行)
- `frontend/src/components/layouts/Footer.tsx.20260212`(冒頭20行)
- `frontend/src/components/layouts/Header.tsx.20260206`(冒頭20行)

## 実行した grep / コマンド一覧

- `head -20 src/app/*.backup src/components/layouts/*.20260212 src/components/layouts/*.20260206`
- `grep -EHni "key\|secret\|password\|token\|api[_-]?key" src/app/*.backup src/components/layouts/*.2026*` → 機密情報なし(`key={...}` のみが偽陽性)
- `ls src/components/layouts/` → 現行 Footer.tsx/Header.tsx と並列にバックアップが存在

## 章をまたぐ関連項目

- J-1-01 ⇔ A-1-03 (.gitignore 不備)、A-1-04 / A-1-05 (バックアップファイル個別)、B-20-01 / K-3-06 (target="_blank")、D-4-01 (.dockerignore)
