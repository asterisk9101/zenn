---
title: "Fedora40 でリバプロを動かす"
emoji: "🚁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

続きです。

<https://zenn.dev/asterisk9101/articles/fedora40server-5>

## SELinux の許可

```bash
setsebool -P httpd_can_network_connect on
```

## nginx.conf の修正

```bash
read -p 'FQDN?>' FQDN
```

```bash
read -p 'IP?>' IP
```

```bash
read -p 'PORT?>' PORT
```

## WebScoket 用の設定の追加

今回、背後のサーバは `websocket` を使うので、設定を追加する。

```bash
cat << EOF > /etc/nginx/conf.d/websocket.conf
map \$http_upgrade \$connection_upgrade {
    default upgrade;
    ''      close;
}
EOF
```

## プロキシする仮想サーバの設定

仮想サーバを設定する。`proxy_set_header` はわりと適当なので。。。

```bash
cat << EOF > /etc/nginx/conf.d/$FQDN.conf
server {
    listen       443 ssl;
    listen       [::]:443 ssl;
    http2        on;
    server_name  $FQDN;

    # このあたりは websocket 用の設定
    proxy_http_version 1.1;
    proxy_buffering off;
    proxy_set_header Host \$host;
    proxy_set_header Upgrade \$http_upgrade;
    proxy_set_header Connection \$connection_upgrade;

    # このあたりは後ろのサーバに接続元の情報を伝えるリバプロの定番設定
    proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Host \$host;
    proxy_set_header X-Real-IP \$remote_addr;
    proxy_set_header Orign http://\$host;

    location / {
    proxy_pass https://$IP:$PORT;
    }

    error_page 404 /404.html;
    location = /404.html {}

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {}

    include /etc/nginx/default.d/*.conf;
}
EOF
```

設定を確認して `nginx` を再起動する。

```bash
nginx -t
systemctl restart nginx
```

以上
