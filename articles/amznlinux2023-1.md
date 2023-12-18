---
title: "Amazon Linux 2023 で MediaWiki を動かす"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Amazon, memo]
published: false
---


```bash
dnf install -y httpd git php php-{intl,gd,mysqlnd}
cat <<EOS > /var/www/html/index.php
<?php
phpinfo();
?>
EOS
systemctl enable --now httpd

# 稼動確認
curl http://localhost/
```

```bash
# https://github.com/amazonlinux/amazon-linux-2023/issues/206
dnf install -y gcc make php-{devel,pear}
pecl channel-update pecl.php.net
pear config-set php_ini /etc/php.ini
yes '' | pecl install apcu
```

```bash
curl -O https://releases.wikimedia.org/mediawiki/1.40/mediawiki-1.40.1.tar.gz
tar xzf mediawiki-1.40.1.tar.gz
cp -r mediawiki-1.40.1/* /var/www/html/
chown -R apache:apache /var/www/html/
```

```bash
dnf install -y mariadb105-server
systemctl enable --now  mariadb
mysql_secure_installation
mysql
```

```sql
CREATE DATABASE my_wiki;
CREATE USER 'wikiuser'@'localhost' IDENTIFIED BY 'database_password';
GRANT ALL PRIVILEGES ON my_wiki.* TO 'wikiuser'@'localhost' WITH GRANT OPTION;
```

images を含むディレクトリは php-fpm にプロキシしない設定が必要だけど方法が分からなかった。

---

## docker を使う方法

```bash
dnf install -y docker
systemctl enable --now docker
```

```bash
docker volume create images
docker volume create db
docker network create mediawiki
docker run -d --restart unless-stopped --network mediawiki --name db -v db:/var/lib/mysql -e MYSQL_DATABASE=my_wiki -e MYSQL_USER=wikiuser -e MYSQL_PASSWORD=P@ssw0rd123 -e MYSQL_RANDOM_ROOT_PASSWORD=yes mariadb
docker run -d --restart unless-stopped --network mediawiki --name mediawiki -v images:/var/www/html/images -v /root/LocalSettings.php:/var/www/html/LocalSettings.php -p 80:80 mediawiki
```

```bash
docker run -it mediawiki bash

# バックアップ
/var/www/html/maintenance/run dumpBackup.php --full --output=gzip:/tmp/wiki.xml.gz

# リストア
/var/www/html//maintenance/run importDump.php < /tmp/wiki.xml.gz
```

images がバックアップされているのか不明
