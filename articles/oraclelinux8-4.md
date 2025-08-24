---
title: "OracleLinux8.10 の Oracle Database 21c にデータベースを作成する"
emoji: "🐧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Oracle, memo]
published: true
---

## 前提

続きです。

<https://zenn.dev/asterisk9101/articles/oraclelinux8-3>

## DBCA でデータベースを構成する

Oracle Database 21c にデータベースを作ります。

Oracle ユーザーで操作します。

```bash
export LANG=C
export DISPLAY=:0
dbca
```

Create a database を選んで次へ

![1](/images/oraclelinux8-4/05.png)

Advanced configuration を選んで次へ

![1](/images/oraclelinux8-4/06.png)

General Purpose or Transaction Processing を選んで次へ

![1](/images/oraclelinux8-4/07.png)

しばらく変更せずに次へ

![1](/images/oraclelinux8-4/08.png)

![1](/images/oraclelinux8-4/09.png)

![1](/images/oraclelinux8-4/10.png)

![1](/images/oraclelinux8-4/11.png)

![1](/images/oraclelinux8-4/12.png)

![1](/images/oraclelinux8-4/13.png)

![1](/images/oraclelinux8-4/14.png)

EM は非推奨らしいので Configure Enterprise Manager ... のチェックを外して次へ

![1](/images/oraclelinux8-4/15.png)

管理者共通のパスワードを設定して次へ

![1](/images/oraclelinux8-4/16.png)

Create database を選んで次へ

![1](/images/oraclelinux8-4/17.png)

確認画面が表示されて完了です。

![1](/images/oraclelinux8-4/18.png)
![1](/images/oraclelinux8-4/19.png)
![1](/images/oraclelinux8-4/20.png)
![1](/images/oraclelinux8-4/21.png)

## 接続確認

ドキュメントには記載がなかったのですが、 `tnsnames.ora` の配置しているパスをエクスポートする必要がありました。

`.bash_profile` に記載しておきます。

```bash
# .bash_profile
export TNS_ADMIN=/u01/app/oracle/homes/OraDB21Home1/network/admin
```

sqlplus を使って sysdba でログインします。

```bash
sqlplus sys@orcl as sysdba
```

接続状態を確認します。

```sql
show con_name

-- CON_NAME
-- ------------------------------
-- CDB$ROOT
```

`CDB$ROOT` に接続されます。ここは各 PDB の共通設定などを置いておくところだそうです。

続いて、PDB の状態を確認します。

```sql
-- オープンモードの確認
COLUMN NAME FORMAT A15
COLUMN RESTRICTED FORMAT A10
COLUMN OPEN_TIME FORMAT A30
SELECT NAME, OPEN_MODE, RESTRICTED, OPEN_TIME FROM V$PDBS;

-- NAME            OPEN_MODE  RESTRICTED OPEN_TIME
-- --------------- ---------- ---------- ------------------------------
-- PDB$SEED        READ ONLY  NO         25-08-22 01:32:32.593 +09:00
-- ORCLPDB1        MOUNTED

```

`MOUNTED` なので、オープンしていません。

PDB をオープンします。

```sql
-- ORCLPDB1 をオープンする
alter pluggable database orclpdb1 open;

-- OPEN したか確認
SELECT NAME, OPEN_MODE, RESTRICTED, OPEN_TIME FROM V$PDBS;

-- NAME            OPEN_MODE  RESTRICTED OPEN_TIME
-- --------------- ---------- ---------- ------------------------------
-- PDB$SEED        READ ONLY  NO         25-08-22 01:32:32.593 +09:00
-- ORCLPDB1        READ WRITE NO         25-08-22 23:39:47.039 +09:00
```

`ORCLPDB1` がオープンしたので、接続しているセッションを変更します。

```sql
-- セッション変更
alter session set container = ORCLPDB1;

-- 現在のセッションの確認
show con_name

-- CON_NAME
-- ------------------------------
-- ORCLPDB1
```

想定した PDB に接続できました。

## 停止と再起動

Grid を含めた停止と起動について確認します。

```bash
# Grid 全体の制御を行っている High Availability Service のステータス確認
crsctl check has

# Grid を構成するリソースのステータス確認
crsctl status res -t

# 停止
crsctl stop has

# 起動
crsctl start has
```

プラガブルデータベースは相変わらず MOUNTED なので、上記の手順で起動してやる必要がありました。

自動起動の方法もありますが、一旦はこのまま。
