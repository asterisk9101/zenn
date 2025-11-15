---
title: "Free IPA を止めずに OS をバージョンアップする"
emoji: "👒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, FreeIPA, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

Fedora 40 がサポート終了していたので 43 にバージョンアップしました。

Free IPA サーバは冗長構成にしていたので、実質的にドメインコントローラを止めずにバージョンアップすることができたと思います。

Free IPA サーバのヘビーユーザーではないので不備はあるかも知れませんが、SSO はできているので備忘録として残します。

## 構成

以下の2台構成です。

- 24-fedora40.localdomain.intra
- 28-fedora40.localdomain.intra

それぞれ FreeIPA ドメインの中で `first master` と `other master` の役割になるそうです。

`first master` は完全な `CA` の機能を持っており、以下のような役割を持ちます。

- CA 証明書の自動更新
- 証明書失効リスト（CRL）の生成と配布

`other master` は `CA` のコピーを持つだけで、障害などで `first master` を失ったときに役割を代替することができます。

## 役割の確認

念の為 CA 証明書の自動更新サーバと、CRLの生成と配布を行っているサーバがどれか確認しておきます。

```bash
# 証明書自動更新マスター
ipa config-show | grep "CA renewal master"

# CRL 生成マスター
ipa-crlgen-manage status
```

## other master のアップグレード

ドメインを維持するために `first master` が生きていれば良いので、まず `other master` である `28-fedora40.localdomain.intra` を降格します。

ドメインの管理者のパスワードを指定します。

```bash
read -s -p 'DOMAIN ADMIN PASSWORD?>' PW
```

```bash
# 認証
echo $PW | kinit admin

# レプリカから削除
ipa-replica-manage del 28-fedora40.localdomain.intra

#FreeIPA Server もアンインストール
ipa-server-install --uninstall -U
```

ドメインから離脱した `28-fedora40.localdomain.intra` の OS をクリーンインストールし、`28-fedora43.localdomain.intra` とします。IPアドレスはクリーンインストール前と同じです。

その後、ドメインに参加します。

```bash
IPA_SERVER=192.168.1.24
IPA_SERVER_FQDN=24-fedora40.localdomain.intra
```

クライアントをインストールし、ドメインに参加します。

```bash
nmcli con mod ens18 ipv4.dns 192.168.1.24
systemctl restart NetworkManager

DOMAIN=$(hostname -d)
ID=admin
ipa-client-install --server=$IPA_SERVER_FQDN --domain $DOMAIN -p $ID -w $PW --mkhomedir -U
```

FreeIPA Server パッケージを追加し、レプリカに昇格します。

```bash
dnf -y install freeipa-server freeipa-server-dns

# freeipa の通信を許可
firewall-cmd --add-service={freeipa-4,dns,ntp} --permanent
firewall-cmd --reload

ipa-replica-install --setup-ca --setup-dns --no-dnssec-validation --forwarder=192.168.1.1 -p admin -w $PW -U
```

## first master に指定

`28-fedora43.localdomain.intra` で完全な CA の役割を引き継ぎます。

```bash
# 証明書自動更新
ipa-csreplica-manage set-renewal-master

# CRL 生成
# 先に 24-fedora40.localdomain.intra で disable しておく
ipa-crlgen-manage enable
```

あとは、`first master` だったサーバの降格とクリーンインストールを実施して、必要に応じて `first master` の役割を戻して完了です。

## 参考

参考になりました

<https://www.freeipa.org/page/Backup_and_Restore>
