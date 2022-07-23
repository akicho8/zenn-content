---
title: "Rubyのコードで理解するRustのResultの振る舞い"
emoji: "🍪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "Ruby", "Result"]
published: true
---
Rust をコードを見るとあちこちで unwrap unwrap_or expect が呼ばれていてわけがわからなかったけど Ruby で抽象化したらだいたいわかった (抽象化したコードがさらに動くのがありがたい)。

なお Option 型も Ok Err が Some None(値なし) なっただけでほぼ同じ。

## 抽象化したコード

```ruby:Ruby
class Result
  def initialize(value)
    @value = value
  end

  def inspect
    "#{self.class.name}(#{@value.inspect})"
  end
end

class Ok < Result
  def unwrap
    @value
  end

  def unwrap_or(iferr)
    unwrap
  end

  def expect(message)
    unwrap
  end

  def map(&block)
    self.class.new(block[@value])
  end
end

class Err < Result
  def unwrap
    raise "panicked: #{inspect}"
  end

  def unwrap_or(iferr)
    iferr
  end

  def expect(message)
    raise message
  end

  def map(&block)
    self
  end
end

def Ok(value)
  Ok.new(value)
end

def Err(value)
  Err.new(value)
end
```

## 基本動作

1. Err に対して値を取り出そうとするとエラーになる
1. ポリモルフィックな操作ができたりする

```ruby:Ruby
x = Ok(200)                 # => Ok(200)
x.unwrap rescue $!          # => 200
x.unwrap_or(300) rescue $!  # => 200
x.map { |e| e * 2 }         # => Ok(400)
x.expect("失敗") rescue $!  # => 200

x = Err(500)                # => Err(500)
x.unwrap rescue $!          # => #<RuntimeError: panicked: Err(500)>
x.unwrap_or(300) rescue $!  # => 300
x.map { |e| e * 2 }         # => Err(500)
x.expect("失敗") rescue $!  # => #<RuntimeError: 失敗>
```

Ruby 的には仰々しく無理矢理な感はあるもののデザインパターンの一つとして有用かもしれない。

## Rustは配列がResult型に依存したメソッドを持っている？

ところで Rust の配列は要素が Result 型であることを想定したメソッドを持っているように見える。

Itertools は外部のライブラリだけど AtCoder でも使えるぐらい標準よりのライブラリで、それが提供している配列のメソッド map_ok は要素が Result 型だと想定している。

```rust:Rust
use itertools::Itertools;

fn main() {
    let v = vec![Ok(5), Err("x"), Ok(6), Err("x")];
    let v = v.into_iter().map_ok(|e| e * 2).collect_vec();
    println!("{:?}", v); // >> [Ok(10), Err("x"), Ok(12), Err("x")]
}
```

Ruby でも書いてみるがめちゃくちゃ抵抗がある。
map なんてのはすべてのオブジェクトが持っているメソッドではないのだから。

```ruby:Ruby
module Enumerable
  def map_ok(&block)
    collect { |e| e.map(&block) }
  end
end

v = [Ok(5), Err("x"), Ok(6), Err("x")]
v.map_ok { |e| e * 2 } # => [Ok(10), Err("x"), Ok(12), Err("x")]
```

ここで試しに Rust のコードの配列要素を整数にするとなんと map_ok が使えなくなった。というかまずコンパイルが通らなくなった。
つまり map_ok は Result 型要素の配列にしか生えていない。Rust の配列は要素の型ごとにもっているメソッドが異なる。正確には map_ok メソッドを持ったトレイトの適用範囲が異なるということなんだろう。たぶん。

## Rust の `if let Ok(x) = xxx` を Ruby で書けるか？

ラップされたままポリモルフィックであれこれできるとはいえ結局パニックを避けつつ値を取り出して分岐するコードを書くことになる。
このとき Rust にはスマートな構文がある。

```rust:Rust
let v: Result<_, &str> = Ok(200);
if let Ok(x) = v {
    println!("{:?}", x);    // >> 200
}
```

これを Ruby で普通に書くと `v.kind_of?(Ok)` なら `x = v.unwrap` としなくてはいけなくて冗長だ。
そこで一度も使ったことがないパターンマッチングを使ってみる。

```ruby:Ruby
class Result
  def to_h
    { self.class.name.to_sym => @value }
  end
end

v = Ok(200)
if v.to_h in { Ok: x }
  x  # => 200
end
```

どうだろう？
Ok をシンボルにしないといけないので、それに合わせて to_h が必要になってきていまいちか。
