---
title: "Rails で文書管理アプリのようなもの(2)"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。

## 前提

最低限のモデルの作成が終わっている状態です。

<https://zenn.dev/asterisk9101/articles/rails8-1>

## Active Storage のインストール

```bash
bundle exec rails active_storage:install
bundle exec rails db:migrate
```

保存先は `config/storage.yml` などで設定しますが、そのままでもサーバのローカルディスクに保存されるので、そのまま進みます。

## 機密性の確保

各所で説明されている通り、`Active Storage` に保存したファイルは公開されます。

<https://railsguides.jp/active_storage_overview.html#%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E9%85%8D%E4%BF%A1%E3%81%99%E3%82%8B>

今回は機密データを扱う想定なので、プロキシモードを使います。

```ruby
# config/initializers/active_storage.rb
Rails.application.config.active_storage.resolve_model_to_route = :rails_storage_proxy
```

ただし、このままでは、`before_action :authorized_user!` が効いていないので、ログインしなくてもファイルにアクセスが可能です。

ガイドに記載されている通り、独自にアクセス制御を行うためコントローラを実装します。

## コントローラの実装

どのように実装するのか具体的な方法が分からなかったのでログを眺めました。

ファイルをダウンロードするときのログは、以下のようになっていました。

```log
14:31:11 web.1  | Started GET "/rails/active_storage/blobs/proxy/eyJfcmFpbHMiOnsiZGF0YSI6OSwicHVyIjoiYmxvYl9pZCJ9fQ==--6b8f936593a145cb34cc058821d43b87600afe25/sample.png" for 192.168.1.133 at 2025-02-16 14:31:11 +0900
14:31:11 web.1  | Processing by ActiveStorage::Blobs::ProxyController#show as PNG
14:31:11 web.1  |   Parameters: {"signed_id"=>"eyJfcmFpbHMiOnsiZGF0YSI6OSwicHVyIjoiYmxvYl9pZCJ9fQ==--6b8f936593a145cb34cc058821d43b87600afe25", "filename"=>"sample"}
14:31:11 web.1  |   ActiveStorage::Blob Load (0.5ms)  SELECT "active_storage_blobs".* FROM "active_storage_blobs" WHERE "active_storage_blobs"."id" = 9 LIMIT 1 /*action='show',application='MyApp',controller='proxy'*/
14:31:11 web.1  |   Disk Storage (1.1ms) Downloaded file from key: toa1oroek9xclbhb1quj0b8dkuat
14:31:11 web.1  | Completed 200 OK in 15ms (ActiveRecord: 0.5ms (1 query, 0 cached) | GC: 0.2ms)
```

`/rails/active_storage/blobs/proxy/...` に GET リクエストが飛んでいることが分かります。

一方で `bin/rails routes | grep active_storage` の結果は以下の通りです。

```bash
              rails_service_blob GET    /rails/active_storage/blobs/redirect/:signed_id/*filename(.:format)                               active_storage/blobs/redirect#show
        rails_service_blob_proxy GET    /rails/active_storage/blobs/proxy/:signed_id/*filename(.:format)                                  active_storage/blobs/proxy#show
                                 GET    /rails/active_storage/blobs/:signed_id/*filename(.:format)                                        active_storage/blobs/redirect#show
       rails_blob_representation GET    /rails/active_storage/representations/redirect/:signed_blob_id/:variation_key/*filename(.:format) active_storage/representations/redirect#show
 rails_blob_representation_proxy GET    /rails/active_storage/representations/proxy/:signed_blob_id/:variation_key/*filename(.:format)    active_storage/representations/proxy#show
                                 GET    /rails/active_storage/representations/:signed_blob_id/:variation_key/*filename(.:format)          active_storage/representations/redirect#show
              rails_disk_service GET    /rails/active_storage/disk/:encoded_key/*filename(.:format)                                       active_storage/disk#show
       update_rails_disk_service PUT    /rails/active_storage/disk/:encoded_token(.:format)                                               active_storage/disk#update
            rails_direct_uploads POST   /rails/active_storage/direct_uploads(.:format)                                                    active_storage/direct_uploads#create
```

`rails_service_blob_proxy` のルートが該当することが分かります。このルートから呼び出されるコントローラは `active_storage/blobs/proxy` です。

