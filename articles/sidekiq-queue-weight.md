---
title: "Sidekiqのキューの重み付け使用時に起こしがちな失敗"
emoji: "🦶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Rails, Sidekiq]
published: false
---

Sidekiqでは、キューの重み付け設定によってJobに優先度をつけることができます。
便利な機能ですが、その仕組みを理解していないと思わぬ失敗を引き起こします。

本記事ではキューの重み付けでやってしまいがちな失敗について、重み付けの仕組みと合わせて紹介します。

## TL;DR
- キューの重み付けはあくまでキューに登録されているjobの実行順番に対しての重み付けである
- jobを処理するスレッドが埋まっているとスレッドが空くまでは優先度が高いjobも実行されない
- 長時間実行されるjobが先にスレッドを埋めていると優先度が高いjobの遅延につながる

## キューの重み付けについて
失敗の話の前にまずはキューの重み付けについて、挙動を確認しながら見ていきます。

### キューの重み付けが必要な場面
Sidekiqで扱うJobにも様々なものがあり、例えば、期限付きのトークンを記載したメール送信など一定時間以内にユーザーに届けたいような優先度が高いJobもあれば、多少処理が遅れても問題ないような優先度の低いJobもあります。
Sidekiqの優先度が低いJobが大量に登録されてキューを埋めてしまい、それらが処理されるまで優先度が高いJobが実行されないといったことがあります。

このようなことを避けて、優先度が高いJobを先に実行したい時にキューの重み付けを

### 重み付け設定方法

sidekiq.yml に例えば以下のように設定します。

```yml:config/initializers/sidekiq.yml
:concurrency: 2
:queues:
  - [critical, 5]
  - [default, 1]
```

### 重み付けの挙動について
上記の設定をした場合、どのような挙動になるかサンプルコードで確認します。

### 検証用のjob

以下のように `critical` と `default` のキューを使うjobを用意します。
どのキューを使うかは `sidekiq_options queue: "critical"` のように指定します。

```ruby
class CriticalJob
  include Sidekiq::Job
  sidekiq_options queue: "critical"

  def perform(number)
    puts "#{number}番目のCriticalJob"
    sleep 1
  end
end

class DefaultJob
  include Sidekiq::Job
  sidekiq_options queue: "default"

  def perform(number)
    puts "#{number}番目のDefaultJob"
    sleep 1
  end
end
```

### 動作検証
以下のコードでjobを動かしてみます。

```ruby
10.times do |i|
  DefaultJob.perform_async(i + 1)
  CriticalJob.perform_async(i + 1)
end
```

#### 1回目の結果

```
1番目のCriticalJob
1番目のDefaultJob
2番目のCriticalJob
3番目のCriticalJob
4番目のCriticalJob
5番目のCriticalJob
6番目のCriticalJob
2番目のDefaultJob
3番目のDefaultJob
7番目のCriticalJob
9番目のCriticalJob
8番目のCriticalJob
10番目のCriticalJob
4番目のDefaultJob
6番目のDefaultJob
5番目のDefaultJob
7番目のDefaultJob
8番目のDefaultJob
10番目のDefaultJob
9番目のDefaultJob
```

#### 2回目の結果

```
1番目のCriticalJob
1番目のDefaultJob
3番目のCriticalJob
2番目のCriticalJob
4番目のCriticalJob
2番目のDefaultJob
3番目のDefaultJob
5番目のCriticalJob
7番目のCriticalJob
6番目のCriticalJob
8番目のCriticalJob
9番目のCriticalJob
4番目のDefaultJob
10番目のCriticalJob
6番目のDefaultJob
5番目のDefaultJob
7番目のDefaultJob
8番目のDefaultJob
10番目のDefaultJob
9番目のDefaultJob
```

#### 結果について
結果は全く同じではありませんが、どちらもCriticalJobが優先して数個実行される中にDefaultJobが実行され、CriticalJobの方が先に全てのjobの実行を終えていることがわかります。

### 重み付けの実装
次に重み付けがどのように実装されているか見ていきます。

