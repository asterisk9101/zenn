---
title: "Fedora40 サーバを FreeIPA クライアントにするときのメモ"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora38, FreeIPA, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

続きです。

<https://zenn.dev/asterisk9101/articles/fedora40server-1>

Free IPA サーバについては、以下を参照。

<https://zenn.dev/asterisk9101/articles/fedora38server-2>

`192.168.1.24` `24-fedora40.localdomain.intra` が FreeIPA サーバであるとします。

## FreeIPA ドメインへの参加

Free IPA ドメインの管理者のパスワードを入力します。

```bash
read -p 'DOMAIN ADMIN PASSWORD?>' PW
```

```bash
nmcli con mod ens18 ipv4.dns 192.168.1.24
systemctl restart NetworkManager

DOMAIN=$(hostname -d)
IPA=24-fedora40.$DOMAIN
ID=admin
ipa-client-install --server=$IPA --domain $DOMAIN -p $ID -w $PW --mkhomedir -U
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
