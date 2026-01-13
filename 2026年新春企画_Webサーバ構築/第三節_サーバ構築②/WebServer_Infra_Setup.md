# Webサーバ公開 基盤設定手順書(インフラ編)

Document-Version: v1.0 (GitHub公開版)
Last-Updated: 2026/01/13 22:30:00 +09:00(JST)
※ このドキュメントは生成AI（ChatGPT, Claude, Gemini）の協働により作成されました。

---

## ⚠️ セキュリティとプライバシーに関する注意

**本手順書について**:
- 本手順書は実際のサーバ構築プロジェクトで使用した手順書を基に作成されています
- 生成AIが作成した手順書を、事前に著者が校閲し再編集しています
- GitHub公開にあたり、以下の情報を匿名化・一般化しています：
  - ユーザ名、ドメイン名、IPアドレス
  - プロジェクト固有の名称
  - セキュリティクリティカルな具体的設定値

**伏字箇所の取り扱い**:
- `[username]`, `[your-domain.com]`, `[myproject]`, `XXX.XXX.XXX.XXX` 等の表記は、実際の環境に合わせて置き換えてください
- セキュリティ上重要な情報（SSHポート番号等）は一般的な説明に置き換えています
- 実際に構築する際は、ご自身の環境に合わせた値を設定してください

---

## 目的
Webサーバを**グローバル公開**するために必要なインフラ・バックエンド領域の設定手順を定義する。

**前提**: 
- LAN環境下での構築

**説明範囲外**：
- Ubuntu Server OSのインストール手順
- HTML/CSS/JavaScript 等のフロントエンド実装

---

## 0. 事前準備（サーバ側での初期ユーザ作成）

**⚠️ このセクションはUbuntu Serverに直接接続して実施**

目的：SSH接続用ユーザを作成し、パスワード認証接続を可能とする

### 👤 SSH接続用ユーザ作成

**目的**: rootログインを避け、sudo可能なユーザでSSH接続する

**推奨**：構築例では `webuser` を使用していますが、実際には推測困難なユーザ名を作成してください。

```bash
# ユーザ存在確認
if ! id webuser &>/dev/null; then
    sudo adduser webuser
    sudo usermod -aG sudo webuser
    echo "✅ ユーザ webuser を作成しました"
else
    echo "ℹ️ ユーザ webuser は既に存在します"
    # sudo権限の確認
    if groups webuser | grep -q sudo; then
        echo "✅ sudo権限も付与済みです"
    else
        sudo usermod -aG sudo webuser
        echo "✅ sudo権限を付与しました"
    fi
fi
```

**注意事項**:
- パスワード設定が対話的に求められます
- **このパスワードで初回SSH接続を行います**

### 🌐 IPアドレス確認（SSH接続に必要）

```bash
# IPアドレス確認（メモ必須）
ip a | grep "inet " | grep -v "127.0.0.1"
```

**期待結果**: 
- サーバのIPアドレスが表示される（このIPをメモ）
- 例: `inet 192.168.1.XXX/24` → **192.168.1.XXX をメモ**

**📋 メモ推奨事項**:
- サーバIPアドレス: ___________________
- ユーザパスワード: ___________________

**🎯 ここまで完了したら、Windows PCに移動します**

---

## 1. SSH鍵ペア生成（Windows側）

**🎯 Windows PC側での作業**

**目的**：クライアント側で鍵ペアを生成

### 📁 ディレクトリ準備

```
1. デスクトップでエクスプローラを開く

2. 以下のパスに移動:
   C:\Users\[ユーザ名]

3. 隠しフォルダを表示:
   エクスプローラ → 表示 → 隠しファイル にチェック

4. .ssh フォルダが存在しない場合、作成:
   右クリック → 新規作成 → フォルダ
   名前: .ssh

5. 以下のフォルダも作成（例）: ※フォルダ名は任意
   C:\Users\[ユーザ名]\Documents\TeraTermScripts
   C:\Users\[ユーザ名]\Documents\TeraTermLogs
```

### 🔑 SSH鍵ペア生成

```
1. コマンドプロンプトまたはPowerShellを管理者として起動
   （スタートメニュー → 「cmd」または「PowerShell」で検索 → 右クリック → 管理者として実行）

2. .sshディレクトリに移動:
   cd C:\Users\[ユーザ名]\.ssh

   ※ [ユーザ名]は実際のWindowsユーザ名に置換

3. SSH鍵ペア生成コマンド（ssh-keygen）実行:
   ssh-keygen -t rsa -b 4096 -C "webuser@web-server" -f server_id_rsa

4. パスフレーズの入力:
   Enter passphrase (empty for no passphrase): 
   → そのままEnterキーを押す（パスフレーズなし）
   
   Enter same passphrase again: 
   → そのままEnterキーを押す

5. SSH鍵ペア生成確認:
   dir
   → server_id_rsa（秘密鍵）
   → server_id_rsa.pub（公開鍵）
   の2つのファイルが生成されていることを確認

   **期待結果**:
   C:\Users\[ユーザ名]\.ssh\server_id_rsa
   C:\Users\[ユーザ名]\.ssh\server_id_rsa.pub

6. SSH公開鍵の内容を表示：
   type C:\Users\[ユーザ名]\.ssh\server_id_rsa.pub

   **期待結果**:
   ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC... webuser@web-server
   （実際には非常に長い1行のテキスト）

```

**⚠️ セキュリティ重要事項**:
- **秘密鍵（server_id_rsa）は絶対に第三者に渡さない**
- **秘密鍵はこのWindows PC上のみに保存**
- **サーバ側に秘密鍵を転送することは絶対にしない**

---

## 2. 初回SSH接続と公開鍵登録

**🎯 Windows/TeraTerm環境での作業**

目的：webサーバにSSH鍵（公開鍵）を登録するため、初回はパスワード接続して作業を実施

### 🔗 初回SSH接続（パスワード認証）

**Windows側での作業**:

```
1. TeraTerm起動 → ファイル → 新しい接続

2. 接続設定:
   ホスト: [サーバのIPアドレス]  ← セクション0でメモしたIP
   TCPポート: 22
   サービス: SSH
   SSHバージョン: SSH2
   → OK

3. 「SSH認証」ダイアログ:
   ユーザ名: webuser
   パスフレーズ: [セクション0で設定したパスワード]
   プレインパスワードを使う: チェック
   → OK

4. 初回接続時の確認:
   「ホストの鍵が登録されていません」と表示される
   → 「このホストをknown hostsリストに追加する」にチェック
   → 「続行」をクリック

5. 期待結果:
   - パスワード認証でログインできる
   - プロンプトが表示される: webuser@ubuntu:~$
```

### 📝 TeraTermログ設定

**重要**: 以降の作業をログに記録

