---
title: "【Ruby脳向け】Rustの配列系メソッド対応"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "Ruby", "Array", "Vec", "イテレーター"]
published: false
---

## `select!` → `retain`
```ruby:Ruby
v = [5, 6, 7, 8]
v.select! { |e| e >= 7 }
v  # => [7, 8]
```
```rust:Rust
let mut v = vec![5, 6, 7, 8];
v.retain(|&e| e >= 7);
v  // => [7, 8]
```
retain の意味は「保持」
 
## `reject!` → `retain(|| !)`
```ruby:Ruby
v = [5, 6, 7, 8]
v.reject! { |e| e >= 7 }
v  # => [5, 6]
```
```rust:Rust
let mut v = vec![5, 6, 7, 8];
v.retain(|&e| !(e >= 7));
v  // => [5, 6]
```
retain の逆版は無いっぽいので retain のクロージャで返す値を反転しよう
 
## `select! しつつ要素も更新` → `retain_mut`
```ruby:Ruby
# おすすめしない書き方です
v = ["a", "b", "c"]
v.select! do |e|
  if e == "b" || e == "c"
    if e == "b"
      e.upcase!
    end
    true
  end
end
v  # => ["B", "c"]
```
```rust:Rust (nightly)
let mut v = vec![String::from("a"), String::from("b"), String::from("c")];
v.retain_mut(|e| {
    if e == "b" || e == "c" {
        if e == "b" {
            *e = e.to_uppercase();
        }
        true
    } else {
        false
    }
});
v  // => ["B", "c"]
```
よっぽどのことがなければ集めるのと変更するのは別々に書いた方がいいと思う
 
