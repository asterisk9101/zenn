---
title: "Fedora41 サーバで PostgreSQL を使うときのメモ"
emoji: "🐘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, PostgreSQL, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

続きです。

<https://zenn.dev/asterisk9101/articles/fedora41server-1>

## インストール

とりあえずインストールします。

```bash
dnf install -y postgresql-server
```

最初のデータベースを作成して、サービスが起動することを確認します。

```bash
/usr/bin/postgresql-setup --initdb
systemctl enable --now postgresql
```

## リモートホストからの接続許可

必要に応じて絞りますが、全許可します。

```bash
# ファイアウォールの許可
firewall-cmd --add-service=postgresql --permanent
firewall-cmd --reload

# リスナーの許可
sed -i.bak "/listen_addresses/a listen_addresses = '*'" /var/lib/pgsql/data/postgresql.conf
diff /var/lib/pgsql/data/postgresql.conf{,.bak}

# 認証方式の許可
sed -i.bak '$a host all all 0.0.0.0/0 scram-sha-256' /var/lib/pgsql/data/pg_hba.conf
diff /var/lib/pgsql/data/pg_hba.conf{,.bak}

# サービスの再起動
systemctl restart postgresql
```

## アプリケーション用データベースの作成

パッケージインストール時点で OS ユーザー `postgres` が作成されているので、スイッチします。

```bash
su - postgres
```

アプリケーション用のユーザーをパスワードの有効期限を無期限で作成します。

```bash
read -p 'User Name?> ' ID
```

```bash
read -s -p 'Password?> ' PW
```

```bash
psql -c "CREATE ROLE $ID LOGIN ENCRYPTED PASSWORD '$PW' VALID UNTIL 'infinity';"

# 確認
psql -c "\du"
```

アプリケーション用のデータベースを作成します。

```bash
read -p 'database name?> ' DB
```

```bash
createdb $DB -O $ID

# 確認
psql -l
```

以上
