---
title: "[Ruby] 自身を実行している処理系の種類を判定する"
date: 2021-10-02T09:37:50+09:00
draft: false
tags: ["ruby"]
aliases: ['/posts/ruby-detect-running-implementation/']
summary: |
  Ruby には複数の実装があるが、自身を実行している処理系の種類をスクリプト上からどのように判定すればよいだろうか。
changelog:
  2021-10-02: Qiita から移植
---

この記事は Qiita から移植してきたものです。
元 URL: https://qiita.com/nsfisis/items/74d7ffeeebc51b20d791


-----------------------------------



Ruby という言語には複数の実装があるが、それらをスクリプト上からどのようにして programmatically に見分ければよいだろうか。

`Object` クラスに定義されている `RUBY_ENGINE` という定数がこの用途に使える。

参考: [Object::RUBY_ENGINE](https://docs.ruby-lang.org/ja/latest/method/Object/c/RUBY_ENGINE.html)

上記ページの例から引用する:

```shell-session
$ ruby-1.9.1 -ve 'p RUBY_ENGINE'
ruby 1.9.1p0 (2009-03-04 revision 22762) [x86_64-linux]
"ruby"
$ jruby -ve 'p RUBY_ENGINE'
jruby 1.2.0 (ruby 1.8.6 patchlevel 287) (2009-03-16 rev 9419) [i386-java]
"jruby"
```

それぞれの処理系がどのような値を返すかだが、stack overflow に良い質問と回答があった。

[What values for RUBY_ENGINE correspond to which Ruby implementations?](https://stackoverflow.com/a/9894232) より引用:

> | RUBY_ENGINE | Implementation    |
> |:-----------:|:------------------|
> | \<undefined\> | MRI < 1.9         |
> | 'ruby'      | MRI >= 1.9 or REE |
> | 'jruby'     | JRuby             |
> | 'macruby'   | MacRuby           |
> | 'rbx'       | Rubinius          |
> | 'maglev'    | MagLev            |
> | 'ironruby'  | IronRuby          |
> | 'cardinal'  | Cardinal          |


なお、この質問・回答は 2014年になされたものであり、値は変わっている可能性がある。MRI (aka CRuby) については執筆時現在 (2020/12/8) も `'ruby'` が返ってくることを確認済み。

この表にない主要な処理系として、[mruby](https://mruby.org) は `'mruby'` を返す。

[mruby 該当部分のソース](https://github.com/mruby/mruby/blob/ed29d74bfd95362eaeb946fcf7e865d80346b62b/include/mruby/version.h#L32-L35) より引用:

```c
/*
 * Ruby engine.
 */
#define MRUBY_RUBY_ENGINE  "mruby"
```

