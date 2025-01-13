---
title: "【Rails】ActiveRecordのallow_retryを使ったクエリのリトライ処理の実装"
emoji: "🔨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Rails]
published: true
---

Railsの7.1でActiveRecordのクエリのリトライを可能とする `allow_retry` というオプションが内部のメソッドに追加されました。現時点では公開APIではないため、ドキュメントにはのっていませんがパッチを当てることで利用することはできます。

本記事では `allow_retry` オプションの以下について書いていきます。

- オプションが追加された経緯
- ユースケース
- モンキーパッチの例
- 挙動の確認
- Tips

## オプション追加の経緯

https://github.com/rails/rails/pull/46273
で追加されています。

> In order to opt-into retry behaviour, the entire #execute method needs to be reimplemented. If instead we allow #execute to take the allow_retry option, apps can patch #execute to call super with allow_retry: true.

と書かれているように、リトライ処理を実装する場合に `#execute` メソッド全体を再実装する必要があるが、オプションとして `allow_retry` を渡せるようにすることで、パッチを当てるだけで実現できるようにしたいとのことです。

また、公開APIとして実装していない理由として

> We discussed adding a config to allow apps to turn on retries across all queries, but decided that this was too risky a config to introduce to Rails, despite the fact that Shopify and Github applications have historically retried all queries without much deliberation. Changing #execute seems like a good compromise -- apps still have to knowingly patch #execute in order to get retry behaviour, but this way our patches don't need to reimplement the entire method, which makes them less brittle.

に書かれているように、Railsにconfigで設定できるようにするのはリスクが高いためのようです。
ただ、GithubやShopifyでは全てのクエリをリトライする運用の実績はあるようなので、多くのケースでは問題がないように思われます。
将来的に公開APIとなることもあるかもしれません。

## ユースケース
色んなケースがあると思いますが実際に自分が利用した例としては、データベースを負荷状況によってスケールイン/スケールアウトするような運用におけるリトライです。

**スケールイン中に停止しようとしているDBに対してクエリを実行しようとした場合にコネクションがロスト**してクエリ実行に失敗します。このような場合にコネクションを貼り直して残ったDBに対してリトライしたいといったケースです。

