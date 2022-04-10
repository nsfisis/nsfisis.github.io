---
title: "[Ruby] then キーワードと case in"
date: 2021-10-02T09:38:50+09:00
draft: false
tags: ["ruby", "ruby3"]
aliases: ['/posts/ruby-then-keyword-and-case-in/']
---

この記事は Qiita から移植してきたものです。
元 URL: https://qiita.com/nsfisis/items/787a8cf888a304497223


-----------------------------------


# TL; DR

`case` - `in` によるパターンマッチング構文でも、`case` - `when` と同じように `then` が使える (場合によっては使う必要がある)。

# `then` とは

使われることは稀だが、Ruby では `then` がキーワードになっている。次のように使う:

```ruby
if cond then
  puts "Y"
else
  puts "N"
end
```

このキーワードが現れうる場所はいくつかあり、`if`、`unless`、`rescue`、`case` 構文がそれに当たる。
上記のように、何か条件を書いた後 `then` を置き、式がそこで終了していることを示すマーカーとして機能する。

```ruby
# Example:

if x then
  a
end

unless x then
  a
end

begin
  a
rescue then
  b
end

case x
when p then
  a
end
```

# なぜ普段は書かなくてもよいのか

普通 Ruby のコードで `then` を書くことはない。なぜか。次のコードを実行してみるとわかる。

```ruby
if true puts 'Hello, World!' end
```

次のような構文エラーが出力される。

```
20:1: syntax error, unexpected local variable or method, expecting `then' or ';' or '\n'
if true puts 'Hello, World!' end
        ^~~~
20:1: syntax error, unexpected `end', expecting end-of-input
...f true puts 'Hello, World!' end
```

二つ目のメッセージは無視して一つ目を読むと、`then` か `;` か改行が来るはずのところ変数だかメソッドだかが現れたことによりエラーとなっているようだ。

ポイントは改行が `then` (や `;`) の代わりとなることである。`true` の後に改行を入れてみる。

```ruby
if true
puts 'Hello, World!' end
```

無事 Hello, World! と出力されるようになった。

# なぜ `then` や `;` や改行が必要か

なぜ `then` や `;` や改行 (以下 「`then` 等」) が必要なのだろうか。次の例を見てほしい:

```ruby
if a b end
```

`then` も `;` も改行もないのでエラーになるが、これは条件式がどこまで続いているのかわからないためだ。
この例は二通りに解釈できる。

```ruby
# a という変数かメソッドの評価結果が truthy なら b という変数かメソッドを評価
if a then
  b
end
```

```ruby
# a というメソッドに b という変数かメソッドの評価結果を渡して呼び出し、
# その結果が truthy なら何もしない
if a(b) then
end
```

`then` 等はこの曖昧性を排除するためにあり、条件式は `if` から `then` 等までの間にある、ということを明確にする。
C系の `if` 後に来る `(`/`)` や、Python の `:`、Rust/Go/Swift などの `{` も同じ役割を持つ。

Ruby の場合、プログラマーが書きやすいよう改行でもって `then` が代用できるので、ほとんどの場合 `then` は必要ない。

# `case` - `in` における `then`

ようやく本題にたどり着いた。来る Ruby 3.0 では `case` と `in` キーワードを使ったパターンマッチングの構文が入る予定である。この構文でもパターン部との区切りとして `then` 等が必要になる。
(現在の) Ruby には formal な形式での文法仕様は存在しないので、yacc の定義ファイルを参照した (yacc の説明は省略)。

https://github.com/ruby/ruby/blob/221ca0f8281d39f0dfdfe13b2448875384bbf735/parse.y#L3961-L3986

```yacc
p_case_body	: keyword_in
		    {
			SET_LEX_STATE(EXPR_BEG|EXPR_LABEL);
			p->command_start = FALSE;
			$<ctxt>1 = p->ctxt;
			p->ctxt.in_kwarg = 1;
			$<tbl>$ = push_pvtbl(p);
		    }
		    {
			$<tbl>$ = push_pktbl(p);
		    }
		  p_top_expr then
		    {
			pop_pktbl(p, $<tbl>3);
			pop_pvtbl(p, $<tbl>2);
			p->ctxt.in_kwarg = $<ctxt>1.in_kwarg;
		    }
		  compstmt
		  p_cases
		    {
		    /*%%%*/
			$$ = NEW_IN($4, $7, $8, &@$);
		    /*% %*/
		    /*% ripper: in!($4, $7, escape_Qundef($8)) %*/
		    }
		;
```

簡略版:

```yacc
p_case_body	: keyword_in p_top_expr then compstmt p_cases
		;
```

ここで、`keyword_in` は文字通り `in`、`p_top_expr` はいわゆるパターン、`then` は `then` キーワードのことではなく、この記事で `then` 等と呼んでいるもの、つまり `then` キーワード、`;`、改行のいずれかである。

これにより、`case` - `when` による従来の構文と同じように、`then` 等をパターンの後ろに挿入すればよいことがわかった。つまり次の3通りのいずれかになる:

```ruby
case x
in 1 then a
in 2 then b
in 3 then c
end

case x
in 1
  a
in 2
  b
in 3
  c
end

case x
in 1; a
in 2; b
in 3; c
end
```

ところで、`p_top_expr` には `if` による guard clause が書けるので、その場合は `if` - `then` と似たような見た目になる。

```ruby
case x
in 0 then a
in n if n < 0 then b
in n then c
end
```

# まとめ

* `if` や `case` の条件の後ろには `then`、`;`、改行のいずれかが必要
  * 通常は改行しておけばよい
* 3.0 で入る予定の `case` - `in` でも `then` 等が必要になる
* Ruby の構文を正確に知るには (現状) `parse.y` を直接読めばよい

