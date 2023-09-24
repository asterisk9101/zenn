---
title: "Fedora38 検証サーバをFreeIPAレプリカにするときのメモ"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora38, FreeIPA, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

<https://zenn.dev/asterisk9101/articles/fedora38server-3>

## Firewalld の設定

freeipa 用の定義があるので利用します。

```bash
firewall-cmd --add-service={freeipa-4,dns,ntp} --permanent
firewall-cmd --reload
```

## FreeIPA レプリカの構成

FreeIPA-server をインストールします。ホームルーター(192.168.1.1)をフォワーダーに設定しています。DNSSEC は付いてないので検証をオフにします。

```bash
dnf -y install freeipa-server freeipa-server-dns

ipa-replica-install --setup-ca --setup-dns --no-dnssec-validation --forwarder=192.168.1.1 -p admin -w P@ssw0rd -U
```

## FreeIPAクライアントのDNS設定の更新

各 FreeIPA クライアントでは、DNS設定を更新する必要があります。

```bash
nmcli con mod ens18 ipv4.dns 192.168.1.30,192.168.1.31
systemctl restart NetworkManager
```

## 手動レプリケーション

通常はリアルタイムもしくは一定期間毎にレプリケーションが実行されるが、手動でレプリケーションを実行したい場合は以下のコマンドを使います。

```bash
ipa-replica-manage force-sync --from 30-fedora38.localdomain.intra
```

## FreeIPA ドメインの離脱

以下のコマンドで FreeIPA クライアント機能も削除されます。

```bash
ipa-server-install --uninstall
```

もしくは、以下のコマンドでトポロジーからサーバを削除します。

```bash
ipa server-del <servername>
```
