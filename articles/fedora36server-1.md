---
title: "Fedora36 検証サーバを作るときに最初にやること"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora36, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## ファイルシステムの拡張

インストール直後はファイルシステムに割り当てられている容量が少ないので拡張しておきます。

```bash
LVPath=$(lvdisplay -c | awk -F: '{print $1}')

# LV の拡張
lvextend -l +100%FREE $LVPath

# ファイルシステムの拡張
xfs_growfs $LVPath
```

## systemd-resolved の無効化

なぜか NetworkManager と systemd-resolved が `/etc/resolv.conf` を取り合って通信を阻害することがあるので、systemd-resolved を無効にします。

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved
rm /etc/resolv.conf
systemctl restart NetworkManager
```

## よく使うツール類のインストール

検証するとき前提になりやすいものやツール類をインストールします。

```bash
dnf -y update && \
dnf -y install \
    git \
    source-highlight \
    jq \
    nano \
    tmux \
    bat \
    fd-find \
    htop \
    podman \
    shellcheck
```

ツールをインストールしたら、ツールの設定を追加します。グローバル設定なのは自分しか使わないからです。本番環境ではやっちゃ駄目。

```bash
# source-highlight
echo "alias hilite='/usr/bin/src-hilite-lesspipe.sh'" >> /etc/bashrc
echo "alias less='less -R'" >> /etc/bashrc

# tmux
echo "set-option -g prefix C-q" >> /etc/tmux.conf
echo "unbind-key C-b" >> /etc/tmux.conf
echo "bind-key C-q send-prefix" >> /etc/tmux.conf

# nano
echo "set autoindent" >> /etc/nanorc
echo "set tabsize 2" >> /etc/nanorc
echo "set tabstospaces" >> /etc/nanorc

# docker
echo "alias docker=podman" >> /etc/bashrc
```

インストールと初期設定が終わったら再起動（もしくは停止してバックアップ）します。

```bash
reboot
```

以上。