```
TeraTerm接続中:
ファイル → ログ → 保存先を指定

推奨ファイル名: server_setup_YYYYMMDD_HHMMSS.log
例: server_setup_20251230_010000.log

保存先: C:\Users\[ユーザ名]\Documents\TeraTermLogs

```

**TeraTerm上でサーバへ公開鍵を登録**

**TeraTerm作業（webサーバ）**
```bash
# .sshディレクトリ作成（存在しない場合は、事前に作成する）
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# authorized_keys ファイルを編集
nano ~/.ssh/authorized_keys

# nanoエディタが開いてから、Windows上のコマンドプロンプトまたはPowerShellへ移動

```

**コマンドプロンプトまたはPowerShell作業**
```bash
# 公開鍵の内容を表示
type C:\Users\[ユーザ名]\.ssh\server_id_rsa.pub

# 表示された内容を全選択してコピー:
   - 「ssh-rsa AAAA...」で始まる1行全体
   - マウスで全選択 → 右クリック → コピー
   - または Ctrl+A → Ctrl+C でコピー
```

**TeraTerm作業（webサーバ）**
```bash

# コピーした公開鍵を貼り付け（TeraTerm上で右クリック → 貼り付け）

# nanoエディタ保存
Ctrl+O → Enter

# nanoエディタ終了
Ctrl+X（終了）

# ファイルの権限設定
chmod 600 ~/.ssh/authorized_keys

# 改行コードをLinux形式に変換（念のため）
dos2unix ~/.ssh/authorized_keys

# 必要な場合：dos2unix（改行コード変換ツール）インストール方法
# sudo apt update
# sudo apt install dos2unix -y

# 公開鍵が正しく登録されていることを確認
cat ~/.ssh/authorized_keys

# 期待結果: 
# Windows側で生成した公開鍵（ssh-rsa AAAA... webuser@web-server）が表示される

# 権限確認
ls -la ~/.ssh/

# 期待結果:
# drwx------ webuser webuser .ssh
# -rw------- webuser webuser authorized_keys

# webサーバからログアウト
exit

# TeraTermが終了

```

---

## 3. 鍵認証による接続テスト

**🎯 Windows/TeraTerm環境での作業**

目的：鍵認証（パスワードなし）で、ログインできることを確認

### 🔗 鍵認証でのSSH接続

**Windows側での作業**:

```
1. TeraTerm起動 → ファイル → 新しい接続

2. 接続設定:
   ホスト: [サーバのIPアドレス]
   TCPポート: 22
   サービス: SSH
   SSHバージョン: SSH2
   → OK

3. 【初回接続時のみ】セキュリティ警告:
   - ホスト鍵のフィンガープリント（暗号化画像のような文字列）が表示
   - 「このホストをknown hostsリストに追加しますか？」
   → 「続行」または「はい」をクリック

4. 「SSH認証」ダイアログ:
   ユーザ名: webuser
   パスフレーズ: (空欄のまま)
   「RSA/DSA/ECDSA/ED25519鍵を使う」にチェック
   秘密鍵: C:\Users\[ユーザ名]\.ssh\server_id_rsa を選択
   → OK

5. 期待結果:
   - パスワード入力なしでログインできる
   - プロンプトが表示される: webuser@web-server:~$
```

**⚠️ トラブルシュート**:

接続できない場合:
```
1. 秘密鍵のパスが正しいか確認
2. サーバのIPアドレスが正しいか確認
3. サーバ側のauthorized_keysに公開鍵が正しく登録されているか確認
   （別のTeraTermでwebサーバへ再度パスワード接続し`cat ~/.ssh/authorized_keys`を実行）
   期待結果：
   Windows側で生成した公開鍵（ssh-rsa AAAA... webuser@web-server）が表示される

4. サーバ側の権限を確認
   （別のTeraTermでwebサーバへ再度パスワード接続し`ls -la /home/webuser/.ssh/`を実行）
   期待結果:
   drwx------ webuser webuser .ssh
   -rw------- webuser webuser authorized_keys
```

---

## 4. TeraTerm自動ログインスクリプト作成

**目的**: 毎回手動で接続設定・ログ設定を行う手間を省き、作業を効率化する

### 📝 自動ログインスクリプト作成

**Windows側での作業**:

```
1. メモ帳やエディタを開く
2. 以下の内容をコピー＆ペースト: ※「```teraterm」から「```」の間
```

```teraterm
; AutoSSH_Logger_Web_Server.ttl
; Automatically SSH login using public key and start logging with timestamped filename

; ============================================================
; === 設定パラメータ（環境に合わせて変更してください） ===
; ============================================================

; サーバのIPアドレス
HOST = '192.168.1.XXX'

; SSHポート
PORT = '22'

; ログインユーザ名
USERNAME = 'webuser'

; 秘密鍵ファイルのパス（Windowsのパス形式で指定）
; ⚠️ 注意: [ユーザ名]の部分を実際のWindowsユーザ名に置き換えてください
KEY_FILE_PATH = 'C:\Users\[ユーザ名]\.ssh\server_id_rsa'

; ログファイル保存先ディレクトリ
LOG_DIR_PATH = 'C:\Users\[ユーザ名]\Documents\TeraTermLogs'

; ============================================================

; === タイムスタンプ付きログファイル名を生成 ===
getdate LOG_FILE_NAME '%Y%m%d_%H%M%S_web_server.log'

; === ログディレクトリの存在確認・作成 ===
foldersearch LOG_DIR_PATH
if result=0 then
    NEW_DIR = 'cmd /c mkdir "'
    strconcat NEW_DIR LOG_DIR_PATH
    strconcat NEW_DIR '"'
    exec NEW_DIR
endif

; === SSH接続文字列を構築 ===
COMMAND = HOST
strconcat COMMAND ':'
strconcat COMMAND PORT
strconcat COMMAND ' /ssh /2 /auth=publickey /user='
strconcat COMMAND USERNAME
strconcat COMMAND ' /keyfile="'
strconcat COMMAND KEY_FILE_PATH
strconcat COMMAND '"'

; === SSH接続実行 ===
connect COMMAND

; === ログファイルパスを構築 ===
LOG_FILE_FULL_PATH = LOG_DIR_PATH
strconcat LOG_FILE_FULL_PATH '\'
strconcat LOG_FILE_FULL_PATH LOG_FILE_NAME

; === ログ記録開始 ===
; パラメータ: ログファイル名, バイナリモード(0=テキスト)
logopen LOG_FILE_FULL_PATH 0

; === 接続成功メッセージ（オプション） ===
; 以下のコメントを外すと接続成功時にメッセージボックスが表示されます
; messagebox 'SSH接続が確立され、ログ記録を開始しました。' 'TeraTerm 自動SSH接続'
```

