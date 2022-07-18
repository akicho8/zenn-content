---
title: "CLI用ライブラリThorの知見メモ"
emoji: "🆑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["thor", "Ruby"]
published: true
---

コマンドラインツールを作るときに便利な [Thor](https://github.com/rails/thor) を使って得た知見をまとめています

# --no-delete としたら削除された #

```ruby
class App < Thor
  desc "func1", "(desc)"
  method_option :delete, type: :boolean
  def func1
    options[:delete] # => false

    params = {
      delete: true,
    }.merge(options)

    if params[:delete]
      p "削除しました"
    end
  end
end

App.start(["func1", "--no-delete"])
# >> "削除しました"
```

  * options は悪しき HashWithIndifferentAccess のインスタンス
  * キーを文字列でもシンボルでもどちらでもアクセスできる
  * しかし、そのメリットよりマージした際にキーが一致せずはまるデメリットの方が多い
  * 結局は `options.transform_keys(&:to_sym)` としてから merge しないといけなかった

# 定義したはずの say メソッドの様子がおかしい #

```ruby
module Kernel
  def say(message)
    puts message
    system "say '#{message}'"
  end
end

class App < Thor
  desc "func1", "(desc)"
  def func1
    say "ok"
  end
end

App.start(["func1"])
# >> ok
```

ok と表示はされるがしゃべらない
原因は say を上書きされていたというオチ
Thor はよかれと次の便利メソッドたちを用意してくれている

> mute mute? padding= indent ask say say_error say_status yes? no? print_in_columns print_table print_wrapped file_collision terminal_width error set_color

これらと名前がかぶると負ける
これに気づかず、めちゃくちゃはまった

# 引数がエラーでもステイタス0を返してしまう問題 #

```ruby
class App < Thor
  def self.exit_on_failure?
    true
  end
end
```

と、書けばいいらしいが、なんでこれがデフォルトじゃないんだ！ と、みんな青筋を立てている

https://github.com/rails/thor/issues/244

しかし最初の Issue が立ってから10年放置っていう

# オプションを繰り返し指定する #

```ruby
class App < Thor
  desc "func1", "(desc)"
  method_option :foo, type: :array, repeatable: true
  method_option :bar, type: :hash, repeatable: true
  def func1
    options[:foo] # => [["1", "2"], ["3", "4"]]
    options[:bar] # => {"a"=>"1", "b"=>"2", "c"=>"3", "d"=>"4"}
  end
end

App.start([
    "func1",
    "--foo", "1", "2",
    "--foo", "3", "4",
    "--bar", "a:1", "b:2",
    "--bar", "c:3", "d:4",
  ])
```

  * `repeatable: true` とすれば引数をマージできる
    * ハッシュの場合は単に merge する
    * 配列の場合はネストしている
      * 平らにするにはあとで flatten が必要
  * デフォルトは false
    * 重複指定でもエラーにはならない
    * 後から指定した方で上書きする

# よく使うコマンドにはショートカットを指定する #

```ruby
class App < Thor
  map "s" => :schedule
  desc "schedule", "スケジュール"
  def schedule
    ...
  end
end
```
`s` で `schedule` が呼べるようになる

# コマンド実行前にクラスオプションを受け取りたい #

```ruby
class App < Thor
  class_option :foo, type: :boolean

  def initialize(...)
    super
    options # => {"foo"=>true}
  end

  desc "func1", "(desc)"
  def func1
  end
end

App.start(["func1", "--foo"])
```

  * 上の例であれば func1 に入った直後で options を参照すればいい
  * しかしコマンドが100個がある場合、100箇所に同じコードを差し込まないといけない
  * このあたり何か用意してくれててもおかしくないがいくら調べてもわからなかった
  * とりあえず initialize の中で参照する方法にして問題はなかった

# options 固有のメソッドを使ってたら後で困った #

コマンド内で使える options (HashWithIndifferentAccess 型) が

```ruby
options[:foo] # => "bar"
```

となっているとき次のように書ける

```ruby
options.foo?        # => true
options.foo         # => "bar"
options.foo?("bar") # => true
```

が、Hash との整合性が取れなくなり、逆にリファクタリングが難しくなったりしたので、活用しない方がいい

# long_desc を使うと改行が削除される #

  * `--help xxx` としたときにコマンド xxx の説明文を最後に出力してくれる便利機能がある
  * その説明文を long_desc メソッドで登録できる
  * なので詳しい使い方を自由に書けると思っていたら改行コードを削除して詰めて画面幅で折り返されてしまった
  * その上、英語である前提なので日本語なんか入れると折り返し位置が変になる
  * とりあえず次のように「何もしない」ようにすると使いやすくなる

```ruby
class Thor::Shell::Basic
  def print_wrapped(message, options = {})
    puts message
  end
end
```

# クラス肥大化による破綻を防ぐ自己流デザインパターン #

次のようなコードを書いていたら数年後に破綻した

```ruby
class App < Thor
  desc "foo_func", "(desc)"
  def foo_func
    foo_func_perform
  end

  desc "bar_func", "(desc)"
  def bar_func
    bar_func_perform
  end

  private

  def foo_func_perform
  end

  def bar_func_perform
  end
end
```

最初は1つのクラスでわかりやすいが、そのまま膨らませていくと、プライベートメソッドが干渉し合ってメンテ不能に陥った
二つの機能 foo と bar は互いに何も関係がないにもかかわらず、同じスコープで共存しているのが失敗だった
これは Rake タスクなどにも言えることで、エントリーポイントには完成されたメソッドを1行書くぐらいでないといけない
可能だからといって同じスコープにプライベートメソッドをぶちまけてはいけない
そこからおかしくなる
Thor継承クラス自体を分けるべきなのかもしれないが、クラスオプションは共有したかったし、サブコマンド扱いでもなかった

改善後

```ruby
module Foo
  module Commands
    extend ActiveSupport::Concern
    included do
      desc "foo_func", "(desc)"
      def foo_func
        Controller.func_perform
      end
    end
  end

  module Controller
    extend self

    def func_perform
    end
  end
end

class App < Thor
  include Foo::Commands
  include Bar::Commands
end
```

  * Module で分離する
  * Bar も同様の構造にする
  * Foo と Bar は何をやっても互いに干渉しないようにする
  * 一つのコードにまとめたが実際はモジュールやクラスはさらにファイルに分割する
    * foo.rb
    * foo/commands.rb
    * foo/controller.rb
  * Thor継承クラスはあくまでエントリーポイントとしての利用に留める

# Thor継承クラスでなければ便利メソッドが使えない問題 #

Thor継承クラスが肥大化しがちな理由の1つとして**そこでしか便利メソッドが使えないから**というのがある

便利メソッドを活用していたコードを別のクラスに移動させるとたちまち `undefine method` の嵐になってしまうのでコードを分離できないというわけ

結局どうすればいいのかよくわかっていないが、ソースを読んだところ、次のようにすればどこからでも呼べることがわかった

```ruby
Thor::Base.shell.new.say("message")
```
