---
title: "Rust のプリミティブ型はどこからやって来るか"
date: 2021-10-02T09:39:27+09:00
draft: false
tags: ["rust"]
aliases: ['/posts/rust-where-are-primitive-types-from/']
---

この記事は Qiita から移植してきたものです。
元 URL: https://qiita.com/nsfisis/items/9a429432258bbcd6c565


-----------------------------------



# 前置き

Rust において、プリミティブ型の名前は予約語でない。したがって、次のコードは合法である。

```rust
#![allow(non_camel_case_types)]
#![allow(dead_code)]

struct bool;
struct char;
struct i8;
struct i16;
struct i32;
struct i64;
struct i128;
struct isize;
struct u8;
struct u16;
struct u32;
struct u64;
struct u128;
struct usize;
struct f32;
struct f64;
struct str;
```

では、普段単に `bool` と書いたとき、この `bool` は一体どこから来ているのか。rustc のソースを追ってみた。

> 前提知識: 一般的なコンパイラの構造、用語。`rustc` そのものの知識は不要 (というよりも筆者自身がよく知らない)

# 調査

調査に使用したソース (調査時点での最新 master)

https://github.com/rust-lang/rust/tree/511ed9f2356af365ad8affe046b3dd33f7ac3c98

どのようにして調べるか。rustc の構造には詳しくないため、すぐに当たりをつけるのは難しい。

大雑把な構造としては、`compiler` フォルダ以下に `rustc_*` という名前のクレートが数十個入っている。これがどうやら `rustc` コマンドの実装部のようだ。

`rustc` はセルフホストされている (= `rustc` 自身が Rust で書かれている) ので、`bool` や `char` などで適当に検索をかけてもノイズが多すぎて話にならない。
しかし、お誂え向きなことに `i128`/`u128` というコンパイラ自身が使うことがなさそうな型が存在するのでこれを使って `git grep` してみる。

```
$ git grep "\bi128\b" | wc      # i128
     165    1069   15790

$ git grep "\bu128\b" | wc      # u128
     293    2127   26667

$ git grep "\bbool\b" | wc      # cf. bool の結果
    3563   23577  294659
```

165 程度であれば探すことができそうだ。今回は、クレート名を見ておおよその当たりをつけた。

```
$ git grep "\bi128\b"
...
rustc_resolve/src/lib.rs:        table.insert(sym::i128, Int(IntTy::I128));
...
```

`rustc_resolve` というのはいかにも名前解決を担いそうなクレート名である。該当箇所を見てみる。

```rust
/// Interns the names of the primitive types.
///
/// All other types are defined somewhere and possibly imported, but the primitive ones need
/// special handling, since they have no place of origin.
struct PrimitiveTypeTable {
    primitive_types: FxHashMap<Symbol, PrimTy>,
}

impl PrimitiveTypeTable {
    fn new() -> PrimitiveTypeTable {
        let mut table = FxHashMap::default();

        table.insert(sym::bool, Bool);
        table.insert(sym::char, Char);
        table.insert(sym::f32, Float(FloatTy::F32));
        table.insert(sym::f64, Float(FloatTy::F64));
        table.insert(sym::isize, Int(IntTy::Isize));
        table.insert(sym::i8, Int(IntTy::I8));
        table.insert(sym::i16, Int(IntTy::I16));
        table.insert(sym::i32, Int(IntTy::I32));
        table.insert(sym::i64, Int(IntTy::I64));
        table.insert(sym::i128, Int(IntTy::I128));
        table.insert(sym::str, Str);
        table.insert(sym::usize, Uint(UintTy::Usize));
        table.insert(sym::u8, Uint(UintTy::U8));
        table.insert(sym::u16, Uint(UintTy::U16));
        table.insert(sym::u32, Uint(UintTy::U32));
        table.insert(sym::u64, Uint(UintTy::U64));
        table.insert(sym::u128, Uint(UintTy::U128));
        Self { primitive_types: table }
    }
}
```

これは初めに列挙したプリミティブ型の一覧と一致している。doc comment にも、

> All other types are defined somewhere and possibly imported, but the primitive ones need special handling, since they have no place of origin.

とある。次はこの struct の使用箇所を追う。追うと言っても使われている箇所は次の一箇所しかない。なお説明に不要な箇所は大きく削っている。

```rust
    /// This resolves the identifier `ident` in the namespace `ns` in the current lexical scope.
    /// (略)
    fn resolve_ident_in_lexical_scope(
        &mut self,
        mut ident: Ident,
        ns: Namespace,
        // (略)
    ) -> Option<LexicalScopeBinding<'a>> {
        // (略)

        if ns == TypeNS {
            if let Some(prim_ty) = self.primitive_type_table.primitive_types.get(&ident.name) {
                let binding =
                    (Res::PrimTy(*prim_ty), ty::Visibility::Public, DUMMY_SP, ExpnId::root())
                        .to_name_binding(self.arenas);
                return Some(LexicalScopeBinding::Item(binding));
            }
        }

        None
    }
```

関数名や doc comment が示している通り、この関数は識別子 (identifier, ident) を現在のレキシカルスコープ内で解決 (resolve) する。
`if ns == TypeNS` のブロック内では、`primitive_type_table` (上記の `PrimitiveTypeTable::new()` で作られた変数) に含まれている識別子 (`bool`、`i32` など) かどうか判定し、そうであればそれに紐づけられたプリミティブ型を返している。

なお、`ns` は「名前空間」を示す変数である。Rust における名前空間はC言語におけるそれとほとんど同じで、今探している名前が関数名/変数名なのか型なのかマクロなのかを区別している。この `if` は、プリミティブ型に解決されるのは型を探しているときだけだ、と言っている。

重要なのは、これが `resolve_ident_in_lexical_scope()` の最後に書かれている点である。つまり、最初に挙げたプリミティブ型の識別子は、「名前解決の最終段階で」、「他に同名の型が見つかっていなければ」プリミティブ型として解決される。

動作がわかったところで、例として次のコードを考える。

```rust
#![allow(non_camel_case_types)]

struct bool;

fn main() {
    let _: bool = bool;
}
```

ここで `main()` の `bool` は `struct bool` として解決される。なぜなら、プリミティブ型の判定をする前に `bool` という名前の別の型が見つかるからだ。


# まとめ

Rust のプリミティブ型は予約語ではない。名前解決の最終段階で特別扱いされ、他に同名の型が見つかっていなければ対応するプリミティブ型に解決される。

