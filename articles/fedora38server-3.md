---
title: "Fedora38 検証サーバをFreeIPAクライアントにするときのメモ"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora38, FreeIPA, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

<https://zenn.dev/asterisk9101/articles/fedora38server-1>

## FreeIPA ドメインへの参加

`192.168.1.30` `30-fedora38.localdomain.intra` が FreeIPA サーバであるとして設定します。

```bash
nmcli con mod ens18 ipv4.dns 192.168.1.30
systemctl restart NetworkManager

DOMAIN=$(hostname -d)
IPA=31-fedora38.$DOMAIN
ID=admin
PW=P@ssw0rd
ipa-client-install --server=$IPA --domain $DOMAIN -p $ID -w $PW --mkhomedir -U
# ipa-client-install --server=$IPA --domain $DOMAIN -p $ID -w $PW --mkhomedir --force-join -U
```

## FreeIPA ドメインから離脱

FreeIPA クライアントでドメインから離脱するには以下のコマンドを実行します。

```bash
ipa-client-install --uninstall
```

FreeIPA サーバの DNS エントリを手動で削除します。

## 次

[FreeIPAレプリカにするとき](https://zenn.dev/asterisk9101/articles/fedora38server-4)

以上
