---
title: "Fedora36 検証サーバでPostgreSQLを使えるようにするときのメモ"
emoji: "🐏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora36, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提の確認

`Podman` を使います。`dnf install podman` でインストールしておきます。

## 参考資料

<https://hub.docker.com/_/postgres/>

## Podman のインストール

検証なので Podman を使います。以下のコマンドでインストールします。

```bash
dnf -y install podman
```

## Firewalld の設定

`postgres` 用の定義があるので利用します。

```bash
firewall-cmd --add-service=postgresql --permanent
firewall-cmd --reload
```

## コンテナの構成

`PostgreSQL` を実行します。

```bash
podman run -d --name postgres \
    -p 5432:5432 \
    -e POSTGRES_PASSWORD=mysecretpassword \
    postgres
```

データを保存する必要があるときは、`-v` でボリュームマウントします。[:Z フラグ](https://docs.docker.jp/engine/userguide/dockervolumes.html)を付けておきます。

```bash
mkdir -p /custom/mount
podman run -d --name postgres \
    -p 5432:5432 \
    -e POSTGRES_PASSWORD=mysecretpassword \
    -e PGDATA=/var/lib/postgresql/data/pgdata \
    -v /custom/mount:/var/lib/postgresql/data:Z \
    postgres
```

## 稼働確認

コンテナに入ってデータベースが起動していることを確認します。

```bash
podman exec -it postgres bash
```

コンテナに入ったら、データベースに接続します。

```bash
psql -U postgres
```

以上
