---
title: "Fedoraサーバーでコンテナを使えるようにするときのメモ"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora36]
published: false
---

お家の検証サーバ用の備忘録です。基本 root です。

```bash
# FW を開通しておきましょう
firewall-cmd --add-service={freeipa-4,dns,ntp} --permanent
firewall-cmd --reload

# systemd でコンテナを実行する場合に true にしておきます
getsebool container_manage_cgroup
setsebool -P container_manage_cgroup 1
getsebool container_manage_cgroup

# データを格納するディレクトリを作成します
mkdir /var/lib/ipa-data

# FreeIPA-server を実行します(IPアドレスはサーバ自身の IP を指定します)
IPA=192.168.1.21
DOMAIN=localdomain.intra
podman run -d --name ipa --log-driver journald \
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

# 稼働確認

```bash
podman ps -n ipa
```
