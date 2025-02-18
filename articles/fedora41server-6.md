---
title: "Fedora41 サーバで Minio Server を使うときのメモ"
emoji: "🐦️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, MinIO, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

基本は公式の通りです。

<https://min.io/docs/minio/linux/index.html>

## インストール

ダウンロードしてインストールします。

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio-20250207232109.0.0-1.x86_64.rpm -O minio.rpm
dnf -y install minio.rpm
```

## サービス登録

`systemd` のユニットファイルを作ります。

```bash
cat << EOF > /usr/lib/systemd/system/minio.service
[Unit]
Description=MinIO
Documentation=https://min.io/docs/minio/linux/index.html
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local
User=minio-user
Group=minio-user
ProtectProc=invisible
EnvironmentFile=-/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"\${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/bin/minio server \$MINIO_OPTS \$MINIO_VOLUMES
Restart=always
LimitNOFILE=65536
TasksMax=infinity
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target
EOF
```

サービスを実行するユーザーを作ります。

```bash
# -r --system
groupadd -r minio-user

# -M --no-create-home
# -r --system
# -g --gid
useradd -M -r -g minio-user minio-user

mkdir /mnt/data
chown minio-user:minio-user /mnt/data
```

`MinIO` にログインするユーザーを設定します。

```bash
read -s -p 'password?> ' PW
```

```bash
cat << EOF >/etc/default/minio
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=$PW
MINIO_VOLUMES="/mnt/data"
MINIO_OPTS="--console-address :9001"
EOF
```

ユニットファイルを読み込んでから起動します。

```bash
systemctl daemon-reload
systemctl start minio

# 確認
systemctl status minio
```

## ファイアウォールの開放

使用するポートを開けます。

```bash
firewall-cmd --add-port=9000/tcp --permanent
firewall-cmd --add-port=9001/tcp --permanent
firewall-cmd --reload
```

## クライアントのインストール

`mc` をインストールします。マニュアル通りです。

<https://min.io/docs/minio/linux/reference/minio-mc.html#minio-client>

```bash
curl https://dl.min.io/client/mc/release/linux-amd64/mc --create-dirs -o $HOME/bin/mc
chmod +x $HOME/bin/mc

# 確認
mc --help
```

WebUI で発行したキーをクライアントに設定する。

```bash
read -s -p 'ACCESS KEY?> ' ACCESS_KEY
```

```bash
read -s -p 'SECRET KEY?> ' SECRET_KEY
```

```bash
SERVER=$(echo http://$(hostname -I):9000 | tr -d ' ')
mc alias set myminio $SERVER $ACCESS_KEY $SECRET_KEY

# 確認(consoleAdmin 権限が必要)
mc admin info myminio
```

認証情報は `.mc` ディレクトリの `config.json` に保存されるようです。

```bash
ls -l .mc
# certs  config.json  config.json.old  share
```

以上
