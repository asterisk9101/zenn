---
title: "Fedora38 検証サーバを作るときに最初にやること"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora38, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

Proxmox 上の仮想サーバとして構築します。

## 仮想マシンの設定

基本的に全て任意（既定値）ですが、QEMU エージェントは有効にしておきます。

また、CPUコアは 2 コアを割り当てておきます。

## OSインストール設定

言語は**英語(English(United States))**を選択します。日本語だと FreeIPA サーバ(pki-tomcatd)のインストールが失敗したので。

LOCALIZATION は、日本語キーボードと日本語サポートを選択しておきます。Time & Date は自動的に Asia/Tokyo に設定されるはずです。

SOFTWARE では、ベースの環境を Fedora Server Edition を選択し、以下の項目を選択します。

- Domain Membership: FreeIPA Client がインストールされます。
- ゲストエージェント: Qemu Guest Agent がインストールされます。

SYSTEM では、インストール先をローカルドライブに自動構成とします。ネットワークとホスト名は、後ほど変更するのでデフォルトのままとします。

root は無効のままとし、wheel グループのユーザーを作成します。

## アップデートとバックアップの作成

ローカルコンソールにてアップデートを実行します。

```bash
dnf -y update
poweroff
```

この時点で Proxmox でバックアップを取得しておき、ベースイメージとします。

## ファイルシステムの拡張

インストール直後はファイルシステムに割り当てられている容量が少ないので拡張しておきます。

```bash
LVPath=$(lvdisplay -c | awk -F: '{print $1}')
lvextend -l +100%FREE $LVPath
xfs_growfs $LVPath
```

## systemd-resolved の停止

NetworkManager で管理しますので、systemd-resolved は停止します。

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved
rm -f /etc/resolv.conf
systemctl restart NetworkManager
```

## ホスト名とネットワークの設定

FreeIPA ドメインに参加する場合は FQDN 形式で設定する必要があります。ゲートウェイとDNSはホームルーターの初期値をそのまま使います。

ホスト名はIPアドレスを元に機械的に設定します（すぐ潰すことが多いので。。。）

```bash
read -p 'IP Address?> ' IP
```

```bash
GW=192.168.1.1
DNS=192.168.1.1
nmcli con mod ens18 ipv4.address ${IP}/24
nmcli con mod ens18 ipv4.gateway $GW
nmcli con mod ens18 ipv4.dns $DNS
nmcli con mod ens18 ipv4.method manual

Name=$(echo $IP | sed 's/.*\.//')-fedora38
Domain=localdomain.intra
hostnamectl hostname $Name.$Domain

reboot
```

## hostsの設定

自分自身の名前解決をできるようにしておきます。

```bash
IP=$(hostname -I)
DOMAIN=$(hostname -d)
ALIAS=${HOSTNAME/.${DOMAIN}/}

echo $IP $HOSTNAME $ALIAS >> /etc/hosts
```

## 次

<https://zenn.dev/asterisk9101/articles/fedora38server-2>
<https://zenn.dev/asterisk9101/articles/fedora38server-3>
