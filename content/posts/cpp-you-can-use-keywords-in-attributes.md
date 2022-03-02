---
title: "[C++] 属性構文の属性名にはキーワードが使える [[void]] [[for]]"
date: 2021-10-02T09:38:30+09:00
draft: false
tags: ["cpp", "cpp17"]
---


この記事は Qiita から移植してきたものです。
元 URL: https://qiita.com/nsfisis/items/94090937bcf860cfa93b


-----------------------------------


タイトル落ち。まずはこのコードを見て欲しい。

```cpp
#include <iostream>

[[alignas]] [[alignof]] [[and]] [[and_eq]] [[asm]] [[auto]] [[bitand]]
[[bitor]] [[bool]] [[break]] [[case]] [[catch]] [[char]] [[char16_t]]
[[char32_t]] [[class]] [[compl]] [[const]] [[const_cast]] [[constexpr]]
[[continue]] [[decltype]] [[default]] [[delete]] [[do]] [[double]]
[[dynamic_cast]] [[else]] [[enum]] [[explicit]] [[export]] [[extern]] [[false]]
[[final]] [[float]] [[for]] [[friend]] [[goto]] [[if]] [[inline]] [[int]]
[[long]] [[mutable]] [[namespace]] [[new]] [[noexcept]] [[not]] [[not_eq]]
[[nullptr]] [[operator]] [[or]] [[or_eq]] [[override]] [[private]]
[[protected]] [[public]] [[register]] [[reinterpret_cast]] [[return]] [[short]]
[[signed]] [[sizeof]] [[static]] [[static_assert]] [[static_cast]] [[struct]]
[[switch]] [[template]] [[this]] [[thread_local]] [[throw]] [[true]] [[try]]
[[typedef]] [[typeid]] [[typename]] [[union]] [[unsigned]]
[[virtual]] [[void]] [[volatile]] [[wchar_t]] [[while]] [[xor]] [[xor_eq]]
// [[using]]
int main() {
    std::cout << "Hello, World!" << std::endl;
}
```

> コンパイラのバージョン
> $ clang++ --version
> Apple clang version 11.0.0 (clang-1100.0.33.8)
> Target: x86_64-apple-darwin19.6.0
> Thread model: posix
> InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
>
> コンパイルコマンド (C++17指定)
> $ clang++ --std=c++17 hoge.cpp

この記事から得られるものはこれ以上ないので以下は蛇足になる。

別件で cppreference.com の [identifier のページ](https://en.cppreference.com/w/cpp/language/identifiers) を読んでいた時、次の文が目に止まった。

> * the identifiers that are keywords cannot be used for other purposes;
>   * The only place they can be used as non-keywords is in an attribute-token. (e.g. [[private]] is a valid attribute) (since C++11)

キーワードでも属性として指定する場合は非キーワードとして使えるらしい。
実際にやってみる。

同サイトの [keywords のページ] (https://en.cppreference.com/w/cpp/keyword) から一覧を拝借し、上のコードが出来上がった (C++17 においてキーワードでないものなど、一部省いている)。
大量の警告 (unknown attribute '〇〇' ignored) がコンパイラから出力されるが、コンパイルできる。

上のコードでは `[[using]]` をコメントアウトしているが、これは `using` キーワードのみ属性構文の中で意味を持つからであり、このコメントアウトを外すとコンパイルに失敗する。

```cpp
// using の例
[[using foo: attr1, attr2]] int x; // [[foo::attr1, foo::attr2]] の糖衣構文
```

C++17 の仕様も見てみる (正確には標準化前のドラフト)。

引用元: https://timsong-cpp.github.io/cppwp/n4659/dcl.attr#grammar-4

> If a keyword or an alternative token that satisfies the syntactic requirements of an identifier is contained in an attribute-token, it is considered an identifier.

「`identifier` の構文上の要件を満たすキーワードまたは代替トークンが `attribute-token` に含まれている場合、`identifier` とみなされる」とある。どうやら間違いないようだ。

ところで、代替トークン (alternative token) とは `and` (`&`) や `bitor` (`|`) などのことだが、`identifier` の構文上の要件を満たさないような代替トークンなどあるのか？
疑問に思って調べたところ、代替トークンという語にはダイグラフも含まれるらしい (参考: [同ドラフト](https://timsong-cpp.github.io/cppwp/n4659/lex.digraph))

- `<%` → `{`
- `%>` → `}`
- `<:` → `[`
- `:>` → `]`
- `%:` → `#`
- `%:%:` → `##`

「`identifier` の構文上の要件を満たさないような代替トークン」はこれらが当てはまると思われる。


調べた感想: 字句解析器か構文解析器が辛そう

