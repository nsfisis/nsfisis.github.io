---
title: "[Python] クロージャとUnboundLocalError: local variable 'x' referenced before assignment"
date: 2021-10-02T09:32:37+09:00
draft: false
---

この記事は Qiita から移植してきたものです。
元 URL: https://qiita.com/nsfisis/items/5d733703afcb35bbf399


-----------------------------------


本記事は Python 3.7.6 の動作結果を元にして書かれている。


Python でクロージャを作ろうと、次のようなコードを書いた。

```python
def f():
    x = 0
    def g():
        x += 1
    g()

f()
```

関数 `g` から 関数 `f` のスコープ内で定義された変数 `x` を参照し、それに 1 を足そうとしている。
これを実行すると `x += 1` の箇所でエラーが発生する。

> UnboundLocalError: local variable 'x' referenced before assignment

local変数 `x` が代入前に参照された、とある。これは、`f` の `x` を参照するのではなく、新しく別の変数を `g` 内に作ってしまっているため。
前述のコードを宣言と代入を便宜上分けて書き直すと次のようになる。`var` を変数宣言のための構文として擬似的に利用している。

```python
# 注: var は正しい Python の文法ではない。上記参照のこと
def f():
    var x           #  f の local変数 'x' を宣言
    x = 0           #  x に 0 を代入
    def g():        #  f の内部関数 g を定義
        var x       #  g の local変数 'x' を宣言
                    #  たまたま f にも同じ名前の変数があるが、それとは別の変数
        x += 1      #  x に 1 を加算 (x = x + 1 の糖衣構文)
                    #  加算する前の値を参照しようとするが、まだ代入されていないためエラー
    g()
```

当初の意図を表現するには、次のように書けばよい。

```python
def f():
    x = 0
    def g():
        nonlocal x   ## (*)
        x += 1
    g()
```

`(*)` のように、`nonlocal` を追加する。これにより一つ外側のスコープ (`g` の一つ外側 = `f`) で定義されている `x` を探しに行くようになる。


