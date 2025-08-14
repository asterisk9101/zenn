---
title: "OracleLinux8.10 で検証サーバを作るときに最初にやること"
emoji: "🐧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Oracle, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

Proxmox 上の仮想サーバとして構築します。

尚、Oracle Linux のバージョンは 8.10 RHCK です。

## 仮想マシンの設定

Oracle Grid Infrastructure 21c を構成したいです。

QEMU エージェントは有効にしておきます。

CPUコアは 2 コアを割り当てておきます。

メモリは、Grid Infrastructure の最小要件は 8GB ではあるものの、Database 21c を載せるので 12GB 程度割り当てておきます。
動きが悪ければ拡張することにします。

後で swap を追加するので sda は 64GB を割り当てます。
32GB のディスクを2つ追加しておきます（sdb, sdc）

シングル想定なので NIC は 1 つでも大丈夫な模様。

## OSインストール設定

特に変更していないため省略。`dnf update` はインストール時点で行われる様子。

この時点で Proxmox でバックアップを取得しておき、ベースイメージとします。

## 公開鍵認証の設定

初回ログイン時はパスワードでログインし、以下のコマンドを実行します。※入力待ちになります

```bash
mkdir -m 700 .ssh
# 入力待ちになるので、公開鍵を貼り付けて Ctrl+D する
cat > .ssh/authorized_keys
```

操作端末(Windows)で以下のコマンドを実行します。

```powershell
Get-Content "$home\.ssh\id_ed25519.pub" | % { $_ + "`n" } | Set-Clipboard
```

入力待ちのコンソールに貼り付けて、`Ctrl+D` を押下します。※入力イメージです。

```bash
cat > .ssh/authorized_keys
ssh-ed25519 xxxxxxxxxxxxxxxxxxxxxxx username@hostname
# ここで Ctrl+D
```

```bash
chmod 600 .ssh/authorized_keys
```

## sudo のパスワード省略設定

root になるときも初回はパスワード入力します。

```bash
sudo su -
```

`wheel` グループのユーザはパスワード無しで `sudo` できるように設定します。

```bash
echo '%wheel ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/wheel
```

## ホスト名とネットワークの設定

FreeIPA ドメインに参加する場合は FQDN 形式で設定する必要があります。ゲートウェイとDNSはホームルーターの初期値をそのまま使います。

ホスト名はIPアドレスを元に機械的に設定します（すぐ潰すことが多いので。。。）

```bash
read -p 'IP Address?> ' IP
```

```bash
read -p 'Domain?> ' Domain
```

```bash
GW=192.168.1.1
DNS=192.168.1.1
nmcli con mod ens18 ipv4.address ${IP}/24
nmcli con mod ens18 ipv4.gateway $GW
nmcli con mod ens18 ipv4.dns $DNS
nmcli con mod ens18 ipv4.method manual

xx=$(echo $IP | tr '\.' '\n' | tail -n 1)
ver=$(cat /etc/os-release | grep "^VERSION=" | sed 's/.*=//' | tr -d '"' | sed 's/\..*//')
hostnamectl set-hostname "${xx}-oracle${ver}.${Domain}"

reboot
```

## hostsの設定

自分自身の名前解決をできるようにしておきます。

```bash
echo $(hostname -I) $(hostname) $(hostname -s) >> /etc/hosts
```
