# Hacking Skills Index

習得スキルの全体マップ。各ノートへのリンク集。

---

## スキルカテゴリ

### 情報収集 / Reconnaissance
- [x] Nmap → [[Skills/Nmap]]
- [ ] OSINT
- [ ] Subdomain Enumeration
- [x] Directory Busting (Gobuster, Feroxbuster)

### Webアタック / Web Attacks
- [ ] SQL Injection → [[Skills/SQL-Injection]]
- [ ] XSS (Cross-Site Scripting)
- [ ] File Inclusion (LFI/RFI)
- [ ] SSRF
- [ ] IDOR
- [ ] Command Injection
- [ ] XXE

### ネットワーク / Network
- [ ] Port Scanning
- [ ] Service Enumeration
- [ ] SMB Enumeration
- [ ] FTP/SSH/Telnet

### 権限昇格 / Privilege Escalation
- [x] Linux PrivEsc → [[Skills/Linux-PrivEsc]]
- [ ] Windows PrivEsc → [[Skills/Windows-PrivEsc]]
- [ ] SUID/SGID
- [ ] Sudo Misconfigurations
- [ ] Cron Jobs

### パスワード攻撃 / Password Attacks
- [ ] Hashcat
- [ ] John the Ripper
- [ ] Hydra (Brute Force)

### エクスプロイト / Exploitation
- [ ] Metasploit
- [ ] Buffer Overflow（基礎）
- [ ] CVE調査・活用

### ポストエクスプロイト / Post-Exploitation
- [ ] シェル安定化
- [ ] ピボッティング
- [ ] データ抽出

### ツール
- [[Tools/Nmap-Cheatsheet]]
- [[Tools/Burp-Suite]]
- [[Tools/Gobuster]]
- [[Tools/Metasploit]]

---

## 攻略済みBox

### TryHackMe
| Box名 | 難易度 | 習得スキル | 日付 | メモ |
|-------|-------|-----------|------|------|
| [[Boxes/TryHackMe/Lookup\|Lookup]] | Easy | ffuf列挙, elFinder RCE, PATH Hijacking, GTFOBins(look) | 2026-01-08 | SUID+PATH Hijackingがポイント |
| [[Boxes/TryHackMe/WebHacking-Using-cURL-AoC2025-W8\|Web Hacking Using cURL (AoC2025 W8)]] | Easy | curl GET/POST, Cookie操作, bash BF, UA偽装 | 2026-01-13 | curl操作の基礎練習 |

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

*最終更新: 2026-03-19*
