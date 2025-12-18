# pimientito_blog_docs
* 概要：はてなブログ「生成AIと作るローカルLLMサーバ奮闘記」の記事で紹介したドキュメントを公開
* 詳細：生成AIと協働で臨んだ実験や検証で実際に使用した資料です。オンプレミスサーバ上に複数のAIエージェントが協調して動作する分散型システム構築を目指します。
* プロジェクト期間：：2025年7月～
* 関連ブログURL： https://pimientito-ai-study.hatenablog.com

## 免責事項等

### 1. 著作権および知的財産権について
* 本リポジトリで公開するコード、スクリプト、設定ファイル等は、生成AI（ChatGPT、Claude、Gemini）の出力を基に作成されている
* 生成AIの学習データには、インターネット上の様々な情報源が含まれており、意図せず既存のコードやアイデアと類似する可能性がある
* 万が一、本リポジトリの内容が第三者の著作権、特許権、商標権その他の知的財産権を侵害している場合は、本リポジトリの作成者への申告をもって速やかに該当箇所の修正または削除を行う
* 本リポジトリで公開するコードやドキュメントは、MITライセンスの下で公開されている

### 2. 技術的内容の正確性について
* 生成AIとの協働により作成されたコード、システム設計、実装方法は、完全な動作を保証するものではない
* 特に、vLLM、llama.cpp、Docker SDK、FastAPI等の外部ライブラリやツールの使用方法は、バージョン更新により仕様が変更される可能性がある
* 本システム上の株価予測機能に関する記述は、あくまで技術的な動作検証の一環であり、投資または投機の予測や助言を意図したものではない

### 3. システム実装時の注意事項
* 本リポジトリで公開する自律型AIシステムは、GPU温度管理、メモリ制御、並列処理など、高度な技術要素を多数含む
* 実装時のハードウェア損傷、データ損失、システム障害等について、本リポジトリの作成者は一切の責任を負わない
* 特に、複数のDockerコンテナを並列実行する際のリソース競合や、GPUの高温状態での継続稼働には十分注意すること

### 4. 外部APIおよびデータソースについて
* 本システムで使用する金融データAPI、ニュースAPI等の外部サービスは、各提供元の利用規約に従って使用すること
* スクレイピングによるデータ収集を行う場合は、対象サイトの利用規約およびrobots.txtを遵守すること
* APIのレート制限、利用料金、データの正確性については、各自で確認の上、自己責任で利用すること

### 5. 生成AI利用に関する注意
* 本リポジトリの内容は、本リポジトリの作成者と生成AI（ChatGPT Plus、Claude Pro、Gemini Pro）による共同作成物である
* 生成AIの出力には、事実と異なる情報（ハルシネーション）が含まれる可能性がある
* 生成AIから得られた情報は、各自で必ず複数の信頼できる情報源で検証することを推奨する

### 6. 本リポジトリでの公開について
* 本リポジトリ上で公開されるソースコード、ドキュメント、設定ファイル等は「現状有姿」（AS IS）で提供され、明示的または暗黙的な保証は一切ない
* プルリクエストやイシューによるコントリビューションは歓迎するが、採用の保証はない
* セキュリティ脆弱性を発見した場合は、公開イシューではなく、下記の連絡先へ報告することを推奨する
  * 連絡先：hiroyuki-kosho@arte-arte.com
* フォークやクローンによる二次利用は自由だが、オリジナルの著作者表示を維持すること

### 7. その他の免責事項
* 本リポジトリは個人の体験を述べたものであり、特定のサービス、製品、企業や団体、技術に対する批判や評価を意図したものではない
* 本リポジトリの情報を基に実施された開発、投資および投機、その他の行為により生じた損害について、本リポジトリの作成者は一切の責任を負わない
* 本免責事項は、必要に応じて予告なく変更される場合がある

### ライセンス
本リポジトリのコードおよびドキュメントは、MITライセンスの下で公開されています。

```
MIT License

Copyright (c) 2025 genai-dev-showcase

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
