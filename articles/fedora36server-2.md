---
title: "Fedora36 検証サーバでFreeIPAを使えるようにするときのメモ"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora36]
published: false
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

## データ用ディレクトリの作成

データを格納するディレクトリを作成します。

```bash
mkdir /var/lib/ipa-data
```

## FreeIPAコンテナの構成

FreeIPA-server を実行します(IPアドレスはサーバ自身の IP を指定します)

```bash
CONTAINER_NAME=ipa
IPA=192.168.1.21
DOMAIN=localdomain.intra
podman run -d --name $CONTAINER_NAME --log-driver journald \
    -h ipa.$DOMAIN \
    --read-only \
    --dns=127.0.0.1 \
    -v /var/lib/ipa-data:/data:Z \
    -e IPA_SERVER_IP=$IPA \
    -p 636:636 -p 80:80 -p 123:123 -p 389:389 -p 443:443 -p 88:88 -p 464:464 -p 53:53 \
    freeipa/freeipa-server:fedora-35 ipa-server-install \
    -a P@ssw0rd -p P@ssw0rd \
    --setup-dns --no-forwarders \
    -r $DOMAIN \
    --no-ntp \
    -U
```

インストールにしばらく時間がかかるのでログを眺めます(終了は Ctrl+C)。

```bash
podman logs -f ipa
```

# 自動起動の設定

podman はデーモンが居ないので、systemd にコントロールして貰うためにユニットファイルを作成します。

```bash
podman exec -it ipa bash
```
