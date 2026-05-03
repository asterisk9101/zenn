---
title: "Rails の ActionText で lexxy を使ってみる"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。

## 前提

前提です。

<https://zenn.dev/asterisk9101/articles/fedora43server-1>

```bash
ruby --version
# ruby 3.4.8 (2025-12-17 revision 995b59f666) +PRISM [x86_64-linux]

rails --version
# Rails 8.1.2
```

## 環境作り

とりあえず `rails new .` します。

```bash
rails new lexxy-actiontext
cd $_
```

`Action Text` をインストールします。この時点では `trix` がインストールされます。

```bash
bin/rails action_text:install
bin/rails db:migrate
```

動作確認のために `Post` モデルを作ります。

```bash
bin/rails g scaffold post title:string
bin/rails db:migrate
```

`Post` モデルに `has_rich_text` を付けます。

```ruby:app/models/post.rb
class Post < ApplicationRecord
  has_rich_text :content
end
```

`Post` のビューを修正します。

```ruby:app/views/posts/_form.html.erb
<div>
  <%= form.label :content, style: "display: block" %>
  <%= form.rich_text_area :content %>
</div>
```

```ruby:app/views/posts/_post.html.erb
<div>
  <strong>Content:</strong>
  <%= post.content %>
</div>
```

`Post` のコントローラを修正します。

```ruby:app/controllers/posts_controller.rb
def post_params
  # params.expect(post: [ :title ])
  params.expect(post: [:title, :content])
end
```

`config/routes.rb` を修正します。

```ruby:config/routes.rb
root "posts#index"
```

一旦動作確認してみます。

```bash
bin/dev
```

`http://localhost:3000/` にアクセスして画面が表示されればOK。

`New post` をクリックすると `trix` エディタが表示される。

## lexxy に入れ替える

`Gemfile` に追記して使えるようにします。

```ruby:Gemfile
gem 'lexxy', '~> 0.1.26.beta'
```

```bash
bundle install
```

`config/importmap.rb` を修正します。

```ruby:config/importmap.rb
# pin "trix"
# pin "@rails/actiontext", to: "actiontext.esm.js"
pin "lexxy", to: "lexxy.js"
pin "@rails/activestorage", to: "activestorage.esm.js"
```

`app/javascript/application.js` を修正します。

```javascript:app/javascript/application.js
// import "trix"
// import "@rails/actiontext"
import "lexxy"
```

ビューを修正します。

```erb:app/views/layouts/application.html.erb
<%= stylesheet_link_tag "lexxy" %>
```

```erb:app/views/layouts/action_text/contents/_content.html.erb
<!-- <div class="trix-content"> ->
<div class="lexxy-content" data-controller="syntax-highlight">
```

コードハイライトするために stimulus につなぎます。
公式ドキュメントは `highlightCode` になっていますが、正しくは `highlightAll` だそうです。

```javascript:app/javascript/syntax_highlight_controller.js
import { Controller } from "@hotwired/stimulus"
import { highlightAll } from "lexxy"

export default class extends Controller {
  connect() {
    highlightAll()
  }
}
```

`bin/dev` して `trix` が `lexxy` に置き換わったことを確認します。

## 参考

<https://basecamp.github.io/lexxy/installation.html>

<https://railsguides.jp/action_text_overview.html>

<https://prismjs.com/>