```
3. 以下の箇所を**必ず**環境に合わせて変更:
   - HOST: サーバのIPアドレス（セクション0でメモしたIP）
   - PORT: SSHポート（初期は22、後でカスタムポートに変更）
   - KEY_FILE_PATH: [ユーザ名]を実際のWindowsユーザ名に置き換え
   - LOG_DIR_PATH: [ユーザ名]を実際のWindowsユーザ名に置き換え

4. 名前を付けて保存:
   保存先: C:\Users\[ユーザ名]\Documents\TeraTermScripts
   ファイル名: AutoSSH_Logger_Web_Server.ttl
   ファイルの種類: すべてのファイル (*.*)
   文字コード: ANSI（または Shift-JIS）

5. 保存
```

### ✅ スクリプト動作確認

```
1. TeraTermを**完全に終了**

2. エクスプローラで以下を開く:
   C:\Users\[ユーザ名]\Documents\TeraTermScripts

3. AutoSSH_Logger_Web_Server.ttl を右クリック
   → 「プログラムから開く」→ 「TeraTerm」を選択

4. 期待動作:
   - 自動的にSSH接続が確立される
   - パスワード入力なしでログインできる
   - TeraTermLogs フォルダにログファイルが自動生成される
     （ファイル名例: 20251230_010000_web_server.log）
```

**トラブルシュート**:
- 接続できない場合: スクリプト内のパラメータ（HOST, PORT, KEY_FILE_PATH等）を再確認
- ログが記録されない場合: LOG_DIR_PATH のパスが正しいか確認
- 「秘密鍵が見つからない」エラー: KEY_FILE_PATH のバックスラッシュ（\または￥）と引用符を確認

### 📋 スクリプトメンテナンスのポイント

**ポート変更後の対応**:

SSH接続セキュリティ強化のためポート番号を変更した場合、スクリプト内の以下の行を変更:

```teraterm
; 変更前
PORT = '22'

; 変更後（例）
PORT = '22222'
```

変更後、新しいスクリプトを保存して実行すれば、変更後のポートで自動接続できます。

---

## 5. 接続確認とシステム初期化

**⚠️ ここからwebサーバの構築作業となります**

**目的**：Windows上のTeraTermからwebサーバへのSSH接続確認と、サーバOSのアップデート作業

```bash
# 現在のユーザ確認
whoami
# → webuser

# sudo権限確認
sudo -v
# パスワード入力後、エラーなし

# OSバージョン確認
cat /etc/os-release | grep VERSION

# パッケージリスト更新と適用（初回必須）
sudo apt update && sudo apt upgrade -y

# 再起動判定
if [ -f /run/reboot-required ]; then
    echo "⚠️ システム再起動が必要です"
    echo "sudo reboot を実行してください"
else
    echo "✅ 再起動は不要です"
fi

# 必要に応じて再起動
# sudo reboot
# （再起動後、再度SSH接続してから次の手順へ）
# サーバを再起動した場合、SSH接続が切断されTeraTermが終了します

# タイムゾーン確認・設定
timedatectl
# 必要に応じて: sudo timedatectl set-timezone Asia/Tokyo

# ホスト名確認
hostnamectl

# ネットワーク疎通確認
ping -c 3 8.8.8.8

# DNS確認
resolvectl status
```

**期待結果**:
- Ubuntu 24.04.2 LTS が表示される
- タイムゾーン: Asia/Tokyo
- ホスト名が適切に設定されている
- 外部疎通が正常
- DNSサーバが適切に設定されている

### 🔍 鍵認証の動作確認

```bash
# 接続ログ確認
sudo tail -n 20 /var/log/auth.log | grep "Accepted publickey"
```

**期待結果**: `Accepted publickey for webuser` という行が表示される

---

## 6. SSHセキュリティ強化（ポート変更・パスワード認証無効化）

### 📝 設定前のバックアップ

```bash
# 必ず`sshd_config`ファイルを事前にバックアップする
sudo cp -p /etc/ssh/sshd_config /etc/ssh/sshd_config.backup.$(date +%Y%m%d)
```

### 🔧 コンフィグファイル編集

```bash
sudo nano /etc/ssh/sshd_config
```

**変更箇所**:
```conf
# ポート番号変更（22 → カスタムポート）
Port [custom-port]  # 例: 22222

# root直接ログイン禁止
PermitRootLogin no

# パスワード認証禁止（鍵認証のみ）
PasswordAuthentication no
PubkeyAuthentication yes
```

### ⚠️ 設定検証と再起動

**注意**：新しいポート番号でのSSH接続が確認できるまで、作業中のTeraTermコンソールは閉じないこと（sshd_configファイルの修正ができなくなります）

```bash
# 設定ファイルの構文チェック
sudo sshd -t

# 期待結果: エラーが表示されないこと（何も表示されないこと）

# 現在のセッションを維持したまま、SSHデーモン（プロセス）を再起動
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo systemctl restart ssh

# 状態確認
sudo systemctl status ssh --no-pager

```

**トラブルシュート**:
- `sshd -t` でエラーが出た場合: バックアップファイルから復元

### 🔄 TeraTerm設定変更（新ポート対応）

SSHサービス再起動後、TeraTermの接続設定を変更します。

**手動接続の場合**:
```
TeraTerm起動 → ファイル → 新しい接続

ホスト: [サーバのIPアドレス]
TCPポート: 22222  ← 変更後のポート番号
サービス: SSH
SSHバージョン: SSH2
→ OK

「SSH認証」ダイアログ:
ユーザ名: webuser
「RSA/DSA/ECDSA/ED25519鍵を使う」にチェック
秘密鍵: C:\Users\[ユーザ名]\.ssh\server_id_rsa
→ OK
```

**自動ログインスクリプトを修正する場合**:

```
1. メモ帳で AutoSSH_Logger_Web_Server.ttl を開く

2. 以下の行を変更:
   PORT = '22'
   ↓
   PORT = '22222'

3. 上書き保存

4. スクリプトを実行して新ポートで接続を確認
```

**期待結果**: 
- 新ポート[custom-port]で鍵認証接続が成功
- パスワード入力は求められない

---

## 7. ファイアウォール（ufw）設定

### 🔒 基本ポリシー設定

```bash
# ufw状態確認（初回は inactive）
sudo ufw status

# 基本ポリシー
sudo ufw default deny incoming   # サーバへの通信はデフォルト拒否（必要なポートのみ明示的に許可）
sudo ufw default allow outgoing  # サーバから外部への通信はデフォルト許可（パッケージ更新・外部API呼び出し等が可能）

# SSH許可（変更後のポート番号を指定）
sudo ufw allow [custom-port]/tcp comment 'SSH'

# HTTP/HTTPSプロトコル許可
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# ファイアウォール有効化
sudo ufw enable

# 表示例：
# `Command may disrupt existing ssh connections. Proceed with operation (y|n)?` 
#（このコマンドは既存のSSH接続を中断する可能性があります。操作を続行しますか？）と表示された場合、'y'と入力し、Enterを押下します。

# `Firewall is active and enabled on system startup`と、表示されればOK

# 状態確認
sudo ufw status verbose
```

