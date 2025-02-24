---
title: "Rails アプリに Active Storage を追加してセキュリティも確保する"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。

## 前提

最低限のモデルの作成が終わっている状態です。

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-1>

ストレージとして `MinIO` をセットアップしておきます。

<https://zenn.dev/asterisk9101/articles/fedora41server-6>

## Active Storage のインストール

`Active Storage` をインストールします。

```bash
bundle exec rails active_storage:install
bundle exec rails db:migrate
```

## ストレージの設定

`MinIO` は `Amazon S3` と互換性があるので、`aws-sdk-s3` を追加することで利用できます。

```bash
bundle add aws-sdk-s3
```

`MinIO` の接続情報を設定します。

```bash
read -p 'minioserver?> ' minioserver
```

```bash
read -p 'bucket name?> ' BUCKET_NAME
```

```bash
read -s -p 'ACCESS KEY?> ' ACCESS_KEY
```

```bash
read -s -p 'SECRET KEY?> ' SECRET_KEY
```

```bash
cat << EOF >> config/storage.yml
minio:
   service: S3
   access_key_id: $ACCESS_KEY
   secret_access_key: $SECRET_KEY
   region: auto
   bucket: $BUCKET_NAME
   endpoint: http://${minioserver}:9000

   # S3 互換ストレージを使用する場合に必須のパラメータ
   # 本物 S3 の場合は不要とのこと
   force_path_style: true
EOF

# 確認
cat config/storage.yml
```

ストレージとして `MinIO` を使うように、`Active Storage` を設定します。

```bash
tgt="config.active_storage.service = :local"
rep="config.active_storage.service = :minio"
sed -i "s/${tgt}/${rep}/" config/environments/development.rb
```

## 文書モデルにファイルを添付する機能追加

`Active Storage` を使うようにモデルに関連を追加します。

```bash
vi app/models/document.rb
```

```ruby
has_many_attached :attachments
```

ビューに追記して、添付ファイルを表示、アップロードできるようにします。削除機能は省略。

```bash
vi app/views/documents/_document.html.erb
```

```erb
<p>
  <strong>Attachments:</strong>
  <% document.attachments.each do |file| %>
    <li><%= link_to file.filename.to_s, rails_blob_path(file) %></li>
  <% end %>
</p>
```

```bash
vi app/views/documents/_form.html.erb
```

```erb
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

```bash
vi app/controllers/documents_controller.rb
```

```diff ruby
- params.expect(document: [ :name, :status, :due_date, :section_id ])
+ params.expect(document: [ :name, :status, :due_date, :section_id, attachments: [] ])
```

Document に複数のファイルを添付できることを確認します。

## 機密性の確保

各所で説明されている通り、`Active Storage` に保存したファイルは公開されます。

<https://railsguides.jp/active_storage_overview.html#%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E9%85%8D%E4%BF%A1%E3%81%99%E3%82%8B>

今回は機密データを扱う想定なので、`Active Storage` の設定ファイルを新規作成し、プロキシモードを使うよう記載します。

```bash
vi config/initializers/active_storage.rb
```

```ruby
Rails.application.config.active_storage.resolve_model_to_route = :rails_storage_proxy
```

ただし、このままでは、`before_action :authorized_user!` など、認証・認可機能が効いていないので、ログインしなくてもファイルにアクセスが可能です。

ガイドに記載されている通り、独自にアクセス制御を行うためコントローラを実装します。

## コントローラの実装

どのように実装するのか具体的な方法が分からなかったのでログを眺めました。

ファイルをダウンロードするときのログは、以下のようになっていました。

```log
16:22:56 web.1  | Started GET "/rails/active_storage/blobs/proxy/eyJfcmFpbHMiOnsiZGF0YSI6NCwicHVyIjoiYmxvYl9pZCJ9fQ==--87f354a3bb32f70e9b0d9b9d76a286f3aeec1bf0/a.xlsx" for 192.168.1.133 at 2025-02-22 16:22:56 +0900
16:22:56 web.1  | Processing by ActiveStorage::Blobs::ProxyController#show as */*
16:22:56 web.1  |   Parameters: {"signed_id"=>"eyJfcmFpbHMiOnsiZGF0YSI6NCwicHVyIjoiYmxvYl9pZCJ9fQ==--87f354a3bb32f70e9b0d9b9d76a286f3aeec1bf0", "filename"=>"a"}
16:22:56 web.1  |   ActiveStorage::Blob Load (0.1ms)  SELECT "active_storage_blobs".* FROM "active_storage_blobs" WHERE "active_storage_blobs"."id" = 4 LIMIT 1 /*action='show',application='MyApp',controller='proxy'*/
16:22:56 web.1  |   S3 Storage (33.1ms) Downloaded file from key: 74midv5cakerpmqfpjyhulouimft
16:22:56 web.1  | Completed 200 OK in 45ms (ActiveRecord: 0.1ms (1 query, 0 cached) | GC: 9.3ms)
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

`app` 配下のコントローラは、`.local` に置いてあるコントローラよりも優先して使用されます。コピーして使います。

```bash
src=~/.local/share/gem/ruby/3.3.0/gems/activestorage-8.0.1/app/controllers/
dst=app/
cp -frp $src $dst
```

コピーなので、そのままでも正常に動くことを確認します。

正常に動くことを確認してから、認証機構を追記します。

```bash
vi app/controllers/active_storage/base_controller.rb
```

```ruby
private
def authorized?
    redirect_to "/" unless current_user

    allow = @blob.attachments.any? do |attachment|
        type = attachment.record_type
        id   = attachment.record_id
        record = type.constantize.find(id)
        record.allow?(current_user)
    end

    redirect_to "/" unless allow
end
```

`base_controller.rb` を継承している各コントローラーに `before_action` を追加します。

```bash
# 多分これだけ追加すれば良いと思う
vi app/controllers/active_storage/disk_controller.rb
vi app/controllers/active_storage/blobs/redirect_controller.rb
vi app/controllers/active_storage/blobs/proxy_controller.rb
vi app/controllers/active_storage/representations/proxy_controller.rb
vi app/controllers/active_storage/representations/redirect_controller.rb
```

```ruby
before_action :authorized?
```

モデルに追記します。`true` を返すようにしておき、後ほど実装することにします。

```bash
vi app/models/document.rb
```

```ruby
def allow?(user)
  true
end
```

`allow?` が返す値を `false` にしたりすると、ルートにリダイレクトされることを確認します。

`representations` のコントローラを持ってくるとか、認証機構を `concerns` に置いたりなど、まだまだ改善の余地はあると思いますが理解が追いつかなくなってきたので一旦よしとします。

次

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-3>
