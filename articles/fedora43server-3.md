---
title: "Fedora43 サーバを FreeIPA クライアントにするときのメモ"
emoji: "👒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, FreeIPA, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

続きです。

<https://zenn.dev/asterisk9101/articles/fedora43server-1>
<https://zenn.dev/asterisk9101/articles/fedora43server-2>

## FreeIPA ドメインへの参加

`192.168.1.24` `24-fedora43.localdomain.intra` が FreeIPA サーバであるとします。

```bash
IPA_SERVER=192.168.1.24
IPA_SERVER_FQDN=24-fedora43.localdomain.intra
```

Free IPA ドメインの管理者のパスワードを入力します。

```bash
read -s -p 'DOMAIN ADMIN PASSWORD?>' PW
```

```bash
nmcli con mod ens18 ipv4.dns 192.168.1.24
systemctl restart NetworkManager

DOMAIN=$(hostname -d)
ID=admin
ipa-client-install --server=$IPA_SERVER_FQDN --domain $DOMAIN -p $ID -w $PW --mkhomedir -U
```

### FreeIPA ドメインへの参加に失敗するとき

検証サーバは潰して立ててを繰り返すものなので、潰したサーバの情報がドメインに残っているのかも知れません。

ドメインコントローラにて、以下のコマンドで昔の情報を削除してから、クライアント側で `ipa-client-install` を再実行します。

```bash
kinit admin
ipa host-del <FQDN> --updatedns
```

## DNSの設定

FreeIPA サーバが複数ある場合は DNS の設定を更新します。

```bash
DNS1=192.168.1.24
DNS2=192.168.1.28
nmcli con mod ens18 ipv4.dns $DNS1,$DNS2
systemctl restart NetworkManager
```

## FreeIPA ドメインから離脱

FreeIPA クライアントでドメインから離脱するには以下のコマンドを実行します。

```bash
ipa-client-install --uninstall
```

以上
