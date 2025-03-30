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

## 所属モデル

ユーザーを部署（Section）に所属させます。

ユーザーは複数の部署に所属することがあるので、中間テーブルとなる `Membership` モデルを作成します。

```bash
bundle exec rails g model Membership \
  section:references \
  user:references 
bundle exec rails db:migrate
bundle exec rails g controller memberships create destroy
```

ユーザーから所属を参照したいのでモデルに追記します。

```bash
vi app/models/user.rb
```

```ruby
has_many :memberships
has_many :sections, through: :memberships
```

同様に、部署から所属を参照したいので、モデルに追記します。

```bash
vi app/models/section.rb
```

```ruby
has_many :memberships
has_many :users, through: :memberships
```

部署ビューに所属メンバーが表示されるようにします。

```bash
vi app/views/sections/_section.html.erb
```

```erb
<p>
  <strong>Members:</strong>
  <%= section.users.pluck(:email).join("; ") %>
</p>
```

部署ビューから所属メンバーを編集できるようにします。

```bash
vi app/views/sections/_form.html.erb
```

```erb
<div>
  <%= form.label :members, style: "display: block" %>
  <%= form.textarea :members, value: section.users.pluck(:email).join("; ") %>
</div>
```

部署モデルにメンバーを更新するメソッドを生やします。

```bash
vi app/models/section.rb
```

```ruby
  def update_members(joined_emails)
    emails = joined_emails.split(";").map(&:strip)
    member_emails = self.users.pluck(:email)

    add_users = User.where(email: emails.difference(member_emails))
    del_users = User.where(email: member_emails.difference(emails))

    self.users << add_users
    self.memberships.where(user_id: del_users.pluck(:id)).destroy_all
  end
```

部署コントローラーにメンバーを受け取って、メンバーを更新するメソッドに渡すよう記載します。

```bash
vi app/controllers/sections_controller.rb
```

```ruby
# create の @section.save の後と
# update の @section.update(section_params) の後
# current_user.email は強制的に追加するようにしておく
@section.update_members(params[:section][:members] + ";" + current_user.email)
```

部署ビューで自分自身の参加と脱退ができるようにします。

```bash
vi app/views/sections/show.html.erb
```

```erb
<% if membership = @section.memberships.find_by(user_id: current_user.id) %>
  <%= button_to "Leave", membership, method: :delete %>
<% else %>
  <%= button_to "Join", memberships_path, params: { membership: { user_id: current_user.id, section_id: @section.id } } %>
<% end %>
```

この状態で `button_to` をクリックすると、Membership ビューの方にリダイレクトされてしまうので、Memberships コントローラを調整します。

```diff ruby
# create, destroy あたりにある
- redirect_to membership(s)_path, ...
+ redirect_to section_path(@membership.section_id), ...
```

部署への参加と脱退ができるようになりました。

次

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-6>