MySQLの場合は[activerecord-mysql-reconnect](https://github.com/winebarrel/activerecord-mysql-reconnect)というgemが便利でお世話になってしまいたがRails6.0までの対応で現在はPublic archiveとなっています。

現在もメンテナンスされていて利用できそうなgemが見つからなかったためallow_retryを使うパッチを当てて対応しました。

## モンキーパッチ
allow_retryオプションを使用するモンキーパッチの例です。


```ruby:config/initializers/database_statements_monkey_patch.rb
require 'active_record/connection_adapters/mysql2/database_statements.rb'

unless ActiveRecord.version == "7.1.2"
  raise "Consider removing this patch"
end

module DatabaseStatementsMonkeyPatch
  def raw_execute(sql, name, async: false, allow_retry: false, materialize_transactions: true)
    allow_retry = true
    super
  end
end

ActiveRecord::ConnectionAdapters::Mysql2::DatabaseStatements.prepend(DatabaseStatementsMonkeyPatch)
```
※ executeではなくraw_executeにパッチを当てるのは、[Refactor Mysql2Adapter and TrilogyAdapter](https://github.com/rails/rails/pull/48054)の修正で、内部のメソッドからexecuteを呼ばないようにする修正が入りました。この修正によってfind, where, create, upate, destroy など良く使われる公開メソッドで `execute` メソッドを経由しなくなったためです。

動作確認は以下で行っています。
- Rails: **7.1.2**
- Ruby: 3.2.2
- adapter: mysql2(0.5.5)
- MySQL: 8.0



### Rails8.0での変更について（2025/1/13 追記）
:::message
Rails8.0でリトライやraw_executeメソッドに関連するいくつかの修正が入っています。
:::

- https://github.com/rails/rails/pull/52428
  - リファクタリングによるraw_executeメソッドが実装されるモジュールの変更とraw_executeメソッドに引数（prepare, binds）追加
- https://github.com/rails/rails/pull/51336
  - Rails側で構築する安全と思われるクエリ（findで実行するselectクエリなど）はデフォルトでallow_retryがtrueとなる
- https://github.com/rails/rails/pull/52466
  - raw_executeメソッドにキーワード引数追加

それに伴いモンキーパッチにも修正が必要になります。
以下は修正例です。

```ruby:config/initializers/database_statements_monkey_patch.rb
require 'active_record/connection_adapters/abstract/database_statements.rb'

module DatabaseStatementsMonkeyPatch
  def raw_execute(sql, name = nil, binds = [], prepare: false, async: false, allow_retry: false, materialize_transactions: true, batch: false) # 引数の追加（binds, prepare, batch）
    allow_retry = true
    super
  end
end

ActiveRecord::ConnectionAdapters::DatabaseStatements.prepend(DatabaseStatementsMonkeyPatch) # prependするmoduleの変更
```

また、デフォルトで一部のselectクエリなどが[リトライされるようになった](https://github.com/rails/rails/pull/51336)ため、使い方によってはこのモンキーパッチ自体を削除してもよいかもしれません。

## コードの調査
上記のパッチで実際にクエリがリトライされるか確認します

### debug用コード
raw_executeメソッド内にdebuggerを仕込みます。

```diff ruby:lib/active_record/connection_adapters/mysql2/database_statements.rb
def raw_execute(sql, name, async: false, allow_retry: false, materialize_transactions: true)
+  debugger
  log(sql, name, async: async) do
    with_raw_connection(allow_retry: allow_retry, materialize_transactions: materialize_transactions) do |conn|
      sync_timezone_changes(conn)
      result = conn.query(sql)
      verified!
      handle_warnings(sql)
      result
    end
  end
end
```

### コード実行
rails cで適当なモデルで`find`を実行してみます

```ruby
User.find(1)
```

raw_executeメソッドのdebuggerで処理が止まるのでbacktraceを実行すると以下のようにfindから順にraw_executeメソッドが実行されるまでの経路が確認できます。

```ruby
# 該当のsqlを実行しようとしていることの確認
(rdbg) sql
"SELECT `users`.* FROM `users` WHERE `users`.`id` = 1 LIMIT 1"
(rdbg) backtrace
=>#0    ActiveRecord::ConnectionAdapters::Mysql2::DatabaseStatements#raw_execute(sql="SELECT `users`.* FROM `users` WHERE `use..., name="User Load", async=false, allow_retry=false, materialize_transactions=true) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/connection_adapters/mysql2/database_statements.rb:97
  #1    ActiveRecord::ConnectionAdapters::AbstractMysqlAdapter#execute_and_free(sql="SELECT `users`.* FROM `users` WHERE `use..., name="User Load", async=false) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:233
  #2    ActiveRecord::ConnectionAdapters::Mysql2::DatabaseStatements#internal_exec_query(sql="SELECT `users`.* FROM `users` WHERE `use..., name="User Load", binds=[], prepare=false, async=false) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/connection_adapters/mysql2/database_statements.rb:23
  #3    ActiveRecord::ConnectionAdapters::DatabaseStatements#select(sql="SELECT `users`.* FROM `users` WHERE `use..., name="User Load", binds=[], prepare=false, async=false) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/connection_adapters/abstract/database_statements.rb:630
  #4    ActiveRecord::ConnectionAdapters::DatabaseStatements#select_all(arel="SELECT `users`.* FROM `users` WHERE `use..., name="User Load", binds=[], preparable=true, async=false) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/connection_adapters/abstract/database_statements.rb:71
  #5    ActiveRecord::ConnectionAdapters::QueryCache#select_all(arel="SELECT `users`.* FROM `users` WHERE `use..., name="User Load", binds=[], preparable=true, async=false) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/connection_adapters/abstract/query_cache.rb:114
  #6    block {|conn=#<Mysql2::Client:0x0000ffff91e1fac8 @curr...|} in select_all at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/connection_adapters/mysql2/database_statements.rb:14
  #7    block in with_raw_connection at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/connection_adapters/abstract_adapter.rb:1028
  #8    ActiveSupport::Concurrency::NullLock#synchronize at /usr/local/bundle/gems/activesupport-7.1.2/lib/active_support/concurrency/null_lock.rb:9
  #9    ActiveRecord::ConnectionAdapters::AbstractAdapter#with_raw_connection(allow_retry=false, materialize_transactions=true) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/connection_adapters/abstract_adapter.rb:1000
  #10   ActiveRecord::ConnectionAdapters::Mysql2::DatabaseStatements#select_all at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/connection_adapters/mysql2/database_statements.rb:10
  #11   ActiveRecord::Querying#_query_by_sql(sql="SELECT `users`.* FROM `users` WHERE `use..., binds=[], preparable=true, async=false) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/querying.rb:62
  #12   ActiveRecord::Querying#find_by_sql(sql="SELECT `users`.* FROM `users` WHERE `use..., binds=[], preparable=true, block=nil) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/querying.rb:51
  #13   ActiveRecord::StatementCache#execute(params=[1], connection=#<ActiveRecord::ConnectionAdapters::Mysql..., block=nil) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/statement_cache.rb:150
  #14   ActiveRecord::Core::ClassMethods#cached_find_by(keys=["id"], values=[1]) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/core.rb:410
  #15   ActiveRecord::Core::ClassMethods#find(ids=[1]) at /usr/local/bundle/gems/activerecord-7.1.2/lib/active_record/core.rb:252
  #16   <main> at (irb):3
```

本記事では割愛しますが、createやupdateなどのCRUD用のメソッドを実行する場合も同様にraw_executeを通ります。
debuggerで中断した状態でMySQLを再起動した後に処理を再開してみるとクエリが実行されることを確認できます。

また、モンキーパッチを当てていない状態で同様の手順を行うと以下のエラーが発生するためパッチによってリトライが動作していることがわかります。

```
Mysql2::Error::ConnectionError: Lost connection to server during query (ActiveRecord::ConnectionFailed)
```

## retryされる条件は？
次にリトライされる条件を見ていきます。

リトライを行うかどうかはここでrescueした後の処理で行っています。
https://github.com/rails/rails/blob/6b93fff8af32ef5e91f4ec3cfffb081d0553faf0/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb#L1029

```ruby:activerecord/lib/active_record/connection_adapters/abstract_adapter.rb
begin
  yield @raw_connection # ⑦ 説明のための番号を振っています
rescue => original_exception
  translated_exception = translate_exception_class(original_exception, nil, nil) # ④
  invalidate_transaction(translated_exception)
  retry_deadline_exceeded = deadline && deadline < Process.clock_gettime(Process::CLOCK_MONOTONIC)

  if !retry_deadline_exceeded && retries_available > 0 # ①
    retries_available -= 1

    if retryable_query_error?(translated_exception) # ②
      backoff(connection_retries - retries_available)
      retry
    elsif reconnectable && retryable_connection_error?(translated_exception) # ③
      reconnect!(restore_transactions: true)
      # Only allowed to reconnect once, because reconnect! has its own retry
      # loop
      reconnectable = false
      retry # ⑥
    end
  end

  # 省略
end
```

前提として `allow_retry` オプションがtrueでない場合は `retries_available` が0になるためretryされずに終了します。（①）

その条件をクリアした上で以下のどちらかの場合にリトライしているようです。
- `retryable_query_error?(translated_exception)` がtrueの場合（②）
- `reconnectable && retryable_connection_error?(translated_exception)`がtrueの場合（③）

先ほどの動作確認の例で `Mysql2::Error::ConnectionError: Lost connection to server during query` が発生していたため、この例外がrescueされたとして処理を追ってみます。

`translated_exception` は  `translate_exception_class` の結果なので中身をみると `translate_exception` の結果が返ります。（④）

https://github.com/rails/rails/blob/6b93fff8af32ef5e91f4ec3cfffb081d0553faf0/activerecord/lib/active_record/connection_adapters/mysql2_adapter.rb#L184

```ruby
def translate_exception(exception, message:, sql:, binds:)
  if exception.is_a?(::Mysql2::Error::TimeoutError) && !exception.error_number
    ActiveRecord::AdapterTimeout.new(message, sql: sql, binds: binds, connection_pool: @pool)
  elsif exception.is_a?(::Mysql2::Error::ConnectionError)
    if exception.message.match?(/MySQL client is not connected/i)
      ActiveRecord::ConnectionNotEstablished.new(exception, connection_pool: @pool)
    else
      ActiveRecord::ConnectionFailed.new(message, sql: sql, binds: binds, connection_pool: @pool) # ⑤
    end
  else
    super
  end
end
```

`translate_exception` をみると今回発生した例外の場合は `ActiveRecord::ConnectionFailed` が返り `translated_exception` に入ることがわかります。(⑤)

元のコードの②に戻って `retryable_query_error?` の中をみると `Deadlocked` や `LockWaitTimeout` でリトライされます。今回の例外には当てはまりませんが、これらの例外でもリトライされることがわかります。

③に進んで `reconectable` は [reconnect_can_restore_state?](reconnect_can_restore_state?)(`transaction_manager.restorable? && !@raw_connection_dirty`) の結果になります。トランザクションやコネクションの状態によるようですが、わたし自身の理解も浅いためここでは飛ばします（debugして確認するとtrueになっていました）。

`retryable_connection_error?` は以下の通り `ConnectionFailed` の場合trueを返すため今回の例外は条件を満たしていることがわかります。

```ruby
def retryable_connection_error?(exception)
  exception.is_a?(ConnectionNotEstablished) || exception.is_a?(ConnectionFailed)
end
```

結果として⑥の `retry` が実行され `begin` 内の⑦でクエリのリトライ処理が実行されることが確認できました。

## Tips
基本的な動きが確認できたので、もう少し細かな使い方などを紹介します。

### retry回数を変更したい
[デフォルトのretry回数は1回](https://github.com/rails/rails/blob/6b93fff8af32ef5e91f4ec3cfffb081d0553faf0/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb#L223)となっていますが、`connection_retries` で変更することができます。

設定例

```yml:config/database.yml
default: &default
  adapter: mysql2
  connection_retries: 3
```

### 書き込みクエリはリトライしたくない
書き込みでretryを行うのはリスクがあるので避けたいような場合には、 `write_query?` メソッドを使って書き込みはの場合はretryを避けることができます。

```ruby
module DatabaseStatementsMonkeyPatch
  def raw_execute(sql, name, async: false, allow_retry: false, materialize_transactions: true)
    allow_retry = !write_query?(sql)
    super
  end
end
```

### raw_executeを通らない場合のretry
上に書いた例ではraw_executeに対してパッチを当てました。しかし、 `execute` メソッドのように `raw_execute` を通らずにクエリが実行される場合もあります。

```ruby
sql = 'SELECT `users`.* FROM `users` WHERE `users`.`id` =
1 LIMIT 1'
ApplicationRecord.connection.execute(sql)
```

そのような場合においても同様にパッチを当てることでリトライ可能です。

```ruby:lib/active_record/connection_adapters/mysql2/database_statements.rb
require 'active_record/connection_adapters/abstract/database_statements.rb'

module DatabaseStatementsMonkeyPatch
  def execute(sql, name, allow_retry: false)
    allow_retry = true
    super
  end
end

ActiveRecord::ConnectionAdapters::DatabaseStatements.prepend(DatabaseStatementsMonkeyPatch)
```


### trilogyでも使いたい
MySQLのアダプターとしてmysql2以外に[trilogy](https://github.com/trilogy-libraries/trilogy)があります。
trilogyにおいても同様のことが可能です。

```ruby:config/initializers/database_statements_monkey_patch.rb
require 'active_record/connection_adapters/trilogy/database_statements.rb'

unless ActiveRecord.version == "7.1.2"
  raise "Consider removing this patch"
end

module DatabaseStatementsMonkeyPatch
  def raw_execute(sql, name, async: false, allow_retry: false, materialize_transactions: true)
    allow_retry = true
    super
  end
end

ActiveRecord::ConnectionAdapters::Trilogy::DatabaseStatements.prepend(DatabaseStatementsMonkeyPatch)
```

## まとめ
クエリのリトライを可能とする `allow_retry` オプションについて、実装例を元に動作の説明などを行いました。
`allow_retry` が追加されたことによって、リトライ処理をかなり楽に書けるようになったので必要な場合はパッチを当てるのも選択肢となるのではないでしょうか。

但し、ActiveRecordの内部のコードは頻繁に修正が入るため、新しいバージョンで動作しなくなる可能性も高いです。また挙動を全て理解することも難しいです。利用する場合はそのリスクを理解した上で関連する修正をキャッチアップするつもりで使っていきましょう。
