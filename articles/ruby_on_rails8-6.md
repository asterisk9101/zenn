---
title: "Rails に Pundit を導入して権限管理できるようにする(1)"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, Rails, memo]
published: true
---

お家の検証サーバ用の備忘録です。

## 前提

前提です。

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-5>

## pundit導入

```bash
bundle add pundit
bundle exec rails g pundit:install
```

各コントローラで使えるように `ApplicationController` でインクルードします。

```bash
vi app/controllers/application_controller.rb
```

```ruby
class ApplicationController < ActionController::Base
  include Pundit::Authorization
# ...
```

## 部署への参加・脱退は自分の分だけに制限するポリシー

Membership コントローラーに `Authorize` を仕込みます。

```bash
vi app/controllers/memberships_controller.rb
```

```ruby
def create
  @membership = Membership.new(membership_params)
  authorize @membership
  # ...

def destroy
  authorize @membership
  # ...
```

ポリシーを作ります。

```bash
vi app/policies/membership_policy.rb
```

```ruby
class MembershipPolicy < ApplicationPolicy
  def create?
    admin? || (self? && public?)
  end

  def destroy?
    admin? || self?
  end

  private
    def admin?
      user.id == 1
    end

    def self?
      user.id == record.user.id
    end

    def public?
      not record.section.private?
    end
end
```

次

<https://zenn.dev/asterisk9101/articles/ruby_on_rails8-7>
