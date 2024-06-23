---
title: "【Rails】Bootbootを使ったDual Boot Upgradeの実践"
emoji: "🐥"
type: "tech"
topics: [Rails]
published: false
---

Railsアプリケーションのアップグレード手法の一つにDual Boot Upgradeがあります。
色んな実装方法がありますが、本記事ではShopifyが提供しているBootbootというbundlerプラグインを使った方法を紹介していきます。

https://github.com/Shopify/bootboot


## Dual Boot Upgradeとは
現在のバージョンと異なるバージョン(次のバージョンやmainブランチなど)を同じコードで動かせるようにし、アップグレードしていくテクニックを言います。
例えば、Rails7.1で動いているアプリケーションコードで、7.2も動かせるようにして事前に必要な修正などを把握し対応を進めることができます。

詳しく知りたい方には[RAILS CONFERENCE 2022 AND THE DUAL BOOT UPGRADE STRATEGY](https://blog.grio.com/2022/06/rails-conference-2022-and-the-dual-boot-upgrade-strategy.html)の記事がおすすめです。

## Bootbootを試す
早速Bootbootを使って試していきます。

### 準備
Gemfileにpluginを追加し


```ruby:Gemfile
plugin 'bootboot', '~> 0.2.2'
```

以下を実行します。

```sh
$ bundle install && bundle bootboot
```

`Gemfile_next.lock` が追加されます。
また`Gemfile`の下部に以下のようなコードが追加されます

```ruby:Gemfile
Plugin.send(:load_plugin, 'bootboot') if Plugin.installed?('bootboot')

if ENV['DEPENDENCIES_NEXT']
  enable_dual_booting if Plugin.installed?('bootboot')

  # Add any gem you want here, they will be loaded only when running
  # bundler command prefixed with `DEPENDENCIES_NEXT=1`.
end
```

### Railsの次のバージョンを入れる

Gemfileに次のバージョンを指定します。（既に最新のバージョンを使っている場合はmainブランチを指定）

```ruby:Gemfile
if ENV['DEPENDENCIES_NEXT']
　# 記事執筆時点では7.2.0のリリース直前でbeta2が出てる直前だったためbeta2を指定していますが深い意味はありません
  gem "rails", "7.2.0.beta2"
else
  gem "rails", "~> 7.1.3"
end
```

環境変数によってどちらを使うか分岐ができるようになったので、gemをinstallします。

```sh
$ DEPENDENCIES_NEXT=1 bundle install
```


新バージョンのRailsがインストールされ `Gemfile_next.lock` に依存関係が反映されます。
準備ができたので新しいバージョンで動かしていきます。

### テストの実行
以下のように`DEPENDENCIES_NEXT=1`をつけて実行することで次のバージョンでテストを実行できます。

```sh
# RSpecを入れている場合の例
$ DEPENDENCIES_NEXT=1 bundle exec rspec spec
```

### Railsの起動
同様に `DEPENDENCIES_NEXT=1` を渡して起動すれば新バージョンのRailsでアプリケーションを動かすことができます。

```
$ DEPENDENCIES_NEXT=1 bundle exec rails s
```

### Gemのupdate
gemのバージョンを上げる場合は`bundle update`や`bundle install`を実行するだけでどちらの依存関係も更新できます。
例えば、例えばconfig gemをアップデートする場合は以下を実行すると、`Gemfile.lock`と`Gemfile_next.lock` 両方の依存関係が更新されます

```sh
$ bundle update config
```

なぜ両方が更新される気になる方は、「番外編1: Bootbootの仕組み」でふれていますのでそちらを参照ください。

### 新旧バージョンでコードを分岐したい場合
新しいRailsでは動かなくるようなコードを一時的に分岐して動作させたくなることがあります。
そのような場合は、`ENV['DEPENDENCIES_NEXT']`が使えます。

例えば新しいRailsでは`bullet`動かなくいので対応されるまで一時的に外したいということがった場合、以下のように書くことができます。

```ruby:Gemfile
unless ENV['DEPENDENCIES_NEXT']
  gem 'bullet'
end
```

## アップグレードの流れ
複数バージョンを同じコードで動かせるようになったので、次はアップグレード時の流れについてです。
例えば以下のような流れでやっていきます。

### 1. CI上での定期実行
新バージョンでのテストをCI上で定期的に実行させます。
現在のバージョンと同様の頻度で動かしてもいいのですが、CIのコストを考えると週に1回などの定期実行がおすすめです。

### 2. 新バージョン用の修正を反映
テストの結果から新バージョンで対応が必要となる内容を検知し以下のような対応を進めて現バージョンのコードに反映していきます。
- DEPRECATION WARNINGの解消
- failするテストを修正し新、旧両方のバージョンで通るように修正する
- 新バージョンで動作しないgemの対応
  - 対応されるのを待つか、もしくは自分でforkして修正するか、gemを外すことができないかなどを検討して対応する進める

gemの依存関係などの問題で全ての対応が難しい場合もありますが、可能な限り修正を進めておきます。
必要に応じて `ENV['DEPENDENCIES_NEXT']` を使った分岐を入れていきますが、分岐が多くなりすぎると後から修正も大変なので使わないで済むならそうします。

### 3. 新バージョンへのアップグレードをリリース
2によってRailsの新バージョンリリース後の準備がほとんど完了している状態を作れます。
あとは新バージョンリリースのタイミングでバージョンを上げてリリースします。

### 何が嬉しいか
バージョンをアップに必要な対応を早期に検知し、少しづつ現在のバージョンのコードに反映できます。
これによって長期間のアップグレード用のブランチ運用が不要となり、最終的なバージョンアップ時の修正を少なくすることで安全に素早くアップグレードを行うことが可能になります！

## まとめ
BootbootはGemfileにpluginを追加するだけで簡単に使い始めることができます。
ぜひみなさんも試してみて安全で素早いRailsアップグレードを実現しましょう！

## 番外編1: Bootbootの仕組み

Bootbootの機能はシンプルですがどんな仕組みで動いているかコードの一部を追ってみます。

### enable_dual_booting
上述の通り `bundle bootboot` を実行すると、以下のようなコードがGemfileに追加されます。
enable_dual_bootingが何をしているか追ってみます。

```ruby:Gemfile
if ENV['DEPENDENCIES_NEXT']
  enable_dual_booting if Plugin.installed?('bootboot')

  # Add any gem you want here, they will be loaded only when running
  # bundler command prefixed with `DEPENDENCIES_NEXT=1`.
end
```

https://github.com/Shopify/bootboot/blob/c0eef7152ef2b53f66e8b9799dcbfac3960fbd0c/lib/bootboot/bundler_patch.rb#L53-L63

が enable_dual_bootingのコードになります。
55行目で `Bundler::Definition` に `DefinitionPatch` をprependしており
https://github.com/Shopify/bootboot/blob/c0eef7152ef2b53f66e8b9799dcbfac3960fbd0c/lib/bootboot/bundler_patch.rb#L5-L15

bundler側のinitializeをオーバーライドして使用するlockファイルのパスを切り替えていました。

57行目では、bundlerのキャッシュのパスを変更しています。
こういった細かいところを自前で実装するのは面倒なのでありがたいですね。

### GemfileNextAutoSync
bootbootの設定をした状態で、`bundle update`を実行すると`Gemfile.lock`と`Gemfile_next.lock`の両方が更新されます。
個人的にBootbootの好きな挙動ですが、この機能は`GemfileNextAutoSync::GemfileNextAutoSync`クラスで実現していました。

メインの処理は`opt_in`というメソッドで行っています。
https://github.com/Shopify/bootboot/blob/c0eef7152ef2b53f66e8b9799dcbfac3960fbd0c/lib/bootboot/gemfile_next_auto_sync.rb#L25-L40

30行目を見ると `after-install-all` eventsをhookしています。
bundler側のコードを見ると
https://github.com/rubygems/rubygems/blob/c1a7fbdd4abf0379cfe507a5cbf1c790cae4c4bd/bundler/lib/bundler/plugin/events.rb#L54-L58
https://github.com/rubygems/rubygems/blob/c1a7fbdd4abf0379cfe507a5cbf1c790cae4c4bd/bundler/lib/bundler/installer.rb#L20-L26

installコマンド内でメインの処理(`installer.run(options)`)の後にhookを入れることができるようです。

gemfile_next_auto_sync.rbのコードに戻るといくつかの条件を満たした後に38行目で更新を行っています。

## 番外編2: Bootboot以外の方法
Bootboot以外にもいくつか実装の候補を紹介します。

### Next Rails
https://github.com/fastruby/next_rails


Bootbootとほぼ同じことができ機能が豊富ですが、一方でBootbootの方がシンプルに使えるようです。
> Both of these upgrade Gems accomplish the same thing; it just depends on your preferences for upgrading. Next-rails is a bit more future rich, but BootBoot is easier to implement.
> 引用元: https://blog.grio.com/2022/06/rails-conference-2022-and-the-dual-boot-upgrade-strategy.html


### 自前で仕組みを準備
[dual-boot-with-main](https://www.fastruby.io/blog/dual-boot-with-main.html)にあるように、自前で実装していくことも選択肢の一つです。

```ruby:Gemfile
def next?
  File.basename(__FILE__) == "Gemfile.next"
end

if next?
  gem "rails", "~> 7.1.0"
else
  gem "rails", "~> 7.0.0"
end
```

のようにGemfileを修正し、 `BUNDLE_GEMFILE=Gemfile.next bundle update` ように実行時にどちらを使用するか指定します。
依存関係分けてインストールするだけならシンプルにできるので、まずはここから始めてもいいかもしれません。
一方でBootbootにあったGemfileNextAutoSyncのように運用を楽にするような機能まで実装していくとなるとgemを使った方が楽だと思います。