#### 重み付け設定の展開
https://github.com/sidekiq/sidekiq/blob/7e73505719138bb7458c9b01fa4899d4b27fa4fe/lib/sidekiq/capsule.rb#L53-L75

で行われています。
strict, weighted, randomの3パターンがあるようですが、今回のように複数の異なる重みを設定した場合はweightedになります。

重みに指定した数だけ@queuesの配列に追加されることになります。

```yml
:queues:
  - [critical, 5]
  - [default, 1]
```

の場合は

```
@queues = ['critical', 'critical', 'critical', 'critical', 'critical', 'default']
```

のように`critical`が5つ、`default`が1つ追加されます。


##### 重み付けの利用

https://github.com/sidekiq/sidekiq/blob/7e73505719138bb7458c9b01fa4899d4b27fa4fe/lib/sidekiq/fetch.rb#L78-L85

queues_cmdで重み付けを利用してjobを取得するコマンドの一部を生成しています。

重み付けを利用する場合は、elseの方の条件に入り、@queuesをランダムに並べ替えた後にユニークにしています。

`critical`と`default`の2つのキューがある場合は、以下のどちらかになりますが、重み付けが高い方が配列の前方（インデックスが小さい要素）に来る確率が高くなります。
- `['critical', 'default']`
- `['default', 'critical']`


##### RedisからJobを取得
https://github.com/sidekiq/sidekiq/blob/7e73505719138bb7458c9b01fa4899d4b27fa4fe/lib/sidekiq/fetch.rb#L38-L49

retrieve_workでredisのコマンドを実行しています。
ここで[brpop](https://redis.io/docs/latest/commands/brpop/)を実行していますが、brpopは与えられたキーを順番にチェックし、空でない最初のリストの末尾から要素をポップします。

今回の例で言うと
`['critical', 'default']` が渡された場合は、 `critical`のキューを先に見て要素（job）が存在する場合はその要素を取得し、なければ `default`を見ます。`['default', 'critical']`が渡された場合は`default`のキューから見ていきます。

そのため、重み付けが大きい方が `配列の前方に来る確率が高い=優先して実行されやすい` という仕組みになっています。

## やってしまいがちな失敗
重み付けで優先度をつけたので優先度が高いjobが遅れて実行されることはないはず！よし！としてしまいがちですが、並列で動く可能性のあるjobの中に**長時間実行されるjob**が含まれる場合は注意が必要です。

例えば、終了に30分かかるLongJobがあったとします。

```ruby
class LongJob
  include Sidekiq::Job
  sidekiq_options queue: "default"

  def perform(number)
    puts "#{number}番目のLongJob"
    sleep 1800
  end
end
```

検証で使ったものと同じ設定で

```yml:config/initializers/sidekiq.yml
:concurrency: 2
:queues:
  - [critical, 5]
  - [default, 1]
```

以下のような順番でjobが実行された場合にCriticalJobが実行されるのはいつになるでしょうか？

```ruby
2.times do |i|
  LongJob.perform_async(i + 1)
end

sleep 10

2.times do |i|
  CriticalJob.perform_async(i + 1)
end
```

正解は約30分後（少なくとも一つのLongJobの処理が終わった後）です。

### LongJobが終わるまでCriticalJobが実行されない理由
挙動としては、まずLongJobがdefaultキューに2つ積まれます。

![](/images/sidekiq-queue-weight/state1.png =600x)

LongJobがキューに入ったタイミングでまだCriticalJobがキューにないため、重み付けに関係なくLongJobが動き始めます。
実装の「RedisからJobを取得」で見たようにcriticalキューにjobが積まれていない場合は、defaultキューのjobがpopされて実行されます。

![](/images/sidekiq-queue-weight/state2.png =600x)

この時、Sidekiqのconcurrencyは2に設定されているため、jobを実行するスレッドが2つともLongJobが使うことになります。
この状態でCriticalJobがキューに積まれますが、スレッドが空いていないためLongJobの終了を待ち続けることになります。

![](/images/sidekiq-queue-weight/state3.png =600x)

このように**jobを実行するスレッドが埋まっていると優先度が高いjobも実行されない**ことを理解しておかないと、思わぬ遅延発生につながるため注意が必要です。
