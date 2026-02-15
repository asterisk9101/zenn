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
<%= form.label :content, style: "display: block" %>
<%= form.rich_text_area :content%>
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
<div class="lexxy-content">
```

`bin/dev` して `trix` が `lexxy` に置き換わったことを確認します。

## コードハイライトが動かない

`lexxy` は2026年現在 early beta のせいか、エディタ上ではハイライトされるのに、ビューに戻るとコードハイライトが機能してないような気がします。

調べてみると `ActionText` で保存されたデータは、HTMLタグも含めて保存されており、コード部分は `<pre data-language="****">`、その中は改行が `<br>` などになっています。

一方で、コーディングエージェントに調べによると `lexxy` では、`prism.js` が使われているとのことでした。`prism.js` を素直に使う場合は、ハイライトしたい部分を `<code class="language-****">` に入れてやる必要があるらしいです。

そこで、`<pre>` タグが描画された後に、`javascript` でコード部分を抜き出して `<code>` タグを差し込んでみます。

---

まず `prism.js` と `prism.css` は、別途ダウンロードしてプロジェクトの `public/` に配置します。

レイアウトビューからそれらを参照します。

```erb:app/views/layouts/application.html.erb
<head>
  <link href="/prism.css" rel="stylesheet">
  <scritp src="/prism.js"></script>
</head>
```

それから `stimulus` のコントローラを作ります。`connect` は作成したコントローラが DOM 上の要素に接続されたときに呼ばれるメソッドらしいです。

```javascript:app/javascript/controllers/prism_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    Array.from(document.getElementsByTagName("pre")).forEach(function(pre) {
      // 言語判定
      const lang = pre.dataset.language
      if (lang === undefined) { return } // 未定義の場合は何もしない

      // ActionText は改行を HTML タグとして保存してしまうので
      // pre 配下のテキストノードだけ抽出して改行で結合する
      const preChildNodes = Array.from(pre.childNodes);
      const textNodes = preChildNodes.filter(node => node.nodeType === Node.TEXT_NODE);
      const textContent = textNodes.map(node => node.nodeValue).join("\n");

      // code 要素を作る
      const code = document.createElement("code");
      code.textContent = textContent
      switch(lang) {
        case "js":  code.classList.add("language-javascript"); break;
        default:    code.classList.add("language-" + lang); break;
      }

      // pre をクリアしてから code を載せる
      pre.textContent = ""; // clear
      pre.classList.add("line-numbers")
      pre.appendChild(code);
    });

    // Prism.js によるハイライトの処理
    if (window.Prism) { Prism.highlightAll() }
  }
}
```

ビューの方でコントローラを接続します。

```erb:app/views/posts/_post.html.erb
<div data-controller="prism" ...>
  ...
</div>
```

理解が及んでない部分がありますが以上です。

## 参考

<https://basecamp.github.io/lexxy/installation.html>

<https://railsguides.jp/action_text_overview.html>

<https://prismjs.com/>
