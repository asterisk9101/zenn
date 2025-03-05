---
title: "Rails でユーザーが複数の部署に所属することを表すメンバーシップモデルを作る"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。

## 前提

前提です。

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-4>

## 権限の境界としての部署

ユーザーを部署（Section）に所属させます。

ユーザーは複数の部署に所属することがあるので、中間テーブルとなる `Membership` モデルを作成します。

また、ユーザーは部署毎に役割を持つので、`role` 属性を持たせます。

```bash
bundle exec rails g scaffold Membership \
  section:references \
  user:references \
  role:string
bundle exec rails db:migrate
```

ユーザーから所属を参照したいので、モデルに追記します。

```bash
vi app/models/user.rb
```

```ruby
has_many :memberships, dependent: :destroy
has_many :sections, through: :memberships
```

同様に、部署から所属を参照したいので、モデルに追記します。

```bash
vi app/models/section.rb
```

```ruby
has_many :memberships, dependent: :destroy
has_many :users, through: :memberships
```

部署ビューに所属メンバーが表示されるようにします。また部署に参加と脱退ができるようにします。

```bash
vi app/views/sections/show.html.erb
```

```erb
<h1>Members</h1>
<ul>
  <% @section.users.each do |user| %>
    <li>
      <%= user.email %>
    </li>
  <% end %>
</ul>

<% if membership = @section.memberships.find_by(user_id: current_user.id) %>
  <%= button_to "Leave", membership, method: :delete %>
<% else %>
  <%= button_to "Join", memberships_path, params: { membership: { user_id: current_user.id, section_id: @section.id } } %>
<% end %>
```

この状態で `button_to` をクリックすると、Membership ビューの方にリダイレクトされてしまうので、Memberships コントローラを調整します。

```diff ruby
# create, update, destroy あたりにある
- redirect_to membership(s)_path, ...
+ redirect_to section_path(@membership.section_id), ...
```

部署への参加と脱退ができるようになりました。

次

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-6>
