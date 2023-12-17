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

images を含むディレクトリは php-fpm にプロキシしない設定が必要だけど方法が分からなかった
