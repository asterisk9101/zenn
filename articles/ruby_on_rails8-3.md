---
title: "Rails に Action Text を追加する"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。

## 前提

`Active Storage` のインストールは終わっている状態です。

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-2>

## 文書にリッチテキストコンテンツを追加する

`ActionText` を追加します。

```bash
bundle exec rails action_text:install
bundle exec rails db:migrate
```

モデルに関連を追記します。

```bash
vi app/models/document.rb
```

```ruby
has_rich_text :content
```

ビューにリッチテキストを表示できるよう追記します。

```bash
vi app/views/documents/_form.html.erb
```

```erb
<div>
  <%= form.label :content, style: "display: block" %>
  <%= form.rich_textarea :content %>
</div>
```

```bash
vi app/views/documents/_document.html.erb
```

```erb
<p>
  <strong>Content:</strong>
  <%= document.content %>
</p>
```

コントローラに追加します。

```bash
vi app/controllers/documents_controller.rb
```

```diff ruby
- params.expect(document: [ :name, :status, :due_date, :section_id, attachments: [] ])
+ params.expect(document: [ :name, :status, :due_date, :section_id, :content, attachments: [] ])
```

## 画像の表示

`Action Text` に添付した画像は `Active Storage` に格納されるので、認証機構に追記が必要。

```bash
vi app/controllers/active_storage/base_controller.rb
```

```diff ruby
def authorized?
  redirect_to "/" unless current_user

  allow = @blob.attachments.any? do |attachment|
    type = attachment.record_type
    logger.debug type
    id   = attachment.record_id
    record = type.constantize.find(id)
+    if record.class.to_s == "ActionText::RichText"
+      record = record.record
+    end
    record.allow?(current_user)
  end

  redirect_to "/" unless allow
end
```

次（予定）

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-4>
