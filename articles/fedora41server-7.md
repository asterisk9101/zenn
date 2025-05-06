---
title: "Fedora41 サーバで Keycloak を使うときのメモ"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Keycloak, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 実行基盤の準備

`keycloak` はコンテナ環境での本番利用が想定されているようなので `podman` を使います。

```bash
dnf install -y podman podman-compose
```

## Keycloak の実行

`IaC` は以下のようにします。本番環境では外部データベースを利用しますが、検証なので内蔵データベースで済ませます。`podman-compose down` してもボリュームが消えないように永続化します。

```yml
# docker-compose.yml
version: "3.8"

services:
  keycloak:
    image: quay.io/keycloak/keycloak:26.2.0
    container_name: keycloak
    command: start-dev
    ports:
      - "8080:8080"
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin
    volumes:
      - keycloak_data:/opt/keycloak/data

volumes:
  keycloak_data:
```

```bash
podman-compose up -d

# 起動完了していることを確認する
podman-compose logs

# 以下のようなログが出力されていれば起動完了している
# Keycloak 26.2.0 on JVM (powered by Quarkus 3.20.0) started in 9.981s. Listening on: http://0.0.0.0:8080
```

## Keycloak の中の設定

レルムとクライアントを設定し、ユーザーも作ります。

```bash
CONTAINER=keycloak

# 1. Keycloak CLI を使ってログイン
podman exec ${CONTAINER} /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 \
  --realm master \
  --user admin \
  --password admin

# 2. レルム作成
REALM=myrealm
podman exec ${CONTAINER} /opt/keycloak/bin/kcadm.sh create realms \
  -s realm=${REALM} -s enabled=true

# 3. クライアント追加
# クライアント（アプリ）の IP アドレスを指定する必要がある
CLIENT=myclient
CLIENT_SECRET=my-very-secret
ALLOWLIST='["http://0.0.0.0:3000/*"]'
podman exec ${CONTAINER} /opt/keycloak/bin/kcadm.sh create clients -r myrealm \
  -s clientId=${CLIENT} -s publicClient=false -s secret=${CLIENT_SECRET} -s enabled=true -s redirectUris="${ALLOWLIST}"

# 4. ユーザー追加
USER=asterisk
EMAIL=asterisk@example.com
podman exec ${CONTAINER} /opt/keycloak/bin/kcadm.sh create users -r ${REALM} \
  -s username=${USER} -s email=${EMAIL} -s enabled=true

# 5. パスワード設定
PASS=asterisk

USER_ID=$(podman exec ${CONTAINER} /opt/keycloak/bin/kcadm.sh get users -r ${REALM} -q username=${USER} --fields id | jq -r .[0].id)

podman exec ${CONTAINER} /opt/keycloak/bin/kcadm.sh set-password -r ${REALM} --userid ${USER_ID} --new-password ${PASS}
```

## 管理コンソール

admin ユーザーは以下の URL にアクセスします。

```txt
http://<Keycloakサーバー>/
```

レルムに作成したユーザーは以下の URL にアクセスします。

```txt
http://<Keycloakサーバー>/realms/<realm名>/account/
```

便利
