---
title: "Railsの複数データベースでconnected_toを使う際はクエリの実行タイミングに注意しよう"
emoji: "🐰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Rails]
published: true
---

Railsの複数データベース機能利用時に意図せずに無駄なクエリ(SQL)を実行しているケースに遭遇したため、そのことについて書いていきます。

# TL;DR
`connected_to`のブロックで`ActiveRecord::Relation`を返す場合、その時点でクエリが実行される。
そのため以下のように`connected_to`のブロックの戻り値に対してリレーションを追加して絞り込みを行うような書き方は無駄にSQLを実行してしまうため基本的には避けた方がいい。


```ruby
relation = ActiveRecord::Base.connected_to(role: :reading) do
             HogeModel.where(condition1)
           end # この時点で HogeModel.where(condition1) の条件でSQLが実行される
relation.where(condition2)
```


# 前提知識
## Active Recordの遅延実行
Active Recordはそのデータが必要になるタイミングまでクエリを実行しないようにする遅延実行の仕組みがあります。
以下のようなコードの場合、SQLが実行されるのはeachのタイミングです。

```ruby
users = User.where(name: 'hoge')
users.each do |user|
  # 何か処理
end
```

また、以下のように条件をつなげたとしてもSQLが実行されるのはeachのタイミングの1度です。

```ruby
users = User.where(name: 'hoge')
users = users.where(status: 'active')
users.each do |user|
  # 何か処理
end
```

## connected_toによるコネクションの手動切り替え
詳しくはRailsガイドの[コネクションを手動で切り替える](https://railsguides.jp/active_record_multiple_databases.html#%E3%82%B3%E3%83%8D%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%92%E6%89%8B%E5%8B%95%E3%81%A7%E5%88%87%E3%82%8A%E6%9B%BF%E3%81%88%E3%82%8B)を見てもらうのがいいですが、簡単にふれます。

Railsアプリケーションから複数のDBに接続する場合に、`connected_to`メソッドで処理中にどのDBに接続するかを個別に選択することができます。
例えば書き込み用のwriter DBと読み見込み用のreplica DBがあり、特定の処理でreplica DBに対してクエリを実行したい場合は以下のようにすることで実現できます。


```ruby
ActiveRecord::Base.connected_to(role: :reading) do
  # このブロック内のコードはすべてreadingロール(replica DB)に対してSQLが実行される
  User.where(name: 'hoge')
end
```


# connected_to使用時に発生し得る問題
以下のようなコードでユーザーを絞り込んでSQLを実行する場合、 `to_a` でのタイミングで実行されます。

```ruby
users = User.where(status: 'active')
# usersには ActiveRecord::Relation が返り、そこにさらに条件を追加する
users = users.where(name: 'hoge').limit(10)

users.to_a # このタイミングでSQLが実行される
```

このクエリをreplica DBに対し実行するために


```ruby
users = ActiveRecord::Base.connected_to(role: :reading) do
          User.where(status: 'active')
        end # この時点でSQLが実行される
users = ActiveRecord::Base.connected_to(role: :reading) do
          users.where(name: 'hoge').limit(10)
        end # この時点でSQLが実行される

users.to_a
```

のように書き換えたとします。

`to_a`のタイミングでのみSQLを実行してほしいですが、一つ目の`connected_toブ`ロックを抜ける際に`User.where(status: 'active')`の条件でSQLが実行されてしまいます。

本来は`where(name: 'hoge').limit(10)`をつけて取得するレコードを絞って実行したいのにactiveなユーザーを全て取ってくるようなSQLが実行されます。usersテーブルが巨大だった場合にはかなり重たいSQLが実行されることが想定されます。

このような挙動となるのは`connected_to`は、ブロックで`ActiveRecord::Relation`を返す場合、**その時点で強制的にクエリを実行する**ためです。
ref: https://github.com/rails/rails/pull/38339

最終的な条件でのみSQLを実行したい場合は以下のようにブロック内でクエリの組み立てを完成さる必要があります。


```ruby
users = ActiveRecord::Base.connected_to(role: :reading) do
          relation = User.where(status: 'active')
          relation.where(name: 'hoge').limit(10)
        end # この時点でSQLが実行される

users.to_a
```

この例のようにシンプルな処理であれば、わざわざ`connected_to`を複数に分けて書くようなことはしないかもしれません。しかし、複雑なリレーションを組み立てる時などは注意が必要です。

実際に私自身が関わっているシステムで発生しました。`connected_to`を使う際は気をつけましょう。
