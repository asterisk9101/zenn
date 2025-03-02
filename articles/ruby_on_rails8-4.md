---
title: "Rails でコメントを追加する機能を追加する"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。

## 前提

前提です。

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-3>

## scaffold

モデルと一緒に諸々を作ります。

```bash
bundle exec rails g scaffold Activity \
    document:references \
    user_department:string \
    user_title:string \
    user_name:string \
    type:string

bundle exec rails db:migrate
```

## コメント本体と添付ファイル

コメントの本体はリッチテキスト（`Action Text`）に持たせます。ファイルの添付(`Active Storage`)ができるようにします。

```bash
vi app/models/activity.rb
```

```ruby
has_rich_text :content
has_many_attached :attachments

# Active Storage のアクセス制御用
def allow?(user)
  true
end
```

## ドキュメントモデルとの関連付け

Document モデルに関連付けて、ドキュメントから参照できるようにします。

```bash
vi app/models/document.rb
```

```ruby
has_many :activities, dependent: :destroy
```

Document ビューにコメントを追加するフォームを追加します。また、コメントは添付ファイルとリッチテキストと関連付けされているので `N+1` 対策もしておきます。

```bash
vi app/views/documents/show.html.erb
```

```erb
<p>activities</p>
<% form_with model: Activity.new do |form| %>
  <%# user_name などは、activities_controller.rb で入力するが省略 %>
  <%= form.hidden_field :document_id, value: @document.id %>
  <%= form.rich_textarea :content %>
  <%= form.file_field :attachments, multiple: true %>
  <br>
  <%= form.submit %>
<% end %>

<%# Action Text の N+1 対策: with_rich_text_content_and_embeds %>
<%# Active Storage の N+1 対策: with_attached_attachments %>
<% @document.activities.with_rich_text_content_and_embeds.with_attached_attachments.each do | act | %>
  <%= act.user_name %>
  <%= act.content %>
  <ul>
    <% act.attachments.each do |file| %>
      <li><%= link_to file.filename.to_s, rails_blob_path(file) %></li>
    <% end %>
  </ul>
<% end %>
```

Activity コントローラに追記して、POST されたときに元のドキュメントに戻るようにします。また、リッチテキストや添付ファイルを受け取れるようにしておきます。

```bash
vi app/controllers/activities_controller.rb
```

```diff ruby
# def create
- format.html { redirect_to @activity, notice: "Activity was successfully created." }
+ format.html { redirect_to document_path(@activity.document_id), notice: "Activity was successfully created." }

# activity_params
- params.expect(activity: [ :document_id, :user_department, :user_title, :user_name, :type ])
+ params.expect(activity: [ :document_id, :user_department, :user_title, :user_name, :type, :content, attachments: [] ])
```

## 編集機能の削除

`update` と `destory` はコメントアウトして編集不可能にします。

```bash
vi app/controllers/activities_controller.rb
```

次

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-5>