**期待結果**:
```
Status: active
To                         Action      From
--                         ------      ----
[custom-port]/tcp          ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

**⚠️ 注意**:
- `ufw enable` 実行前に必ずSSHポートを許可すること（SSHポート許可が漏れているとSSH接続が不可となります）
- SSHポート許可設定が漏れてファイアウォールを有効化した場合、直接、webサーバへログインし修正してください

---

## 8. fail2ban（侵入防止）設定

### 📦 パッケージインストール

```bash
# インストール状況確認
if dpkg -l | grep -q fail2ban; then
    echo "ℹ️ fail2ban は既にインストール済みです"
else
    sudo apt install -y fail2ban
    echo "✅ fail2ban をインストールしました"
fi

# 自動起動設定
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# 状態確認
sudo systemctl status fail2ban --no-pager
sudo fail2ban-client status
```

**期待結果**:
- fail2ban.serviceが**enabled**、**active（running）**であること
- fail2ban-clientが未設定のため、何も表示されない

### 🔧 SSH用カスタム設定

```bash
# カスタム設定ファイル作成
sudo nano /etc/fail2ban/jail.local
```

```conf
[DEFAULT]
bantime = 3600  # 不正アクセス元IPアドレスブロック時間（3600秒＝1時間）
findtime = 600  # 不正アクセス元IP監視期間（600秒＝10分）
maxretry = 3    # 不正アクセス最大回数（3回目でブロック）

# 自己BAN対策：作業元IPを除外
# ⚠️ LAN環境の場合、LAN内のIPアドレス範囲を指定
ignoreip = 127.0.0.1/8 ::1 192.168.1.0/24

# 特定IPのみ除外する場合:
# ignoreip = 127.0.0.1/8 ::1 192.168.1.XXX

[sshd]
enabled = true
port = [custom-port]  # 実際のSSHポート番号を指定
logpath = /var/log/auth.log
```

```bash
# 設定反映
sudo systemctl restart fail2ban

# jail状態確認
sudo fail2ban-client status sshd
```

**期待結果**：
- 監視状態が表示される

表示例：
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:

---

## 9. OS自動更新設定（セキュリティパッチ適用）

**目的**: セキュリティパッチを自動適用し、長期運用の安全性を確保する

### 📦 パッケージインストール

```bash
# インストール状況確認
if dpkg -l | grep -q unattended-upgrades; then
    echo "ℹ️ unattended-upgrades は既にインストール済みです"
else
    sudo apt install -y unattended-upgrades
    echo "✅ unattended-upgrades をインストールしました"
fi

# 自動更新有効化
sudo dpkg-reconfigure -plow unattended-upgrades
# GUI画面「Configuring unattended-upgrades」が表示されたら
# → 「はい（Yes）」を選択
```

### 🔧 自動更新設定

```bash
# 設定ファイル編集
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

**補足**
- アップデート種別を設定
- アップデート動作を詳細設定

**推奨設定**:
```conf
// 通常リリースを除外（セキュリティのみに限定する場合）
Unattended-Upgrade::Allowed-Origins {
        "${distro_id}:${distro_codename}-security";              // セキュリティパッチ（OS）
        "${distro_id}ESMApps:${distro_codename}-apps-security";  // セキュリティパッチ（アプリケーション/パッケージ）
        "${distro_id}ESM:${distro_codename}-infra-security";     // セキュリティパッチ（インフラ）
};

// 自動削除設定を有効化
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";  // 再起動は手動で実施

```

### 📅 自動更新頻度設定

```bash
# 自動更新タイミング設定
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```

**補足**
- 自動アップデートのON/OFFを制御
- 実行頻度を設定

```conf
APT::Periodic::Update-Package-Lists "1";           // パッケージリスト更新（毎日）
APT::Periodic::Download-Upgradeable-Packages "1";  // ダウンロード（毎日）
APT::Periodic::AutocleanInterval "7";              // 不要パッケージ削除（7日毎）
APT::Periodic::Unattended-Upgrade "1";             // 自動アップグレード（毎日）
```

### ✅ 動作確認

```bash
# 自動更新のドライラン（実際には実行されない）
sudo unattended-upgrades --dry-run --debug

# 自動更新ログ確認
sudo tail -f /var/log/unattended-upgrades/unattended-upgrades.log

# 自動更新ログのリアルタイム監視を強制終了
Ctrl + C
```

**期待結果**:
- `--dry-run`でエラーメッセージが出ていないこと
- ログファイルが存在すること(`/var/log/unattended-upgrades/`)

**⚠️ 運用上の注意**:
- 自動再起動は本番環境では無効化推奨（計画的な再起動が望ましい）
- 重要なアップデート後は手動で `sudo reboot` を実行
- 再起動の必要性は、以下の手順「# ファイルの存在確認」参照

```bash
# ファイルの存在確認
# "ファイル"とは、OSやパッケージが自動更新され再起動が必要な際、自動的に作成される
test -f /run/reboot-required && echo "再起動が必要です" || echo "再起動は不要です"
  
# または
ls -l /run/reboot-required 2>/dev/null

```

**期待結果**
- ファイルが見つからない場合、再起動は不要

### 📋 推奨運用スケジュール

**目的**: セキュリティパッチの完全適用と安定運用の両立

自動更新は毎日実行されますが、カーネルアップデート等の反映には再起動が必要です。
以下の運用スケジュールを推奨します：

**月次メンテナンススケジュール**:
```
毎月第3日曜日 午前3:00
1. 再起動前の確認
   - アクセス状況の確認（ピークタイムを避ける）
   - バックアップの完了確認
   
2. 再起動実行
   sudo reboot
   
3. 再起動後の確認（約5分後）
   - SSH接続確認
   - nginx稼働確認: sudo systemctl status nginx
   - 各種サービス稼働確認
```

**確認コマンド**:
```bash
# 再起動が必要かどうかの確認
test -f /run/reboot-required && echo "再起動が必要です" || echo "再起動は不要です"

# 最後の再起動日時確認
who -b

# システム稼働時間確認
uptime
```

**運用カレンダー例**:
- 第1週: 通常運用
- 第2週: 通常運用
- **第3週日曜日**: 定期メンテナンス（再起動）
- 第4週: 通常運用

**⚠️ 緊急時の対応**:
重大なセキュリティパッチが公開された場合は、スケジュール外でも速やかに再起動を実施してください。

---

## 10. nginx基本構成

### 📦 パッケージインストール

