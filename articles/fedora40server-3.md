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

`zabbix-agent` をインストールします。

```bash
dnf -y install zabbix-agent
```

`zabbix server` の IP アドレスを入力します。

```bash
read -p "Zabbix Server IpAddress? " zbx
```

`zabbix server` のアドレスを設定するとともに、 `zabbix-agent` を `root` で動かすようにします。`root` でしか参照できないログ(`/var/log/messages`など)を監視するためです。

```bash
sed -i.bak -E \
    -e "s/^Server=127.0.0.1/Server=$zbx/" \
    -e "s/^ServerActive=127.0.0.1/ServerActive=$zbx/" \
    -e "s/^Hostname=Zabbix server/Hostname=$(hostname -s)/" \
    -e 's/# AllowRoot=0/AllowRoot=1/' \
    /etc/zabbix_agentd.conf
```

`root` で動かすために `zabbix_agentd.conf` だけではなくて、`systemd` のユニットファイルも修正する必要があります。

```bash
sed -i.bak -E -e '/User=zabbix/d' /usr/lib/systemd/system/zabbix-agent.service
systemctl daemon-reload
```

`zabbix` の特徴として、サーバからエージェントにメトリクスを取りに行くプル型であるので、監視される側のファイアウォールを開ける必要があります。

```bash
firewall-cmd --add-service=zabbix-agent
```

`zabbix-agent` を起動します。

```bash
systemctl enable --now zabbix-agent
```

## 監視設定の追加

`zabbix server` の web コンソールにログインし、「設定」→「ホスト」→「ホストの作成」をクリックします。

「ホスト名」と「テンプレート」、「グループ」を設定し、「追加」をクリックします。

※「テンプレート」を設定しておかないと、サーバがメトリクスを取りに行かないので、いつまで経っても動いていないように見えます。

## トラブルシューティング

`zabbix server` 側で `zabbix_get` コマンドを使って、値が取得できるか確認します。

<https://www.zabbix.com/documentation/6.4/jp/manual/concepts/get>
