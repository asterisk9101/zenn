---
title: "Ruby on Rails の最初のページを作成するときのメモ"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。今回は `wheel` グループのユーザーです。

## Rails の用意

`Rails` をインストールした状態にします。

<https://zenn.dev/asterisk9101/articles/fedora41server-4>

## Homeページの作成

最初に簡単なホームページを作成します。

```bash
bundle exec rails generate controller home index
```

ルートに設定します。

```bash
sed -i 's/get "home\/index"/root "home#index"/' config/routes.rb
```

`bin/dev` で開発サーバを起動します。

ブラウザで `http://{サーバのIPアドレス}:3000/` にアクセスすると、`Home#index` と記載されたページが表示されれば成功です。

## 認証機能の追加

普通は何かしら認証機能があるはずなので、広く利用されている認証ライブラリ `devise` をいれてみます。

```bash
bundle add devise
bundle exec rails g devise:install

# user モデルが追加される
bundle exec rails g devise user

# マイグレーション
bundle exec rails db:migrate
```

一旦、`bin/dev` を終了して、起動し直します。

ブラウザで `http://{サーバのIPアドレス}:3000/` にアクセスすると、先ほどと同じく `Home#index` と記載されたページが表示されます。

`app/controllers/application_controller.rb` に以下を追記します。

```ruby
before_action :authenticate_user!
```

ブラウザで `http://{サーバのIPアドレス}:3000/` にアクセスすると、認証画面が表示されるようになります。

## ログインユーザーの作成

ログイン画面が表示されるようになりましたが、ユーザーが定義されていないためログインできません。

seed ファイルにユーザー登録用のコードを記載します。

```bash
vi db/seed.rb
```

```ruby
User.find_or_initialize_by(email: 'admin@example.com') do |user|
  user.password = 'P@ssw0rd'
  user.save!
end
```

以下のコマンドを実行して seed を実行します。

```bash
bundle exec rails db:seed
```

ブラウザで `http://{サーバのIPアドレス}:3000/` の認証画面で `admin@example.com` を使ってログインを試みます。

`Home#index` と記載されたページが表示されれば認証成功です。

## ログアウトとメッセージの表示

ログアウトリンクやログイン失敗の際にメッセージを表示するには、レイアウトに以下を追記します。

```bash
vi app/views/layouts/application.html.erb
```

```erb
<% if flash[:alert] %>
  <p style="color: red"><%= flash[:alert] %></p>
<% end %>
<header>
  <% if user_signed_in? %>
    <li>
      <%= link_to "logout", destroy_user_session_path, method: :delete %>
    </li>
  <% else %>
    <li>
      <%= link_to "signup", new_user_registration_path %>
    </li>
    <li>
      <%= link_to "login", new_user_session_path %>
    </li>
  <% end %>
</header>
```

以上
