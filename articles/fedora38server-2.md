---
title: "Fedora38 検証サーバをFreeIPAサーバにするときのメモ"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora38, FreeIPA, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

<https://zenn.dev/asterisk9101/articles/fedora38server-1>

## Firewalld の設定

freeipa 用の定義があるので利用します。

```bash
firewall-cmd --add-service={freeipa-4,dns,ntp} --permanent
firewall-cmd --reload
```

## FreeIPA サーバのインストール

FreeIPA-server をインストールします。ホームルーターをフォワーダーに設定しています。DNSSEC は付いてないので検証をオフにします。

```bash
dnf -y install freeipa-server freeipa-server-dns

ADMINPASS=P@ssw0rd
DOMAIN=$(hostname -d)
FORWARDER=192.168.1.1
ipa-server-install --setup-dns -a $ADMINPASS -p $ADMINPASS -r $DOMAIN --no-ntp --mkhomedir --no-dnssec-validation --forwarder=$FORWARDER -U
```

## 稼働確認

サービスのステータスを確認します。

```bash
ipactl status
```

認証チケットが取得ができるか確認します。

```bash
kinit admin
klist
```

ついでにデフォルトのログインシェルを `bash` に変更しておきます。ただし、Fedora38 の /bin/sh は /bin/bash へのリンクになっているので、同じ OS を使う限り意味はありません。

```bash
ipa config-mod --defaultshell=/bin/bash
```

## 停止と再起動の確認

停止と再起動の方法を確認します。ipactl でサービスの停止や起動は推奨されていないようです。

```bash
systemctl stop ipa
ipactl status
systemctl start ipa
ipactl status

systemctl restart ipa
ipactl status
```

以上
