---
title: "Fedora40 で HTTPS のウェブサーバを動かす"
emoji: "🚁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

続きです。

<https://zenn.dev/asterisk9101/articles/fedora40server-1>

## ファイアウォールの許可

`http` と `https` のサービスを許可します。

```bash
firewall-cmd --add-service=http --permanent
firewall-cmd --add-service=https --permanent
firewall-cmd --reload
```

## nginx のインストール

起動して動作確認をします。

```bash
dnf -y install nginx
systemctl enable --now nginx

curl http://localhost 2>&1 | grep '<title>'
# => <title>Test Page for the HTTP Server on Fedora</title>
```

## certbot を使って証明書を入手

予めドメインを取得しておき、自宅のグローバル IP を設定しておきます。

```bash
read -p 'DOMAIN?> ' DOMAIN
```

DNSに登録したサーバ名を入力します。

```bash
read -p 'SERVER_NAME?>' SERVER_NAME
```

また、証明書が期限切れになりそうなときに、Let's Encrypt から通知を受けるメールアドレスを設定します。

```bash
read -p 'EMAIL?>' EMAIL
```

certbot で証明書を取得します。

```bash
dnf -y install certbot python3-certbot-nginx
certbot certonly --nginx -d $DOMAIN -m $EMAIL --agree-tos -n
```

## nginx の TLS を有効化

`nginx` の TLS を有効化して、certbot で入手した証明書を参照するように設定します。

```bash
FQDN="$SERVER_NAME.$DOMAIN"
RANGE1='Settings for a TLS enabled server'
RANGE2='^}'
REPLACEMENT1="_;"
REPLACEMENT2="/etc/pki/nginx/server.crt"
REPLACEMENT3="/etc/pki/nginx/private/server.key"
sed -i.bak -r \
    -e "/$RANGE1/,/$RANGE2/s@$REPLACEMENT1@$FQDN;@" \
    -e "/$RANGE1/,/$RANGE2/s@$REPLACEMENT2@/etc/letsencrypt/live/$DOMAIN/fullchain.pem@" \
    -e "/$RANGE1/,/$RANGE2/s@$REPLACEMENT3@/etc/letsencrypt/live/$DOMAIN/privkey.pem@" \
    -e "/$RANGE1/,/$RANGE2/s@^#@@" \
    -e "/$RANGE1/d" \
    /etc/nginx/nginx.conf
```

`nginx.conf` の構文をチェックして問題無ければ、`nginx` を再起動します。

```bash
nginx -t
systemctl restart nginx

curl -k https://localhost
# => <title>Test Page for the HTTP Server on Fedora</title>
```

## 証明書の自動更新

証明書を自動的に更新するよう設定します。

```bash
systemctl enable --now certbot-renew.timer
# systemctl daemon-reload でも良いかも

systemctl list-timers certbot-renew.timer

cat /etc/sysconfig/certbot | grep POST_HOOK
# => POST_HOOK="--post-hook 'systemctl restart nginx'"
```

以上
