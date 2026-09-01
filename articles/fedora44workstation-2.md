---
title: "Fedora44 workstation を FreeIPA ドメインに参加させるときのメモ"
emoji: "👒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, memo]
published: true
---

お家の検証用の備忘録です。

## 前提

続きです。

<https://zenn.dev/asterisk9101/articles/fedora44workstation-1>

## IP固定化とホスト名の設定

ホスト名はIPアドレスを元に機械的に設定します（すぐ潰すことが多いので。。。）

```bash
read -p 'IP Address?> ' IP
```

```bash
read -p 'Domain?> ' Domain
```

```bash
GW=192.168.1.1
DNS=192.168.1.1
con="有線接続 1"
sudo nmcli con mod "$con" ipv4.address ${IP}/24
sudo nmcli con mod "$con" ipv4.gateway $GW
sudo nmcli con mod "$con" ipv4.dns $DNS
sudo nmcli con mod "$con" ipv4.method manual

VERSION_ID=$(cat /etc/os-release | awk -F= '/VERSION_ID/{print $2}')
Name=$(echo $IP | sed 's/.*\.//')-fedora$VERSION_ID
sudo hostnamectl hostname $Name.$Domain

sudo reboot
```

## hosts の設定

再起動後に hosts の更新。

```bash
sudo echo $(hostname -I) $(hostname) $(hostname -s) >> /etc/hosts
```

## FreeIPA ドメインへの参加

`192.168.1.24` `24-fedora43.localdomain.intra` が FreeIPA サーバであるとします。

```bash
sudo dnf -y install freeipa-client
```

```bash
IPA_SERVER=192.168.1.24
IPA_SERVER_FQDN=24-fedora43.localdomain.intra
```

Free IPA ドメインの管理者のパスワードを入力します。

```bash
read -s -p 'DOMAIN ADMIN PASSWORD?>' PW
```

```bash
sudo nmcli con mod '有線接続 1' ipv4.dns 192.168.1.24
sudo systemctl restart NetworkManager

DOMAIN=$(hostname -d)
ID=admin
sudo ipa-client-install --server=$IPA_SERVER_FQDN --domain $DOMAIN -p $ID -w $PW --mkhomedir -U
```

## DNSを修正

```bash
DNS1=192.168.1.24
DNS2=192.168.1.28
sudo nmcli con mod '有線接続 1' ipv4.dns $DNS1,$DNS2
sudo systemctl restart NetworkManager
```

## 次

未定