```bash
# インストール状況確認とインストール
for pkg in nginx; do
    if dpkg -l | grep -q "^ii  $pkg"; then
        echo "ℹ️ $pkg は既にインストール済みです"
    else
        sudo apt install -y $pkg
        echo "✅ $pkg をインストールしました"
    fi
done
```

**パッケージの用途**:
- nginx: Webサーバ本体

### 📁 ディレクトリ設計と作成

**ディレクトリ階層**
```text
/var/www/myproject/
├── data/          # データファイル
├── logs/          # アクセスログ・エラーログ
├── public/        # 公開HTML/CSS/JS
└── scripts/       # 管理スクリプト
```

```bash
# ディレクトリ作成（べき等性確保）
sudo mkdir -p /var/www/myproject/{public,logs,data,scripts}

# 所有権設定
# webuser: 管理者、www-data: nginx実行ユーザ
sudo chown -R webuser:www-data /var/www/myproject

# 権限設定（最小権限の原則）
sudo chmod -R 750 /var/www/myproject

# 確認
ls -la /var/www/myproject
```

**期待結果**:
```
drwxr-x--- webuser www-data data
drwxr-x--- webuser www-data logs
drwxr-x--- webuser www-data public
drwxr-x--- webuser www-data scripts
```

---

## 11. nginx仮想ホスト設定

**nginx仮想ホスト設定の推奨**

**補足**：
- 物理サーバ（クラウドサーバ含む）1台に対して複数サイトを構築する場合、管理が容易になります
- `/etc/nginx/conf.d/default.conf`を直接編集するよりファイル管理が効率的です

### 📝 設定ファイル作成

```bash
sudo nano /etc/nginx/sites-available/myproject
```

```nginx
server {
    listen 80;
    server_name _;  # HTTPS化する場合、実在のドメイン名に変更必須。(例: server_name example.com www.example.com;)
                    # 重要：セクション14でHTTPS化する際、この行を変更しないと証明書取得に失敗します

    root /var/www/myproject/public;
    index index.html;

    access_log /var/www/myproject/logs/access.log;
    error_log  /var/www/myproject/logs/error.log;

    # 静的ファイル配信
    location / {
        try_files $uri $uri/ =404;
    }

    # セキュリティヘッダー
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
}
```

**📋 ドメイン名設定について**

- nginx設定ファイル`/etc/nginx/sites-available/myproject`内の`server_name _;`は **「すべてのホスト名」を受け入れるという意味**
- **将来的にHTTPS化（Let's Encrypt）する場合は、必ず実際のドメイン名に変更してください。**
- Let's Encryptは無料のSSL/TLS証明書を自動発行する認証局（CA）
   - 無料でHTTPS化が可能
   - 証明書の有効期限：90日（自動更新可能）
   - コマンドライン（certbot）で取得・更新
- 用途：WebサイトをHTTPからHTTPSに変更し、通信を暗号化する際に使用

変更例：
```nginx
# 変更前
server_name _;

# 変更後（独自ドメイン取得後）
server_name example.com www.example.com;
```

**変更が必要なタイミング**:
1. 独自ドメインを取得した後
2. DNSレコードでサーバのグローバルIPを設定した後
3. サーバをインターネットに公開した後

### 🔗 有効化と検証

```bash
# シンボリックリンク作成（べき等性確保）
if [ ! -L /etc/nginx/sites-enabled/myproject ]; then
    sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
    echo "✅ 仮想ホストを有効化しました"
else
    echo "ℹ️ 仮想ホストは既に有効化済みです"
fi

# デフォルトサイト無効化
# nginxインストール時の初期サンプルサイトを削除し、myprojectのみ有効化
if [ -L /etc/nginx/sites-enabled/default ]; then
    sudo rm /etc/nginx/sites-enabled/default
    echo "✅ デフォルトサイトを無効化しました"
fi

# 設定ファイル構文チェック
sudo nginx -t

# 期待結果:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# nginx再読み込み
sudo systemctl reload nginx

# 状態確認
sudo systemctl status nginx --no-pager
```

### 🧪 動作確認用ファイル作成

```bash
# テスト用HTMLファイル
cat <<'EOF' | sudo tee /var/www/myproject/public/index.html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>Under Construction</title>
</head>
<body>
    <h1>Work in Progress</h1>
    <p>このサイトは構築中です。</p>
</body>
</html>
EOF

# 権限修正
sudo chown webuser:www-data /var/www/myproject/public/index.html
```

### ✅ 疎通確認

```bash
# nginx稼働確認
sudo systemctl status nginx --no-pager

# ポート80でリッスン確認
sudo ss -tlnp | grep :80
```

**期待結果**:
- nginx: active (running)
- ポート80でnginxがリッスン中

**Windows PC（LAN内）からの確認**:
```
1. Webブラウザを開く

2. アドレスバーに入力:
   http://[構築中サーバのIPアドレス]/
   
   例: http://192.168.1.10

3. 期待結果:
   「Work in Progress - このサイトは構築中です。」ページが表示される
```

---

## 12. バックアップ戦略（スクリプト・cron・リストア）

**目的**: システム障害時の復旧を可能にし、データ損失を防ぐ

### 📋 バックアップ対象

```text
/etc/fail2ban/jail.local              # fail2ban設定
/etc/letsencrypt/                     # SSL/TLS証明書
/etc/nginx/sites-available/myproject  # nginx設定ファイル
/etc/ssh/sshd_config                  # SSH設定
/etc/ufw/                             # ファイアウォール設定
/var/www/myproject/
├── data/                             # データファイル
├── logs/                             # ディレクトリ構造のみ（ログファイルは、バックアップ対象外）
├── public/                           # 公開HTML/CSS/JS
└── scripts/                          # 管理スクリプト
```

### 🔧 バックアップスクリプト作成

```bash
# バックアップスクリプト作成
sudo nano /var/www/myproject/scripts/backup.sh
```

```bash
#!/bin/bash
# Web Server Backup Script
# 実行: sudo /var/www/myproject/scripts/backup.sh

# バックアップ保存先
BACKUP_DIR="/var/backups"
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="server_backup_${BACKUP_DATE}.tar.gz"

# バックアップディレクトリ作成
mkdir -p ${BACKUP_DIR}

# バックアップ実行
echo "🔄 バックアップを開始します..."

tar -czf ${BACKUP_DIR}/${BACKUP_FILE} \
    --exclude='/var/www/myproject/logs/*' \
    /etc/fail2ban/jail.local \
    /etc/letsencrypt/ \
    /etc/nginx/sites-available/myproject \
    /etc/ssh/sshd_config \
    /etc/ufw \
    /var/www/myproject/data \
    /var/www/myproject/logs \
    /var/www/myproject/public \
    /var/www/myproject/scripts \
    2>/dev/null

# 結果確認
if [ $? -eq 0 ]; then
    echo "✅ バックアップが完了しました: ${BACKUP_FILE}"
    ls -lh ${BACKUP_DIR}/${BACKUP_FILE}

    # 古いバックアップ削除（30日以上前）
    find ${BACKUP_DIR} -name "server_backup_*.tar.gz" -mtime +30 -delete
    echo "🗑️ 30日以上前のバックアップを削除しました"
    echo "✅ バックアップ処理が完了しました"
else
    echo "❌ バックアップに失敗しました"
fi
```

