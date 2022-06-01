---
title: "Railsで子要素の件数集計のN+1問題対処"
date: 2022-05-06T21:37:06+09:00
categories:
- Ruby on Rails
tags:
- 技術課題
keywords:
- N+1
- count
- group
---
## 概要
子要素を条件別にカウントして一覧表示する際に、N+1解消するのに苦戦しました。

もっとシンプルな書き方がありましたら、コメントなどでご指導いただきたいです。

## 実現したいこと
以下のテーブルのように、子要素のステータス別の件数を一覧にしたい
|要件名|チケットの枚数(未実施/実施中/完了)|
|----|----|
|要件1|0 / 0 / 10|
|要件2|5 / 1 / 3|
|...|...|

## 普通に書いたやつ(N+1)
- モデルでカウントする関数を作る
```ruby:app/models/ticket.rb
class Ticket < ApplicationRecord
  belongs_to :requirement
  enum :status, %i[notyet doing done]
end
```
```ruby:app/models/requirement.rb
class Requirement < ApplicationRecord
  has_many :tickets

  def count_tickets_by_status(status)
    tickets.where(status:).count
  end

  def count_all_tickets_by_status
    Ticket.statuses.keys.map { |s| count_tickets_by_status(s) }.join(' / ')
  end
end
```
```ruby:app/controllers/requirements_controller.rb
  ...
  def index
    @requirements = current_user.requirements
  end
```
```haml:app/views/requirements/index.html.haml
%table.table
  ...
  - @requirements.each do |requirement|
    = requirement.count_all_tickets_by_status
```

当然、count_tickets_by_statusが呼び出されるたびにSQLが発行される(N+1どころか3N+1)

## 考えた2つの案
### 一つ一つselect(count)したテーブルをjoinしてやる
- かなり複雑化してしまった
- 同じテーブルを何度もjoinしていてやばい
- SQLインジェクション的にもよろしくない(任意の文字受け付けたらアウト)
```ruby:app/models/ticket.rb
class Ticket < ApplicationRecord
  ...
  scope :add_count_by_requirements, ->(column_name) {
    group(:requirement_id)
      .select("requirement_id, count(*) as #{column_name}")
  }
end
```
```ruby:app/models/requirement.rb
class Requirement < ApplicationRecord
  ...
  scope :join_count_if_status, ->(status) {
    joins(%(LEFT JOIN (#{Ticket.where(status:).add_count_by_requirements("count_#{status}").to_sql})
          as sub_count_#{status}
          ON requirements.id = sub_count_#{status}.requirement_id))
      .select('*')
  }

  scope :join_counts, -> {
    join_count_if_status(:notyet)
    .join_count_if_status(:doing)
    .join_count_if_status(:done)
  }

  def count_all_tickets_by_status
    "#{count_notyet} / #{count_doing} / #{count_done}"
  end
end
```

```ruby:app/controllers/requirements_controller.rb
  ...
  def index
    @requirements = current_user.requirements.join_counts
  end
```

### 一つ一つアソシエーションを作ってしまう
- 意外とシンプルに行けた
   - 通常のカウントと同様にincludesとsizeで行ける

```ruby:app/models/ticket.rb
class Ticket < ApplicationRecord
  belongs_to :requirement
end
```
```ruby:app/models/requirement.rb
class Requirement < ApplicationRecord
  has_many :tickets
  has_many :tickets_notyet, -> { where(status: :notyet) }, class_name: 'Ticket'
  has_many :tickets_doing, -> { where(status: :doing) }, class_name: 'Ticket'
  has_many :tickets_done, -> { where(status: :done) }, class_name: 'Ticket'
end
```

```ruby:app/controllers/requirements_controller.rb
  ...
  def index
    @requirements = current_user.requirements
                                .includes(:tickets_notyet)
                                .includes(:tickets_doing)
                                .includes(:tickets_done)
  end
```

```haml:app/views/requirements/index.html.haml
%table.table
  ...
  - @requirements.each do |r|
    %tr
      %td
        r.name
      %td
        #{r.tickets_notyet.size} / #{r.tickets_doing.size} / #{r.tickets_done.size}
```

## まとめ
どちらもステータスの種類を増やしたときにはメンテが必要だが、後者の方が飛び道具を使っていない分、分かりやすいと思う
