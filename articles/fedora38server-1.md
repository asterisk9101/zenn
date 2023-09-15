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

言語は日本語を選択します。

インストール先はローカルドライブに自動構成とします。

ソフトウェアの選択では、ベースの環境を Fedora Server Edition を選択し、以下の項目を選択します。

- Container Management: Podman がインストールされます。
- Domain Membership: FreeIPA Client がインストールされます。
- ゲストエージェント: Qemu Guest Agent がインストールされます。

ネットワークとホスト名はデフォルトのままとします。

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

# LV の拡張
lvextend -l +100%FREE $LVPath

# ファイルシステムの拡張
xfs_growfs $LVPath
```

## ホスト名とネットワークの設定

```bash
hostnamectl hostname hogehoge
nmcli con mod ens18 ipv4.address 192.168.1.30/24
nmcli con mod ens18 ipv4.gateway 192.168.1.1
nmcli con mod ens18 ipv4.dns 192.168.1.1
nmcli con mod ens18 ipv4.method manual
reboot
```

以上
