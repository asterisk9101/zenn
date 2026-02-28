---
title: "認証付き minitest の使い方"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。

## プロジェクト

まずプロジェクトを初期化します。

```bash
mkdir minitest && cd $_
rails new .
```

## データの準備

`scaffold` する

```bash
bin/rails g scaffold Post title:string content:text
bin/rails db:migrate
```

見えるように

```ruby:config/routes.rb
root "posts#index"
```

この時点では全てのテストがパス

```bash
bin/rails test
```

## 認証機能の追加

認証モジュールの追加

```bash
bundle add devise
```

認証機能を利用できるように

```bash
bin/rails g devise:install
bin/rails g devise user
bin/rails db:migrate
```

```ruby:app/controller/application_controller.rb
before_action :authenticate_user!
```

認証していないのでテストが失敗するようになります。

```bash
bin/rails test
```

## テストを成功させる

ヘルパーを追加します。

```ruby:test/test_helper.rb
# ...
class ActionDispatch::IntegrationTest
  include Devise::Test::IntegrationHelpers
end
# ...
```

認証用のユーザーを追加します。

```yaml:test/fixtures/users.yml
one:
  email: user@example.com
  encrypted_password: <%= Devise::Encryptor.digest(User, 'password') %>
two:
  email: user2@example.com
  encrypted_password: <%= Devise::Encryptor.digest(User, 'password') %>
```

認証を追加する。

```ruby:test/controllers/posts_controller_test.rb
# ...
setup do
  @post = posts(:one)
  @user = users(:one)
  sign_in @user
end
# ...
```

これでテストが成功するようになります。

```bash
bin/rails test
```
