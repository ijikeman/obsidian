# Hacking Skills Index

習得スキルの全体マップ。各ノートへのリンク集。

---

## スキルカテゴリ

### 情報収集 / Reconnaissance
- [x] Nmap → [[Skills/Nmap]]
- [x] Rustscan (高速ポートスキャン) ← md2pdf
- [x] Directory Busting (Gobuster, Feroxbuster) → [[Tools/Gobuster]]
- [x] ffuf (ユーザ名列挙・パスワードBF) ← Lookup
- [x] SMB Enumeration (enum4linux) ← Basic Pentesting
- [x] wpscan (WordPress調査) → [[Tools/wpscan]] ← Bricks Heist
- [ ] OSINT
- [ ] Subdomain Enumeration

### Webアタック / Web Attacks
- [x] curl を使ったWeb操作 (GET/POST/Cookie/UA偽装) → [[Tools/curl-cheatsheet]] ← AoC2025 W8
- [ ] SQL Injection → [[Skills/SQL-Injection]]
- [ ] XSS (Cross-Site Scripting)
- [ ] File Inclusion (LFI/RFI)
- [x] SSRF (iframeインジェクション, PDF変換悪用) ← md2pdf
- [ ] IDOR
- [ ] Command Injection
- [ ] XXE
- [ ] WordPress Exploitation (Bricks Builder CVE-2024-25600) ← Bricks Heist
- [x] Tomcat Manager WAR Upload RCE ← bsidesgtthompson
- [x] JWT alg:none 攻撃 (role/credits改ざん) ← TryHeartMe

### ネットワーク / Network
- [x] Port Scanning
- [x] Service Enumeration
- [x] SMB Enumeration (enum4linux, smbclient)
- [ ] FTP/SSH/Telnet
- [ ] Active Directory

### 権限昇格 / Privilege Escalation
- [x] Linux PrivEsc → [[Skills/Linux-PrivEsc]]
- [x] SUID/SGID 悪用 ← Lookup
- [x] PATH Hijacking ← Lookup
- [x] GTFOBins 活用 (sudo look 等) ← Lookup
- [x] Cron Job Hijacking (書き込み可能スクリプト) ← bsidesgtthompson → [[Skills/Cron-Job-Hijacking]]
- [ ] Windows PrivEsc → [[Skills/Windows-PrivEsc]]
- [ ] Sudo Misconfigurations
- [ ] Cron Jobs

### パスワード攻撃 / Password Attacks
- [x] Hydra (SSH ブルートフォース) ← Lookup, Basic Pentesting
- [x] John the Ripper → [[Tools/John-the-Ripper]] ← Basic Pentesting
- [x] ssh2john (SSH鍵パスフレーズ解析) ← Basic Pentesting
- [ ] Hashcat
- [ ] パスワードスプレー

### エクスプロイト / Exploitation
- [x] Metasploit (RCE モジュール活用) ← Lookup, Bricks Heist, bsidesgtthompson
- [x] CVE調査・活用 (searchsploit, PoC実行) ← Lookup, Bricks Heist
- [x] Ghostcat (CVE-2020-1938) AJP13経由ファイル読み取り ← bsidesgtthompson
- [x] SSH 秘密鍵の悪用 (パーミッション不備) ← Basic Pentesting
- [ ] Buffer Overflow（基礎）
- [ ] Active Directory 攻撃

### ポストエクスプロイト / Post-Exploitation
- [x] リバースシェル (nc, PHP, bash) ← Lookup, Bricks Heist → [[Skills/Reverse-Shell]]
- [x] シェル安定化・TTYアップグレード (python3 pty, stty raw) ← Lookup → [[Skills/Reverse-Shell]]
- [x] プロセス・サービス調査 (systemctl) ← Bricks Heist
- [ ] ピボッティング
- [ ] データ抽出

### フォレンジック / マルウェア調査
- [x] マイニングマルウェア調査 (nm-inet-dialog) ← Bricks Heist
- [x] エンコード解析 (CyberChef: Hex→Base64) ← Bricks Heist
- [ ] メモリフォレンジック
- [ ] ログ解析

### 脅威インテリジェンス / Threat Intel
- [x] ブロックチェーン調査 (blockchain.com) ← Bricks Heist
- [x] OFAC サンクションリスト調査 ← Bricks Heist
- [ ] MITRE ATT&CK フレームワーク
- [ ] IOC 分析

### ツール
- [[Tools/Nmap-Cheatsheet]]
- [[Tools/Gobuster]]
- [[Tools/ffuf]]
- [[Tools/curl-cheatsheet]]
- [[Tools/John-the-Ripper]]
- [[Tools/wpscan]]
- [[Tools/Burp-Suite]]
- [[Tools/Metasploit]]
- [[Tools/Apache-Tomcat]]
- [[Tools/Pentest-Resources]] ← SecLists / GTFOBins / linpeas 等リソース集

---

## 攻略済みBox

### TryHackMe
| Box名 | 難易度 | 習得スキル | 日付 | メモ |
|-------|-------|-----------|------|------|
| [[Boxes/TryHackMe/Lookup\|Lookup]] | Easy | ffuf列挙, elFinder RCE, PATH Hijacking, GTFOBins(look) | 2026-01-08 | SUID+PATH Hijackingがポイント |
| [[Boxes/TryHackMe/WebHacking-Using-cURL-AoC2025-W8\|Web Hacking Using cURL (AoC2025 W8)]] | Easy | curl GET/POST, Cookie操作, bash BF, UA偽装 | 2026-01-13 | curl操作の基礎練習 |
| [[Boxes/TryHackMe/Basic-Pentesting\|Basic Pentesting]] | Easy | enum4linux, Hydra SSH BF, ssh2john, John the Ripper | 2026-01-15 | SSH鍵のパーミッション不備 + パスフレーズ解析 |
| [[Boxes/TryHackMe/TryHack3M-Bricks-Heist\|TryHack3M: Bricks Heist]] | Easy | wpscan, CVE-2024-25600 RCE, マイニングマルウェア調査, CyberChef, Threat Intel | 2026-01-15 | LockBit関連BTC追跡まで |
| [[Boxes/TryHackMe/md2pdf\|md2pdf]] | Easy | Rustscan, Gobuster, SSRF (iframeインジェクション) | 2026-03-27 | PDF変換機能を悪用した内部API(localhost:5000)へのSSRF |
| [[Boxes/TryHackMe/bsidesgtthompson\|bsidesgtthompson]] | Easy | Ghostcat (CVE-2020-1938), Tomcat Manager WAR Upload RCE, Cron Job Hijacking | 2026-03-27 | 401ページのヒントから認証情報取得 → WAR RCE → cronジョブ乗っ取りでroot |
| [[Boxes/TryHackMe/tryheartme\|TryHeartMe]] | Easy | JWT alg:none攻撃, JWTペイロード改ざん (role/credits), 隠し商品アクセス | 2026-03-30 | 登録レスポンスのSet-CookieにJWT → alg:none改ざんでadmin権限取得 |

### HackTheBox
| Box名 | 難易度 | 習得スキル | 日付 | メモ |
|-------|-------|-----------|------|------|
|       |       |           |      |      |

---

## 学習ロードマップ

```
初心者
  └─ Reconnaissance (Nmap, Gobuster)
      └─ Web Attacks (SQLi, XSS)
          └─ PrivEsc (Linux → Windows)
              └─ Active Directory
                  └─ Red Team Operations
```

---

*最終更新: 2026-03-30 (TryHeartMe 完全攻略)*
