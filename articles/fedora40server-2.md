---
title: "Fedora40 で zabbix を動かす"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora40, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

<https://zenn.dev/asterisk9101/articles/fedora40server-1>

## SELinuxの無効化

良くないなぁと思いながら無効化します。

```bash
sed -i.bak -E \
    's/SELINUX=enforcing/SELINUX=disabled/' \
    /etc/selinux/config
reboot
```

## PostgreSQL のインストール

RDBMS は `postgresql` を使います。インストールできたら、`initdb` してデータベースクラスタを作ります。後続の作業をするためにサービスを起動しておきます。

```bash
dnf -y install postgresql-server
postgresql-setup --initdb
systemctl enable --now postgresql
```

`postgres` ユーザーが作成されるので `sudo` して、`zabbix` ユーザーを作ります。

```bash
# 初期パスワードの入力を求められる
sudo -u postgres createuser zabbix -P
```

`zabbix` ユーザをオーナーに設定した `Zabbix` 用のデータベースを作ります。

```bash
sudo -u postgres createdb zabbix -O zabbix
```

Zabbix Web サービスから接続するために、認証方式を peer から md5 に変更し、変更を反映するためにサービスを再起動します。

```bash
sed -i.bak -E \
        -e '/^local +all/s/peer/md5/' \
        -e '/^host +all/s/ident/md5/' \
        /var/lib/pgsql/data/pg_hba.conf

systemctl restart postgresql
```

## IPv6 の無効化

IPv6 は邪魔になるらしいので無効化します。`ens18` の部分は環境に応じて変更します。

```bash
nmcli con show
nmcli con show ens18 | grep ipv6
nmcli con modify ens18 ipv6.method "disabled"
nmcli con up ens18
sed -i.bak -e '/^::/s/^/#/' /etc/hosts
```

## Apache HTTPD のインストール

Web サーバとして Apache HTTPD を使います。`ServerToken` だけ `Prod` にしてサービスを起動します。また、ファイアウォールを開けておきます。

```bash
dnf -y install httpd
echo 'ServerTokens Prod' >> /etc/httpd/conf/httpd.conf
systemctl enable --now httpd
firewall-cmd --add-service=http
firewall-cmd --runtime-to-permanent
```

## Zabbix サーバのインストール

Fedora 40 では以下のパッケージが提供されていますので、まとめてインストールします。

```bash
dnf -y install zabbix zabbix-dbfiles-pgsql zabbix-selinux zabbix-server zabbix-server-pgsql zabbix-web zabbix-web-pgsql
```

データベースの初期構築を行います。

```bash
read -s -p 'db password?> ' DBPASSWORD
```

```bash
export PGPASSWORD=$DBPASSWORD
psql -U zabbix zabbix < /usr/share/zabbix-postgresql/schema.sql
psql -U zabbix zabbix < /usr/share/zabbix-postgresql/images.sql
psql -U zabbix zabbix < /usr/share/zabbix-postgresql/data.sql
```

Zabbix サーバから DB へ接続できるようにパスワードを設定します。

```bash
echo 'DBPassword='$DBPASSWORD >> /etc/zabbix_server.conf
```

`setup.php` を実行するのが面倒なので、手動でコンフィグファイルを作ります。状況に応じて適宜変更してください。

```bash
cat << EOF > /etc/zabbix/web/zabbix.conf.php
<?php
\$DB['TYPE'] = 'POSTGRESQL';
\$DB['SERVER'] = '127.0.0.1';
\$DB['PORT'] = '0';
\$DB['DATABASE'] = 'zabbix';
\$DB['USER'] = 'zabbix';
\$DB['PASSWORD'] = '$DBPASSWORD';
\$DB['SCHEMA'] = '';
\$DB['ENCRYPTION'] = false;
\$DB['KEY_FILE'] = '';
\$DB['CERT_FILE'] = '';
\$DB['CA_FILE'] = '';
\$DB['VERIFY_HOST'] = false;
\$DB['CIPHER_LIST'] = '';
\$DB['VAULT_URL'] = '';
\$DB['VAULT_DB_PATH'] = '';
\$DB['VAULT_TOKEN'] = '';
\$DB['DOUBLE_IEEE754'] = true;
\$ZBX_SERVER = '127.0.0.1';
\$ZBX_SERVER_NAME = 'zbx';
\$IMAGE_FORMAT_DEFAULT = IMAGE_FORMAT_PNG;
?>
EOF
chown apache:apache /etc/zabbix/web/zabbix.conf.php
```

Zabbix の自動起動の設定を行います。なぜか `systemctl enable` できなかったので、手動で `ln` しています。

```bash
ln -s /usr/lib/systemd/system/zabbix-server.service /etc/systemd/system/multi-user.target.wants/zabbix-server.service
```

エージェントからの通信を受けつけるためのポートを開放します。

```bash
firewall-cmd --add-service=zabbix-server
firewall-cmd --runtime-to-permanent
```

各種設定が終わったらサービスを起動します。

```bash
systemctl restart httpd
systemctl start zabbix-server
```

## 接続テスト

以下の URL で接続できます。ユーザは `Admin` で、パスワードは `zabbix` です。

```bash
http://localhost/zabbix/
```

## Zabbix Agent のインストール

```bash
dnf -y install zabbix-agent
```

`zabbix-agent` を `root` で稼動させるように設定します。`/var/log/messages` など `root` でないと参照できないログを監視する場合はやむを得ず。

```bash
sed -i.bak -E \
    -e 's/# AllowRoot=0/AllowRoot=1/' \
    /etc/zabbix_agentd.conf
sed -i.bak -E \
    -e '/User=zabbix/d' \
    /usr/lib/systemd/system/zabbix-agent.service
```

`zabbix-agent` を起動します。

```bash
systemctl daemon-reload
systemctl enable --now zabbix-agent
```

## 参考

構築時の参考になりそうなログです。

```bash
less /var/log/zabbixsrv/zabbix_server.log
```

以上
