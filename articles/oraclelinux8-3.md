---
title: "OracleLinux8.10 に Oracle Database 21c を構成する"
emoji: "🐧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Oracle, memo]
published: true
---

## 前提

続きです。

<https://zenn.dev/asterisk9101/articles/oraclelinux8-2>

## Oracle Database 21c のインストーラの準備

インストーラを展開します。

```bash
cd /u01/app/oracle/product/21.3.0/dbhome_1
unzip -q ~/db_home.zip
```

## Oracle Database 21c をインストールする

```bash
export LANG=C
export DISPLAY=:0
./runInstaller
```

Set Up Software Only を選びます。自分で作った方が理解が捗るので。

![1](/images/oraclelinux8-3/01.png)

Single instance database installation を選びます。

![2](/images/oraclelinux8-3/02.png)

Standard Edition 2 を選びます。

![3](/images/oraclelinux8-3/03.png)

インストールディレクトリを選択します。

![4](/images/oraclelinux8-3/04.png)

Oracle Database の権限とOSグループを紐づけます。

![5](/images/oraclelinux8-3/05.png)

`root.sh` を実行します。

![6](/images/oraclelinux8-3/06.png)

インストール前のチェックがあります。

![7](/images/oraclelinux8-3/07.png)

clock source として tsc を選択するように推奨されますが、特にパフォーマンスを重視しているわけではないので kvm-clock のまま Ignore All をチェックして次へ。

![8](/images/oraclelinux8-3/08.png)

確認画面が表示されます。

![9](/images/oraclelinux8-3/09.png)

確認画面が表示されますが次へ。

![10](/images/oraclelinux8-3/10.png)

Oracle Database 21c のインストールが終わりました。

![11](/images/oraclelinux8-3/11.png)

## 次

<https://zenn.dev/asterisk9101/articles/oraclelinux8-4>
