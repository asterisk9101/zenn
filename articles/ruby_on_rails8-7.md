---
title: "Rails に Pundit を導入して権限管理できるようにする(2)"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。

## 前提

前提です。

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-6>

## 文書の編集権限を管理するモデル

文書モデルのアクセス制御をするために管理するモデルを作ります。

```bash
bundle exec rails g model stakeholder \
    user:references \
    document:references

bundle exec rails db:migrate
bundle exec rails g controller stakeholders create destroy
```

モデルを関連付けます。

```bash
vi app/models/document.rb
```

```ruby
has_many :stakeholders
has_many :documents, through: :stakeholders
```

```bash
vi app/models/user.rb
```

```ruby
has_many :stakeholders
has_many :users, through: :stakeholders
```

ビューから入力できるようにします。

```bash
vi app/views/documents/_form.html.erb
```

```erb
<div>
  <%= form.label :stakeholders, style: "display: block" %>
  <%= form.text_field :stakeholders, value: documents.users.pluck(:email).join("; ") %>
</div>
```

ビューから表示するようにします。

```bash
vi app/views/documents/_document.html.erb
```

```erb
<div>
  <strong>Stakeholders:</strong>
  <%= @document.users.pluck(:email).join("; ") %>
</div>
```

文書モデルに stakeholders を更新するメソッドを生やします。

```bash
vi app/models/document.rb
```

```ruby
def update_stakeholders(joined_emails)
  registerd_emails = self.users.pluck(:email)
  emails = joined_emails.split(";").map(&:strip)

  add_users = User.where(email: emails.difference(registerd_emails))
  del_users = User.where(email: registerd_emails.difference(emails))

  self.users << add_users
  self.stakeholders.where(user_id: del_users.pluck(:id)).destroy_all
end
```

文書コントローラに、ビューから受け取ったパラメータで関係者を更新するメソッドを追加します。

```bash
vi app/controllers/document_controller.rb
```

```ruby
# create と update メソッドに追記
@document.update_stakeholders(params[:document][:stakeholders]+";"+current_user.email)
```

## 文書は stakeholders に含まれるユーザーにだけ開示する

文書モデルのポリシーを作ります。

```bash
vi app/policies/document_policy.rb
```

```ruby
class DocumentPolicy < ApplicationPolicy
  class Scope < ApplicationPolicy::Scope
    def resolve
      if user.id == 1
        scope.all
      else
        user.documents
      end
    end
  end

  def index?
    admin? || section_public? || section_member?
  end

  def show?
    admin? || section_public? || document_stakeholder?
  end

  def create?
    admin? || section_public? || section_member?
  end

  def update?
    admin? || document_stakeholder?
  end

  def destroy?
    admin?
  end

  private
    def admin?
      user.id == 1
    end

    def section_public?
      not record.section.private
    end

    def section_member?
      record.section.users.include?(user)
    end

    def document_stakeholder?
      record.users.include?(user)
    end
end
```

文書コントローラにポリシーを仕込みます。

```bash
vi app/controllers/document_controller.rb
```

```ruby
def index
  authorize Document
  @documents = policy_scope(Document)
end

def show
  authorize @document
  # ...
def create
  @document = Document.new(document_params)
  authorize @document
  # ...

def update
  authorize @document
  # ...

def destroy
  authorize @document
  # ...
```

テストが欲しくなってきましたが、全体のイメージが掴めるまで突き進むことにします。

次

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-8>