```bash
# スクリプトに実行権限付与
sudo chmod +x /var/www/myproject/scripts/backup.sh

# 初回実行テスト
sudo /var/www/myproject/scripts/backup.sh
```

### 📅 自動バックアップ設定（cron）

```bash
# cron設定編集
sudo crontab -e
```

**期待結果**:

- 選択項目の表示

```bash
no crontab for root - using an empty one

Select an editor.  To change later, run 'select-editor'.
  1. /bin/nano        <---- easiest
  2. /usr/bin/vim.basic
  3. /usr/bin/vim.tiny
  4. /bin/ed

Choose 1-4 [1]:
```

```bash
1. エディタ選択（nanoを推奨）
   `Select an editor. To change later, run 'select-editor.'`
   `Choose 1-4[1]:`に、1を入力してEnter

2. cron設定を追加
   # Web Server 自動バックアップ（毎日午前3時）
   0 3 * * * /var/www/myproject/scripts/backup.sh >> /var/log/server_backup.log 2>&1

3. 保存して終了
   Ctrl + O → Enter → Ctrl + X

4. 設定確認
   sudo crontab -l

5. cronジョブのテスト実行（ログファイルへ出力）
   sudo touch /var/log/server_backup.log
   sudo bash -c '/var/www/myproject/scripts/backup.sh >> /var/log/server_backup.log 2>&1'

6. ログファイル生成確認
   ls -lh /var/log/server_backup.log

7. ログ内容確認
   sudo cat /var/log/server_backup.log

```

### 🔄 リストア（復元）手順

**障害時の復元方法**:

```bash
# 1. 最新のバックアップファイル確認
ls -lht /var/backups/ | head

# 2. 復元実行（例）
# ⚠️ 注意: 既存ファイルは上書きされます

cd /
sudo tar -xzf /var/backups/server_backup_YYYYMMDD_hhmmss.tar.gz
```

**期待される結果:**
以下のファイルが元の場所に上書き復元されます:
├─ /etc/nginx/sites-available/myproject (nginx設定)
├─ /etc/ssh/sshd_config (SSH設定)
├─ /etc/fail2ban/jail.local (fail2ban設定)
├─ /etc/letsencrypt/ (SSL証明書)
├─ /var/www/myproject/public/ (公開ファイル)
├─ /var/www/myproject/data/ (データファイル)
└─ /var/www/myproject/scripts/ (スクリプト)

```bash
# 3. nginx再読み込み
sudo nginx -t
sudo systemctl reload nginx

# 4. SSH再起動（設定を復元した場合）
sudo systemctl restart ssh
```

**⚠️ 重要な注意事項**:
- 復元前に現在の設定をバックアップすること
- 本番環境での復元は慎重に実施
- 可能であればテスト環境で復元テストを実施

### 📍 バックアップのベストプラクティス

1. **バックアップ頻度**: 毎日1回（深夜）
2. **保存期間**: 30日間
3. **保存場所**: サーバローカル + 外部ストレージ（推奨）
4. **外部バックアップ**: 定期的にWindows PCやクラウドストレージにコピー
5. **復元テスト**: 月1回は復元テストを実施

### ⚠️ 運用上の重要事項

**バックアップの外部保存は必須です**

`/var/backups` はOS再インストール時に消失します。以下のリスクに備えて、**必ず外部への定期的なコピー**を実施してください。

**想定されるリスク**:
- ディスク障害によるデータ完全消失
- OS再インストールが必要な深刻なシステム障害
- ランサムウェア等による暗号化被害

**推奨する外部バックアップ方法**:

**Windows PCへの定期コピー**
```bash
# Windows側での作業（コマンドプロンプトまたはPowerShell）

# 1. バックアップ保存先ディレクトリ作成
mkdir C:\Users\[ユーザ名]\{任意の場所}\ServerBackups
例：mkdir C:\Users\[ユーザ名]\Documents\ServerBackups

# 2. SCPでサーバからコピー（例：週1回実施）
# ※ TeraTerm SCPやWinSCPを使用
# 接続先: [サーバIPアドレス] (ポート[custom-port])
# リモートディレクトリ: /var/backups/
# ローカルディレクトリ: C:\Users\[ユーザ名]\Documents\ServerBackups
```

**外部バックアップの運用ルール**:
- **頻度**: 最低でも週1回
- **タイミング**: 毎週月曜日の業務開始前（cronで深夜自動実行後の翌朝等）
- **保管期間**: 最低3ヶ月分を保持
- **確認**: 月1回はバックアップファイルが正常に開けるか確認

**運用カレンダー例**:
```
毎週月曜日: サーバから最新バックアップを外部へコピー
毎月第1月曜日: 外部バックアップの整合性確認（tar -tzf でファイルリスト確認）
```

**⚠️ 重大な警告**:
外部バックアップを怠ると、サーバ障害時に**すべてのデータと設定が失われます**

---

## 13. ログ管理（確認コマンド・ログローテーション）

### 📊 ログ確認コマンド

```bash
# システムログ（nginx起動・エラー等）※ Ctrl + Cで強制終了
sudo journalctl -u nginx --since today

# アクセスログ（直近10行のログを確認）
tail -n 10 /var/www/myproject/logs/access.log

# エラーログ（直近10行のログを確認）※ エラーの記録が無い場合は、何も表示されない
tail -n 10 /var/www/myproject/logs/error.log

# fail2ban ログ
sudo tail -n 10 /var/log/fail2ban.log

# 自動更新ログ
sudo tail -n 10 /var/log/unattended-upgrades/unattended-upgrades.log

# バックアップログ
sudo tail -n 10 /var/log/server_backup.log
```

### 🗂️ ログローテーション設定

```bash
# ログローテーション設定ファイル作成
sudo nano /etc/logrotate.d/myproject
```

```conf
/var/www/myproject/logs/*.log {
    daily
    missingok
    rotate 90
    compress
    delaycompress
    notifempty
    create 0640 webuser www-data
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

---

## 14. HTTPS（Let's Encrypt）設定

### ⚠️ 重要：サーバ切り替え時の注意

**新規サーバ構築中の場合**:
※ 手順14.は、実際にインターネットに接続した状況で実施する必要があります。

1. 192.168.1.XXXでの構築・テストを完了
2. 現行サーバ（XXX.XXX.XXX.XXX）をシャットダウン
3. 新規サーバのNIC設定をXXX.XXX.XXX.XXXに変更
4. **IPアドレス変更後**に本セクションのcertbot実行を行う

**理由**: Let's Encryptは、DNSで設定されたIPアドレス（XXX.XXX.XXX.XXX）からHTTP応答を確認するため、IP変更前には証明書取得できません。

### 📦 Certbotインストール

```bash
# インストール確認
if dpkg -l | grep -q certbot; then
    echo "ℹ️ certbot は既にインストール済みです"
