---
title: "Fedora41 サーバで Ruby on Rails を使うときのメモ"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。今回は `wheel` グループのユーザーです。

## DBサーバの用意

`PostgreSQL` を用意します。

<https://zenn.dev/asterisk9101/articles/fedora41server-3>

`postgres` ユーザーでアプリケーション用のユーザーとデータベースを作成しておきます。

```bash
read -p "App Name?> " AppName
```

```bash
psql -c "CREATE ROLE $AppName LOGIN ENCRYPTED PASSWORD '"$AppName"' VALID UNTIL 'infinity';"

# 開発環境のデータベースは docker などで開発端末のローカルに立てるべきです。
# 今回は本番に近い検証がしたかったので、独立したデータベースを用意しています。
createdb $AppName -O $AppName
createdb ${AppName}_development -O $AppName
createdb ${AppName}_test -O $AppName

# 確認
psql -l
```

## Webサーバの用意

アプリケーションの名前は DB サーバと同じにします。

```bash
read -p 'app name?> ' AppName
```

とりあえず必要なパッケージをインストールします。

```bash
# 多分必須のパッケージ
sudo dnf install -y ruby ruby-devel gcc libyaml-devel git

# DB接続や画像を扱うための要件次第で必要なパッケージ
sudo dnf install -y vips yarn postgresql-devel

# アップデートしておく
sudo gem update --system
```

Webサーバが使うポートを開放しておきます。

```bash
sudo firewall-cmd --add-port=3000/tcp --permanent
sudo firewall-cmd --reload
```

## rails のインストール

CI/CD は無いので Web サーバ上で直接作り始めます。普通はこんな状況無いです。

```bash
gem install rails

# ruby と一緒にインストールされるのは古いらしいので、改めてインストールする
gem install bundler
```

これらはシステム全体ではなく、ユーザーのディレクトリ(`~/.local`)にインストールされます。

`rails` の初期構築を行います。

```bash
mkdir $AppName
cd $AppName
rails new . --css bootstrap --javascript esbuild --database=postgresql
```

`sass` が `bootstrap` をビルドするときに警告が大量に出力するので抑制します。

```bash
sed -i -e 's/--no-source-map/-q --no-source-map/' package.json

# 確認
cat package.json
```

開発サーバを起動したときに外部から接続できるようにしておきます。

```bash
sed -i -e '1s/$/ -b 0.0.0.0/' Procfile.dev

# 確認
cat Procfile.dev
```

デバッグ用？のコンソールも外部から接続できるようにしておきます。

```bash
tgt="/Rails.application.configure do/"
cfg="config.web_console.allowed_ips = '0.0.0.0/0'"
sed -i -e "${tgt}a\  ${cfg}" config/environments/development.rb

# 確認
cat config/environments/development.rb | head
```

初回接続の際にデータベースが初期化されるので、データベースへの接続情報を設定する必要があります。

```bash
read -p 'database ip address?> ' DB
```

```bash
read -p 'database user?> ' ID
```

```bash
read -s -p 'database password?> ' PW
```

```bash
sed -i.bak -E \
    -e '/^\s*username:.*/d' \
    -e '/^\s*password:.*/d' \
    -e '/default:/a \  username: '$ID'' \
    -e '/default:/a \  password: '$PW'' \
    -e '/default:/a \  host: '$DB'' \
    config/database.yml

# 確認
diff config/database.yml{,.bak}
```

開発サーバを起動します。

```bash
bin/dev
```

`http://{WebサーバのIPアドレス}:3000/` にアクセスできれば開発の準備は完了です。

以上
