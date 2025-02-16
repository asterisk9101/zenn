---
title: "Rails で文書管理アプリのようなもの(1)"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。

## 前提

適当なページが表示できるようになった状態です。

<https://zenn.dev/asterisk9101/articles/fedora41server-5>

## 目標

どこかで見たことがある文書管理アプリを作ってみます。

網羅性のないざっくり要件は以下。

- 部署(Section)
  - 各リソースを管理する入れ物です
  - 公開されていて一覧表示可能な場合もあり、非公開で招待のみ表示可能な場合もあります
  - セキュリティの境界です（Punditの練習）
- 文書(Document)
  - 部署に所属します（referencesの練習）
  - リッチテキストのコンテンツを持ちます（ActionTextの練習）
  - 複数のファイルを添付できます（ActiveStorageの練習）
  - 現在のステータスを持ちます（状態遷移をしたい、AASM のようなgemは使わない独自）
  - 公開部署に所属する場合は、部署に所属していないユーザーも参照することができます
  - 機密性があり、部署内であっても厳密に閲覧制限を設定できます（Punditの練習）
  - コメントを追加できます（hotwire, bootstrap の modal の練習、画面遷移なしで追加したい）
  - 一覧は csv でダウンロードできます(そういうgemある？)
- コメント(Activity)
  - 文書に従属します
  - ユーザーが文書に対して行ったアクションを記録する監査履歴の役割があります
  - コメントしたユーザー名や役職などを固定値として持ちます
  - 削除したり編集はできません（監査ログなので）
  - 非表示や非活性にすることはできた方が良いかも
- ワークフロー(Workflow)
  - 部署に従属します
  - 文書の状態遷移を表します
  - 部署の管理者が任意に作成できます
  - 複数のアクションで構成されます
- アクション(Action)
  - ワークフローに従属します
  - 元の状態と遷移後の状態を定義し、アクション同士の前後関係はありません。
  - 実行できる担当者を定義します
  - セキュリティの境界です（Punditの練習）
  - アクションに伴ってメールを送ることができます（ActionMailerの練習）

思ったよりたくさん出てきてしまったので調整するかも。風呂敷広げすぎてもちょっとねぇ。

権限管理は難しそうなので後ほど実装します。

## モデルの作成

scaffold でコントローラとビューもまとめて作成します。

```bash
# 部署
bundle exec rails g scaffold Section \
  name:string:uniq \
  description:text \
  private:boolean

# 文書
bundle exec rails g scaffold Document \
  name:string \
  status:string \
  due_date:datetime \
  section:references

bundle exec rails db:migrate
```

## サンプルデータの投入

イメージしやすくするために、開発用のサンプルデータを用意します。

開発環境用のサンプルデータと、システム上必要な admin は別ファイルで管理したいので、環境別のシードファイルを用意します。

```bash
mkdir db/seeds/
touch db/seeds/development.rb
```

`db/seeds/development.rb` に追記します。

```ruby
10.times do |index|
  Section.create!({
    name: SecureRandom.alphanumeric(10),
    description: SecureRandom.alphanumeric(300),
    private: false
  })
end

100.times do |index|
  Document.create!({
    name: SecureRandom.alphanumeric(20),
    status: SecureRandom.alphanumeric(10),
    due_date: Date.today + rand(1..1000),
    section: Section.find(rand(1..10))
  })
end
```

個別の seed ファイルを読み込むために `db/seeds.rb` に追記します。

```ruby
seed_file = Rails.root.join('db', 'seeds', "#{Rails.env}.rb")
load seed_file if File.exist?(seed_file)
```

データを投入します。

```bash
# 必要があれば DB をリセットする
# bundle exec rails db:migrate:reset
bundle exec rails db:seed
```

## ホームページから部署の一覧を参照できるようにする

開発中は無くても良いのですが、導線がある方がイメージしやすいので `app/view/home/index.html.erb` に追記します。

```bash
<p>
  <%= link_to "Sections", sections_path %>
</p>
<p>
  <%= link_to "Documents", documents_path %>
</p>
```

`http://{サーバのIPアドレス}:3000/` にリンクが２つ追加されていることを確認します。

リンクから `http://{サーバのIPアドレス}:3000/sections` や `http://{サーバのIPアドレス}:3000/documents` にアクセスして、データが表示されることを確認します。

## モデルの調整

scaffold で自動的に作成されたモデルを調整します。

`app/views/section.rb` に追記します。

```bash
# @section.documents で所属しているドキュメントのリストを取得できるようにします
# 部署を削除したらドキュメントも削除されるようにします
has_many :documents, dependent: :destroy
```

## 部署の文書を一覧できるようにする

`app/controller/sections_controller.rb` の `show` メソッドに追記します。

```ruby
@documents = @section.documents
```

`app/views/sections/show.html.erb` に追記します。

```erb
<hr>
<% @documents.each do |document| %>
  <%= render partial: "documents/document", locals: { document: document } %>
  <p>
    <%= link_to "Show Detail", document %>
  </p>
<% end %>
<hr>
```

また、文書の画面から戻るリンクは、文書の一覧ではなく、部署の一覧に戻る動作に変更しておきます。

```diff
- <%= link_to "Back to documents", documents_path %>
+ <%= link_to "Back to Section", section_path(@document.section_id) %>
```

リンクから `http://{サーバのIPアドレス}:3000/sections/1` などにアクセスすると、部署の情報とともに文書の一覧が表示されます。

次

<https://zenn.dev/asterisk9101/articles/rails8-2>
