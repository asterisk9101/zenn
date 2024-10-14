---
title: "flarectl で DDNS する"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Cloudflare]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

Zero Trust Access Tunnel あれば要らんやんという話もありつつ `flarectl` を使ってみる感じで。

## flarectlをインストール

```bash
dnf -y install go
go install github.com/cloudflare/cloudflare-go/cmd/flarectl@latest
./go/bin/flarectl --version
# => flarectl version dev
```

## API Token を取得

Cloudflare に登録しているゾーンの編集権限のあるトークンを発行します。

<https://dash.cloudflare.com/profile/api-tokens>

一旦シェル変数に保存しておきます。

```bash
read -p 'CF_API_TOKEN?>' CF_API_TOKEN
```

## レコードIDを確認

ゾーンの名前を指定します。

```bash
read -p 'ZONE?>' ZONE
```

A レコードのレコードIDを確認します。

```bash
export CF_API_TOKEN
/root/go/bin/flarectl --json dns list --zone ${ZONE} --type A | jq -r .[].ID
```

レコードIDが一意であればシェル変数に入れておきます。

```bash
RECID=$(/root/go/bin/flarectl --json dns list --zone ${ZONE} --type A | jq -r .[].ID)
```

## 更新スクリプト作成

グローバルIPが変わったらDNSレコードを更新するシェルスクリプトを作ります。

```bash
cat << EOF > MyDDNS.sh
#!/bin/bash
IPFILE="GLOBAL_ADDRESS"
CF_API_TOKEN=${CF_API_TOKEN}
zone=${ZONE}
recname=${ZONE}
recid=${RECID}

GLOBAL_ADDRESS=\$(curl -s https://checkip.amazonaws.com)
BEFORE_ADDRESS=\$(cat \${IPFILE})

if [ "\$GLOBAL_ADDRESS" != "\$BEFORE_ADDRESS" ]; then
    echo "\${GLOBAL_ADDRESS}" > \${IPFILE}
    echo "\$(date '+%F %T') \${BEFORE_ADDRESS} > \${GLOBAL_ADDRESS}" > /root/MyDDNS.log
    export CF_API_TOKEN
    /root/go/bin/flarectl dns update --zone \${zone} --id \${recid} --name \${recname} --content \${GLOBAL_ADDRESS}
fi
EOF
```

ちゃんとスクリプトができたか確認して実行権限を付与します。`GLOBAL_ADDRESS` は、IP アドレスを覚えておくためのファイルです。

```bash
cat MyDDNS.sh
chmod +x MyDDNS.sh
touch GLOBAL_ADDRESS
```

## crontab に登録

5分毎に動くようにします。

```bash
echo '*/5 * * * * bash /root/MyDDNS.sh' >> /var/spool/cron/root
```

以上
