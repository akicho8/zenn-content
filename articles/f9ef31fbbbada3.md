---
title: "[Rust] Pathに含まれる ~/ を展開する方法"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "Path"]
published: true
---

## 概要

標準クレートの Path を一通り使ってみて、かなり気が利く作りになっていることはわかった。が、 Ruby の expand_path に相当する `~/` の展開をやってくれないのが気になったので調べてみる。

## 標準で対応していた？

ホームディレクトリ自体は `home_dir` で取得できた。

```rust
std::env::home_dir() // => Some("/Users/alice")
```

なのに展開してくれないのは `home_dir` が非推奨になっているからのようだ。
上のコードをコンパイルすると下の警告が出てしまう。

```
warning: use of deprecated function `std::env::home_dir`: This function's behavior is unexpected and probably not what you want. Consider using a crate from crates.io instead.
```

## 外部クレートを探す

そこで外部クレートと探してみると `home-dir` というそれっぽいのを見つけたので使ってみる。

```rust
use std::path::Path;
use home_dir::*;

Path::new("~/src").expand_home() // => Ok("/Users/alice/src")
```

これだー！！

## どういう実装？

覗いてみるとこうなっていた。

まず環境変数 HOME を見る。

```rust
use std::env;
env::var("HOME") // => Ok("/Users/alice")
```

取れなければ uid から何かをあれして取得する。

```rust
use nix::unistd::{Uid, User};
let uid = Uid::current();
User::from_uid(uid).unwrap().unwrap().dir // => "/Users/alice"
```

外部クレートの nix はUNIXに依存した情報を取得できるらしい。

## ちょっとはまったこと

`home_dir` が `nix` に依存しているからといって `home_dir` を使うプログラムから `use nix` とはできないようだ。
Ruby (bundler) や JavaScript (npm) では、依存しているライブラリがさらに依存しているライブラリをメインのプログラムのどこからでもついでに使うことができたので戸惑ってしまった。
Rust の仕様の方がスコープが明確で良さそうに感じる。
