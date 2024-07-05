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

## PostgreSQL のインストール

RDBMS は `postgresql` を使います。インストールできたら、`initdb` してデータベースクラスタを作ります。後続の作業をするためにサービスを起動しておきます。

```bash
dnf -y install postgresql-server
postgresql-setup --initdb
systemctl enable --now postgresql
```

`postgres` ユーザーが作成されるのでスイッチして、`zabbix` ユーザーとデータベースを作ります。

```bash
su - postgres

# 初期パスワードの入力を求められる
createuser zabbix -P
```

`Zabbix` 用のデータベースを作って、`zabbix` ユーザをオーナーに設定します。`postgres` ユーザーからはログアウトします。

```bash
createdb zabbix -O zabbix
exit
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

Zabbix ウェブアプリケーションが DB にアクセスできるように、SELinux ポリシーを変更します。

```bash
setsebool -P httpd_can_network_connect_db=1
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
cd /usr/share/zabbix-postgresql
psql -U zabbix zabbix < schema.sql
psql -U zabbix zabbix < images.sql
psql -U zabbix zabbix < data.sql
# ホームディレクトリに戻る
cd
```

Zabbix サーバから DB へ接続できるようにパスワードを設定します。

```bash
echo 'DBPassword=zabbix' >> /etc/zabbix_server.conf
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
\$DB['PASSWORD'] = 'zabbix';
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

`php-fpm` から `zabbix-server` に接続するための許可ポリシーを有効にします。

```bash
setsebool -P httpd_can_connect_zabbix 1
```

`zabbix-server` から `/var/kerberos/krb5` にアクセスできないとエラーログが残るので、SELinux ポリシーを作ってインストールします。

```bash
ausearch -c 'zabbix_server' --raw | audit2allow -M my-zabbixserver
semodule -i my-zabbixserver.pp
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

## 参考

構築時の参考になりそうなログです。

```bash
less /var/log/audit/audit.log
less /var/log/zabbixsrv/zabbix_server.log
```

以上