定義されているファイル名を想像すると、`proxy_controller.rb` です。

プロジェクトのコントローラのディレクトリ `app/controllers/` には、そのようなファイルは無いので `~/.local` を検索します。

```bash
find ~/.local -name proxy_controller.rb
#/home/.../.local/share/gem/ruby/3.3.0/gems/activestorage-8.0.1/app/controllers/active_storage/blobs/proxy_controller.rb
#/home/.../.local/share/gem/ruby/3.3.0/gems/activestorage-8.0.1/app/controllers/active_storage/representations/proxy_controller.rb
```

ファイルが２つ見つかりました。

`representations` の方は、加工したファイルを管理するためのコントローラということなので、今は不要です。

`blobs` の方のコントローラをプロジェクトのディレクトリにコピーすると、`.local` に置いてあるコントローラよりも優先して使用されます。

```bash
mkdir -p app/controllers/active_storage/blobs/
src=~/.local/share/gem/ruby/3.3.0/gems/activestorage-8.0.1/app/controllers/active_storage/blobs/proxy_controller.rb
dst=app/controllers/active_storage/blobs/proxy_controller.rb
cp -p $src $dst
```

コピーなので、そのままでも正常に動くことを確認します。

```ruby
# app/controllers/active_storage/blobs/proxy_controller.rb
# ...
class ActiveStorage::Blobs::ProxyController < ActiveStorage::BaseController
  include ActiveStorage::SetBlob
  include ActiveStorage::Streaming
  include ActiveStorage::DisableSession

  def show
    if request.headers["Range"].present?
      send_blob_byte_range_data @blob, request.headers["Range"]
    else
      http_cache_forever public: true do
        response.headers["Accept-Ranges"] = "bytes"
        response.headers["Content-Length"] = @blob.byte_size.to_s

        send_blob_stream @blob, disposition: params[:disposition]
      end
    end
  end
end
```

正常に動くことを確認してから、認証機構を追記します。

```ruby
# app/controllers/active_storage/blobs/proxy_controller.rb
# root_path が効いてなかったので include も追加
include Rails.application.routes.url_helpers
before_action :authorized?

# private
def authorized?
    redirect_to root_path unless current_user

    allow = @blob.attachments.any? do |attachment|
        type = attachment.record_type
        id   = attachment.record_id
        record = record_type.constantize.find(id)
        record.allow?(current_user)
    end

    redirect_to root_path unless allow
end
```

`app/models/document.rb` に追記します。`true` を返すようにしておき、後ほど実装することにします。

```ruby
def allow?(user)
  true
end
```

`allow?` が返す値を `false` にしたりすると、`root_path` にリダイレクトされることを確認します。

`representations` のコントローラを持ってくるとか、認証機構を `concerns` に置いたりなど、まだまだ改善の余地はあると思いますが理解が追いつかなくなってきたので一旦よしとします。

## 文書モデルにファイルを添付する機能追加

モデルに関連を追加します。

```ruby
# app/models/document.rb
has_many_attached :attachments
```

ビューに追記して、添付ファイルを表示、アップロードできるようにします。削除機能は省略。

```erb
<%# app/views/documents/_document.html.erb %>
<p>
  <strong>Attachments:</strong>
  <% document.attachments.each do |file| %>
    <li><%= link_to file.filename.to_s, rails_blob_path(file) %></li>
  <% end %>
</p>
```

```erb
<%# app/views/documents/_form.html.erb %>
<div>
  <%# 既存の添付ファイルを維持するための hidden_field %>
  <% document.attachments.each do |file| %>
    <%= form.hidden_field :attachments, multiple: true, value: file.signed_id %>
  <% end %>

  <%# 新しい添付ファイルを追加するための file_field %>
  <%= form.label :attachements, style: "display: block" %>
  <%= form.file_field :attachments, multiple: true %>
</div>
```

コントローラがファイルを受け取れるように修正します。

```diff ruby
# app/models/document_controller.rb
- params.expect(document: [ :name, :status, :due_date, :section_id ])
+ params.expect(document: [ :name, :status, :due_date, :section_id, attachments: [] ])
```

Document に複数のファイルを添付できることを確認します。

次（予定）

<https://zenn.dev/asterisk9101/articles/rails8-3>
