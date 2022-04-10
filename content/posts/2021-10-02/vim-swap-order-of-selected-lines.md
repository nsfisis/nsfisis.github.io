---
title: "Vimで選択した行の順番を入れ替える"
date: 2021-10-02T09:37:25+09:00
draft: false
tags: ["vim"]
aliases: ['/posts/vim-swap-order-of-selected-lines/']
---

この記事は Qiita から移植してきたものです。
元 URL: https://qiita.com/nsfisis/items/4fefb361d9a693803520


-----------------------------------



# バージョン情報

`:version` の一部

> VIM - Vi IMproved 8.2 (2019 Dec 12, compiled Jan 26 2020 11:30:30)
> macOS version
> Included patches: 1-148
> Huge version without GUI.



# よく紹介されている手法

## `tac` / `tail`

`tac` や `tail -r` などの外部コマンドを `!` を使って呼び出し、置き換える。

> :h v_!

`tac` コマンドや `tail` の `-r` オプションは環境によって利用できないことがあり、複数の環境を行き来する場合に採用しづらい




## `:g/^/m0`

こちらは外部コマンドに頼らず、Vim の機能のみを使う。`g` は `:global` コマンドの、`m` は `:move` コマンドの略

`:global` コマンドは `:[range]global/{pattern}/[command]` のように使い、`[range]` で指定された範囲の行のうち、`{pattern}` で指定された検索パターンにマッチする行に対して、順番に `[command]` で指定された Ex コマンドを呼び出す。

> :h :global

`:move` コマンドは `[range]:move {address}` のように使い、`[range]` で指定された範囲の行を `{address}` で指定された位置に移動させる。

> :h :move

`:g/^/m0` のように組み合わせると、「すべての行を1行ずつ 0行目(1行目の上)に動かす」という動きをする。これは確かに行の入れ替えになっている。

なお、`:g/^/m0` は全ての行を入れ替えるが、`:N,Mg/^/mN-1` とすることで N行目から M行目を処理範囲とするよう拡張できる。手でこれを入力するわけにはいかないので、次のようなコマンドを用意する。

```vim
command! -bar -range=%
    \ Reverse
    \ <line1>,<line2>g/^/m<line1>-1
```

これは望みの動作をするが、実際に実行してみると全行がハイライトされてしまう。次節で詳細を述べる。



# `:g/^/m0` の問題点

`:global` コマンドは各行に対してマッチングを行う際、現在の検索パターンを上書きしてしまう。`^` は行の先頭にマッチするため、結果として全ての行がハイライトされてしまう。`'hlsearch'` オプションを無効にしている場合その限りではないが、その場合でも直前の検索パターンが失われてしまうと `n` コマンドなどの際に不便である。

> :h @/



# 解決策

> [2020/9/28追記]
> より簡潔な方法を見つけたので次節に追記した

前述した `:Reverse` コマンドの定義を少し変えて、次のようにする:

```vim
function! s:reverse_lines(from, to) abort
    execute printf("%d,%dg/^/m%d", a:from, a:to, a:from - 1)
endfunction

command! -bar -range=%
    \ Reverse
    \ call <SID>reverse_lines(<line1>, <line2>)
```

実行しているコマンドが変わったわけではないが、関数呼び出しを経由するようにした。これだけで前述の問題が解決する。

この理由は、ユーザー定義関数を実行する際は検索パターンが一度保存され、実行が終了したあと復元されるため。結果として検索パターンが `^` で上書きされることがなくなる。

Vim のヘルプから該当箇所を引用する (強調は筆者による)。

> :h autocmd-searchpat
>
> **Autocommands do not change the current search patterns.**  Vim saves the current
> search patterns before executing autocommands then restores them after the
> autocommands finish.  This means that autocommands do not affect the strings
> highlighted with the 'hlsearch' option.

これは autocommand の実行に関しての記述だが、これと同じことがユーザー定義関数の実行時にも適用される。このことは `:nohlsearch` のヘルプにある。同じく該当箇所を引用する (強調は筆者による)。

> :h :nohlsearch
>
> (略) This command doesn't work in an autocommand, because
> the highlighting state is saved and restored when
> executing autocommands |autocmd-searchpat|.
> **Same thing for when invoking a user function.**

この仕様により、`:g/^/m0` の呼び出しをユーザー定義関数に切り出すことで上述の問題を解決できる。


# 解決策 (改訂版)

> [2020/9/28追記]
> より簡潔な方法を見つけたため追記する

```vim
command! -bar -range=%
    \ Reverse
    \ keeppatterns <line1>,<line2>g/^/m<line1>-1
```

まさにこのための Exコマンド、`:keeppatterns` が存在する。`:keeppatterns {command}` のように使い、読んで字の如く、後ろに続く Exコマンドを「現在の検索パターンを保ったまま」実行する。はるかに分かりやすく意図を表現できる。

> :h :keeppatterns


# コピペ用再掲


```vim
" License: Public Domain

command! -bar -range=%
    \ Reverse
    \ keeppatterns <line1>,<line2>g/^/m<line1>-1
```

