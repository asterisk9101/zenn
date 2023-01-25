---
title: "Fedora36 検証サーバでFreeIPAを使えるようにするときのメモ"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora36, FreeIPA, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 参考にした記事

<https://github.com/freeipa/freeipa-container>

## Podman のインストール

検証なので Podman を使います。

```bash
dnf -y install podman
```

## selinuxの設定

FreeIPA のイメージはコンテナ内で `systemd` を使うので `container_manage_cgroup` を true にしておきます。

```bash
getsebool container_manage_cgroup
setsebool -P container_manage_cgroup 1
getsebool container_manage_cgroup
```

## Firewalld の設定

freeipa 用の定義があるので利用する（freeipa-ldap や freeipa-replication は deprecated だそうです）

```bash
firewall-cmd --add-service={freeipa-4,dns,ntp} --permanent
firewall-cmd --reload
```

## データ用ディレクトリの作成

データを格納するディレクトリを作成します。

```bash
mkdir /var/lib/ipa-data
```

## FreeIPAコンテナの構成

FreeIPA-server を実行します(IPアドレスはサーバ自身の IP を指定します)

```bash
CONTAINER_NAME=ipa
ADMINPASS=P@ssw0rd
IPA=192.168.1.21
DOMAIN=localdomain.intra
podman run -d --name $CONTAINER_NAME --log-driver journald \
    -h $CONTAINER_NAME.$DOMAIN \
    --read-only \
    --dns=127.0.0.1 \
    -v /var/lib/ipa-data:/data:Z \
    -e IPA_SERVER_IP=$IPA \
    -p 636:636 -p 80:80 -p 123:123 -p 389:389 -p 443:443 -p 88:88 -p 464:464 -p 53:53 -p 53:53/udp \
    freeipa/freeipa-server:fedora-35 ipa-server-install \
    -a $ADMINPASS -p $ADMINPASS \
    --setup-dns --no-forwarders \
    -r $DOMAIN \
    --no-ntp \
    -U
```

インストールにしばらく時間がかかるのでログを眺めます(終了は Ctrl+C)。

```bash
podman logs -f ipa
```

## 稼働確認

実際に FreeIPA が稼働しているか確認するために、コンテナに入って Kerberos チケットが発行されることを確認します。

```bash
podman exec -it ipa bash
```

コンテナの中の操作です（抜けるには exit）

```bash
# ログイン直後はチケットが無い
klist

# チケットの入手
kinit admin

# チケットが表示される
klist
```

ついでにデフォルトのログインシェルを `bash` に変更しておきます。

```bash
ipa config-mod --defaultshell=/bin/bash
```

## 自動起動の設定

podman はデーモンが居ないので、systemd にコントロールして貰うためにユニットファイルを作成します。

```bash
podman generate systemd --name ipa > /etc/systemd/system/ipa.service
podman stop -t 20 ipa

# ipa.service ファイルの ExceStop も同様に -t 20 に変更する
nano /etc/systemd/system/ipa.service
```

```bash
systemctl status ipa
systemctl enable ipa
reboot

# 起動確認
systemctl status ipa
```

## 管理画面のポップアップを抑止する

FreeIPAの管理画面（Webコンソール）へアクセスしたとき、認証を要求するポップアップが表示されますが、端末が正しく設定されていないとこの認証は2回失敗します。

何が起きているのかは以下のブログに記載があります。

<http://jdshewey.blogspot.com/2017/08/fixing-annoying-popup-in-freeipa.html>

ブログの解説に従って、`/etc/httpd/conf.d/ipa-rewrite.conf` に、以下の設定を追記することでポップアップを抑止します。

```conf
#The following disables the annoying kerberos popup for Chrome
RewriteCond %{HTTP_COOKIE} !ipa_session
RewriteCond %{HTTP_REFERER} ^(.+)/ipa/ui/$
RewriteRule ^/ipa/session/json$ - [R=401,L]
RedirectMatch 401 ^/ipa/session/login_kerberos
```

以上
