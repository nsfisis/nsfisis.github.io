---
title: "[Vim] autocmd events の BufWrite/BufWritePre の違い"
date: 2021-10-02T09:37:12+09:00
draft: false
tags: ["vim"]
aliases: ['/posts/vim-difference-between-autocmd-bufwrite-and-bufwritepre/']
summary: |
  Vim の autocmd events における BufWrite/BufWritePre がどう違うのかを調べた結果、違いはないことがわかった。
---

この記事は Qiita から移植してきたものです。
元 URL: https://qiita.com/nsfisis/items/79ab4db8564032de0b25


-----------------------------------



# TL; DR

違いはない。ただのエイリアス。


# 調査記録

Vim の autocmd events には似通った名前のものがいくつかある。大抵は `:help` に説明があるが、この記事のタイトルにある2つを含めた以下のイベントには、その違いについて説明がない。

* `BufRead`/`BufReadPost`
* `BufWrite`/`BufWritePre`
* `BufAdd`/`BufCreate`

このうち、`BufAdd`/`BufCreate` に関しては、`:help BufCreate` に

> The BufCreate event is for historic reasons.

とあり、おそらくは `BufAdd` のエイリアスであろうということがわかる。他の2組も同様ではないかと予想されるが、確認のため vim と neovim のソースコードを調査した。

> ソースコードへのリンク
> [vim (調査時点での master branch)](https://github.com/vim/vim/tree/8e6be34338f13a6a625f19bcef82019c9adc65f2)
> [neovim (上に同じ)](https://github.com/neovim/neovim/tree/71d4f5851f068eeb432af34850dddda8cc1c71e3)

## vim のソースコード

以下は、autocmd events の名前と内部で使われている整数値とのマッピングを定義している箇所である。見ての通り、上でエイリアスではないかと述べた3組には、それぞれ同じ内部値が使われている。

https://github.com/vim/vim/blob/8e6be34338f13a6a625f19bcef82019c9adc65f2/src/autocmd.c#L85-L86

```c
    {"BufAdd",		EVENT_BUFADD},
    {"BufCreate",	EVENT_BUFADD},
```

https://github.com/vim/vim/blob/8e6be34338f13a6a625f19bcef82019c9adc65f2/src/autocmd.c#L95-L97

```c
    {"BufRead",		EVENT_BUFREADPOST},
    {"BufReadCmd",	EVENT_BUFREADCMD},
    {"BufReadPost",	EVENT_BUFREADPOST},
```

https://github.com/vim/vim/blob/8e6be34338f13a6a625f19bcef82019c9adc65f2/src/autocmd.c#L103-L105

```c
    {"BufWrite",	EVENT_BUFWRITEPRE},
    {"BufWritePost",	EVENT_BUFWRITEPOST},
    {"BufWritePre",	EVENT_BUFWRITEPRE},
```

## neovim のソースコード

neovim の場合でも同様のマッピングが定義されているが、こちらの場合は Lua で書かれている。以下にある通り、はっきり `aliases` と書かれている。

https://github.com/neovim/neovim/blob/71d4f5851f068eeb432af34850dddda8cc1c71e3/src/nvim/auevents.lua#L119-L124

```lua
  aliases = {
    BufCreate = 'BufAdd',
    BufRead = 'BufReadPost',
    BufWrite = 'BufWritePre',
    FileEncoding = 'EncodingChanged',
  },
```

ところで、上では取り上げなかった `FileEncoding` だが、これは `:help FileEncoding` にしっかりと書いてある。

```
                                                           *FileEncoding*
FileEncoding                    Obsolete.  It still works and is equivalent
                                to |EncodingChanged|.
```

## まとめ

記事タイトルについて言えば、どちらも変わらないので好きな方を使えばよい。あえて言えば、次のようになるだろう。

* `BufAdd`/`BufCreate`
  * → `BufCreate` は歴史的な理由により ("for historic reasons") 存在しているため、新しい方 (`BufAdd`) を使う
* `BufRead`/`BufReadPost`
  * → `BufReadPre` との対称性のため、あるいは `BufWritePost` との対称性のため `BufReadPost` を使う
* `BufWrite`/`BufWritePre`
  * → `BufWritePost` との対称性のため、あるいは `BufReadPre` との対称性のため `BufWritePre` を使う
* `FileEncoding`/`EncodingChanged`
  * → `FileEncoding` は "Obsolete" と明言されているので、`EncodingChanged` を使う

ところでこの調査で知ったのだが、`BufRead` と `BufWrite` は上にある通り発火するタイミングが「後」と「前」で対称性がない。可能なら `Pre`/`Post` 付きのものを使った方が分かりやすいだろう。

