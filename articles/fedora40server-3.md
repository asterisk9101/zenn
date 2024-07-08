---
title: "Fedora40 で Zabbix Agent を動かす"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

Zabbix Server に Agent をインストールして監視してみます。

<https://zenn.dev/asterisk9101/articles/fedora40server-2>

## SELinuxの無効化

良くないなぁと思いながら無効化します。

```bash
sed -i.bak -E \
    's/SELINUX=enforcing/SELINUX=disabled/' \
    /etc/selinux/config
reboot
```

## インストール

```bash
dnf -y install zabbix-agent
```

```bash
read -p "Zabbix Server IpAddress? " zbx
```

```bash
sed -i.bak -E \
    -e "s/^Server=127.0.0.1/Server=$zbx/" \
    -e "s/^ServerActive=127.0.0.1/ServerActive=$zbx/" \
    -e "s/^Hostname=Zabbix server/Hostname=$(hostname -s)/" \
    -e 's/# AllowRoot=0/AllowRoot=1/' \
    /etc/zabbix_agentd.conf
```

```bash
firewall-cmd --add-service=zabbix-agent
```

```bash
sed -i.bak -E -e '/User=zabbix/d' /usr/lib/systemd/system/zabbix-agent.service
systemctl daemon-reload
systemctl enable --now zabbix-agent
```