else
    sudo apt install -y certbot python3-certbot-nginx
    echo "✅ certbot をインストールしました"
fi
```
### 🔒 SSL証明書取得

**⚠️ 事前確認必須**:
1. **`server_name` を実際のドメイン名に変更済みか確認**
   ```bash
   # 設定ファイル確認
   grep "server_name" /etc/nginx/sites-available/myproject
   # → "server_name _;" になっている場合は変更が必要
   ```

2. DNSレコードがグローバルIPを正しく指しているか確認
   ```bash
   # ドメイン名解決確認
   nslookup example.com
   # または
   dig example.com
   ```
3. **本サーバのIPアドレスがXXX.XXX.XXX.XXXであることを確認**
   ```bash
   ip addr show
   # または
   hostname -I
   ```
```bash

# ドメイン名を指定して実行（推奨）
sudo certbot --nginx -d example.com -d www.example.com

# 対話形式で実行（ドメイン名を自動検出）
# sudo certbot --nginx

# 自動更新テスト
sudo certbot renew --dry-run
```

**⚠️ 注意事項**:
- 事前にドメインのDNS設定が必要
- グローバルIPとドメインの紐付けを確認
- Let's Encryptは80番ポートでの検証が必要
- **`server_name _;` のままだと証明書取得に失敗します**

---

## 15. 完了チェックリスト

```bash
# 以下のコマンドで各項目を確認
```

- [ ] **Windows側SSH鍵ペア生成**: `C:\Users\[ユーザ名]\.ssh\server_id_rsa` と `server_id_rsa.pub` 存在確認
- [ ] **初回パスワード接続**: TeraTerm経由で接続成功
- [ ] **公開鍵サーバ登録**: `cat ~/.ssh/authorized_keys` で公開鍵確認
- [ ] **鍵認証接続**: パスワードなしでSSH接続成功
- [ ] **TeraTerm自動ログインスクリプト**: 動作確認済み
- [ ] **SSHセキュリティ**: `sudo sshd -t` でエラーなし
- [ ] **ファイアウォール**: `sudo ufw status` で適切なルール確認
- [ ] **fail2ban**: `sudo fail2ban-client status` で稼働確認
- [ ] **OS自動更新**: `systemctl status unattended-upgrades` で有効確認
- [ ] **nginx自動起動**: `sudo systemctl is-enabled nginx` が `enabled`
- [ ] **HTTP到達**: ブラウザで `http://[サーバのIPアドレス]/` にアクセスし「Work in Progress」ページ表示確認
- [ ] **ディレクトリ権限**: `ls -la /var/www/myproject` で適切な権限確認
- [ ] **ログ出力**: `/var/www/myproject/logs/` にログファイル生成確認
- [ ] **バックアップスクリプト**: `sudo /var/www/myproject/scripts/backup.sh` で動作確認
- [ ] **cron設定**: `sudo crontab -l` でバックアップジョブ確認
- [ ] **HTTPS証明書取得**（グローバル公開後）: `sudo certbot certificates` で証明書確認
- [ ] **HTTPS動作確認**（グローバル公開後）: ブラウザで `https://example.com/` にアクセスし表示を確認

---

## 16. 最終確認手順（10項目の詳細検証）

**目的**: すべての設定が正しく完了していることを検証する

### 📋 検証コマンド一覧

以下のコマンドを実行して、各設定項目が正しく完了していることを確認してください。

#### 1. ufwルール詳細確認

```bash
sudo ufw status verbose
```

**期待結果**:
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
[custom-port]/tcp          ALLOW IN    Anywhere                   # SSH
80/tcp                     ALLOW IN    Anywhere                   # HTTP
443/tcp                    ALLOW IN    Anywhere                   # HTTPS
[custom-port]/tcp (v6)     ALLOW IN    Anywhere (v6)              # SSH
80/tcp (v6)                ALLOW IN    Anywhere (v6)              # HTTP
443/tcp (v6)               ALLOW IN    Anywhere (v6)              # HTTPS
```

#### 2. fail2ban設定確認

```bash
# 設定ファイル確認
cat /etc/fail2ban/jail.local

# fail2ban稼働状況確認
sudo fail2ban-client status

# sshd jail詳細確認
sudo fail2ban-client status sshd
```

**期待結果**:
- jail.local: bantime=3600, findtime=600, maxretry=3
- jail.local: ignoreip: 192.168.1.0/24 が設定されている
- fail2ban-client status: 
   Status
   └ Number of jail: 1
   └ Jail list:      sshd
- fail2ban-client status sshd: 
   Status for the jail: sshd
   |- Filter
   |  |- Currently failed: 0
   |  |- Total failed:     0
   |  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
   `- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:

#### 3. nginxシンボリックリンク確認

```bash
ls -la /etc/nginx/sites-enabled/

sudo nginx -t
```

**期待結果**:
```
lrwxrwxrwx 1 root root 31 ... myproject -> /etc/nginx/sites-available/myproject
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

#### 4. nginx自動起動確認

```bash
sudo systemctl is-enabled nginx

sudo systemctl status nginx --no-pager
```

**期待結果**:
- is-enabled: `enabled`
- status: `active (running)`

#### 5. バックアップファイル保存先確認

```bash
cat /var/www/myproject/scripts/backup.sh | grep BACKUP_DIR  

ls -lht /var/backups/ | head

```

**期待結果**:
- BACKUP_DIR="/var/backups"
- /var/backups/ に server_backup_*.tar.gz ファイルが存在

#### 6. cron設定確認

```bash
sudo crontab -l
```

**期待結果**:
`0 3 * * * /var/www/myproject/scripts/backup.sh >> /var/log/server_backup.log 2>&1`

#### 7. ログローテーション設定確認

```bash
cat /etc/logrotate.d/myproject
```

**期待結果**:
```
/var/www/myproject/logs/*.log {
    daily
    missingok
    rotate 90
    compress
    delaycompress
    notifempty
    create 0640 webuser www-data
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

#### 8. ディレクトリ権限総合確認

```bash
ls -la /var/www/myproject/
ls -la /var/www/myproject/public/
ls -la /var/www/myproject/data/
ls -la /var/www/myproject/logs/
ls -la /var/www/myproject/scripts/
```

**期待結果**:
- すべてのディレクトリ: drwxr-x--- (750) webuser www-data
- ファイル権限: 適切に設定されている

#### 9. SSH設定確認

```bash
sudo grep -E '^Port|^PermitRootLogin|^PasswordAuthentication' /etc/ssh/sshd_config
```

**期待結果**:
```
Port [custom-port]  # 例: 22222
PermitRootLogin no
```

**注意**: PasswordAuthenticationの行が表示されない場合は、コメントアウトされているため、デフォルト設定が適用されています。

#### 10. OS自動更新設定確認

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades

systemctl status unattended-upgrades --no-pager
```

**期待結果**:
- Update-Package-Lists "1"
- Download-Upgradeable-Packages "1"
- AutocleanInterval "7"
- Unattended-Upgrade "1"
- unattended-upgrades: active (running)

### ✅ 検証完了の判断基準

すべての検証コマンドで期待結果が得られた場合、基盤設定は正常に完了しています。

### ⚠️ 期待結果と異なる場合

期待結果と異なる項目がある場合は、該当セクションに戻って設定を再確認してください。

---

## 17. トラブルシューティング

### 🚨 よくある問題と対処法

#### SSH接続できない（ロックアウト時の救出策）

**状況**: SSHポート変更やファイアウォール設定ミスで接続不可

**対処法**:
1. **コンソール経由でログイン**
   - 物理アクセス、またはVPS管理画面のコンソール機能を使用
   - rootまたはwebuserでログイン

2. **SSH設定の確認・復元**
   ```bash
     # コンソールからログ確認（"10 minutes ago"の時間は適宜変更可能）
     sudo journalctl -u ssh --since "10 minutes ago"

     # ログを閉じる際は、"q"を入力
   
     # ポート確認
     sudo ss -tlnp | grep ssh
   
     # 設定ファイルをバックアップから復元
     sudo cp /etc/ssh/sshd_config.backup.YYYYMMDD /etc/ssh/sshd_config
     sudo systemctl restart ssh
   ```

3. **ファイアウォール確認・一時無効化**
   **注意**：TeraTermからの接続が不可となった場合は、webサーバに直接ログインして作業してください
   ```bash
     # ファイアウォール状態確認
     sudo ufw status
   
     # 緊急時：一時的に無効化
     sudo ufw disable
   
     # SSH接続確認後、再設定
     sudo ufw enable
   ```

4. **fail2banで自己BANされた場合**
   ```bash
     # BAN状態確認
     sudo fail2ban-client status sshd
   
     # 自分のIPをアンバン
     sudo fail2ban-client set sshd unbanip YOUR_IP_ADDRESS
   ```

#### 公開鍵認証が失敗する

**確認ポイント**:
```bash
  # サーバ側での確認（パスワード接続で再接続）
  # 1. authorized_keysの内容確認
  cat ~/.ssh/authorized_keys
  # → Windows側で生成した公開鍵が正しく登録されているか

  # 2. 権限確認
  ls -la ~/.ssh/
  # → .ssh: drwx------ (700)
  # → authorized_keys: -rw------- (600)

  # 3. SSHログ確認（終了時は、`Ctrl + C`を入力）
  sudo tail -f /var/log/auth.log
  # → 接続試行時のエラーメッセージを確認

  # Windows側での確認
  # 1. 秘密鍵の存在確認
  dir C:\Users\[ユーザ名]\.ssh\server_id_rsa

  # 2. TeraTerm設定確認
  # - 秘密鍵のパスが正しいか
  # - ユーザ名が正しいか
```

#### 公開鍵が正しく登録できない

**対処法**:
```bash
1. Windows側で公開鍵の内容を再度確認:
   type C:\Users\[ユーザ名]\.ssh\server_id_rsa.pub
   → 全選択してコピー

2. TeraTermでパスワード接続

3. 以下のコマンドで再登録:
   nano ~/.ssh/authorized_keys
   → コピーした公開鍵を貼り付け
   → Ctrl+O → Enter → Ctrl+X
   
   chmod 600 ~/.ssh/authorized_keys

   # 改行コードをLinux形式に変換（念のため）
   dos2unix ~/.ssh/authorized_keys
   # 必要な場合：dos2unix（改行コード変換ツール）インストール方法
   # sudo apt update
   # sudo apt install dos2unix -y

4. 再度鍵認証接続テストを実施
```

#### nginxが起動しない
```bash
# 詳細ログ確認
sudo journalctl -xe -u nginx

# 設定ファイル検証
sudo nginx -t

# ポート競合確認
sudo ss -tlnp | grep :80
```

#### パーミッションエラー
```bash
# 所有権再設定
sudo chown -R webuser:www-data /var/www/myproject

# 権限再設定（全て750で統一）
sudo chmod -R 750 /var/www/myproject
```

#### TeraTerm自動ログインスクリプトが動作しない
```bash
# Windows側での確認ポイント
1. スクリプト内のパラメータ確認:
   - HOST, PORT, USERNAME が正しいか
   - KEY_FILE_PATH のパスが正しいか（server_id_rsa に統一）
   - LOG_DIR_PATH のパスが正しいか

2. 秘密鍵ファイルの確認:
   - ファイルが存在するか
   - パスに日本語や空白文字が含まれていないか

3. ログディレクトリの確認:
   - ディレクトリが存在するか
   - 書き込み権限があるか

4. TeraTermのバージョン確認:
   - 古いバージョンの場合、マクロ機能が制限されている可能性
```

#### 自動更新が動作しない
```bash
# unattended-upgrades状態確認
sudo systemctl status unattended-upgrades

# 設定ファイル確認
sudo cat /etc/apt/apt.conf.d/50unattended-upgrades

# 手動実行テスト
sudo unattended-upgrades --dry-run --debug
```

#### バックアップが失敗する
```bash
# ディレクトリ確認
ls -la /var/backups/

# 権限確認
ls -la /var/www/myproject/scripts/backup.sh

# 手動実行してエラー確認
sudo /var/www/myproject/scripts/backup.sh
```

---

## 18. 位置づけと次のステップ

### 📍 本手順書の対象範囲

- 初期ユーザ作成とタイムゾーン設定
- **クライアント側**SSH鍵認証設定（Windows/TeraTerm環境）
- SSHセキュリティ強化（ポート変更、ログイン制限）
- TeraTerm自動ログインスクリプト
- ファイアウォール設定（ufw）
- 侵入防止設定（fail2ban）
- OS自動更新設定
- Webサーバ（nginx）設定とディレクトリ構造
- HTTPS対応（Let's Encrypt）
- ログローテーション設定
- バックアップ戦略（スクリプト、cron）
- よくある問題の対処法

**対象外**:
- Ubuntu Server OSのインストール手順
- アプリケーション実装（HTML/CSS/JavaScript）
- データベース設定
- CI/CDパイプライン

### ➡️ 次の工程

フロントエンド設計/実装（HTML/CSS/JS）

---

