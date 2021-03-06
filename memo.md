### Item 1 p.12の

>These examples all show lvalue reference parameters, but
>type deduction works exactly the same way for rvalue reference parameters.

の意味が分からない。[テキストの間違い](misc/mail.md)

### Item 2のauto a{3}がinitializer_listになるのは気持ち悪い。

C++17で仕様変更された(by [@pepshiso](https://twitter.com/pepshiso/status/560384555257851904))。
[New Rules for auto deduction from braced-init-list](http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2014/n3922.html)

```
                 // C++11, C++14    C++17
auto x = {1};    // 初期化リスト    初期化リスト
auto x = {1, 2}; // 初期化リスト    初期化リスト
auto x{1};       // 初期化リスト    int
auto x{1, 2};    // 初期化リスト    error
```

### Item 3のdecltype(x)とdecltype((x))で意味が変わる例

```
// C++14 only
decltype(auto) f2()
{
    static int x = 0;
    return (x); // ここをreturn xにするとコンパイルエラーになる。
}

int main()
{
    printf("f2()=%d\n", f2());
    f2() = 3;
    printf("f2()=%d\n", f2());
}
```

### Item 6のvector<bool>のoperatorをautoで受けると駄目な話。

[item6-1.cpp](https://github.com/herumi/emcjp/blob/master/src/item6-1.cpp)
VecがintではなくRefを返す。
intで受けるとRef()からintに変換されてから一時オブジェクトのVecとRefが消滅する。

autoで受けるとRef()のままになり、Refの寿命はあるけどそれが参照しているVecは先に消滅するのでそのあとintに変換するとアウト。
```
use int
Vec cstr
Ref cstr
read x
Ref dstr
Vec dstr
type=int a=5
use auto
Vec cstr
Ref cstr
Vec dstr
read x
type=struct Ref a=5
Ref dstr
```

### std::vector<T>::size_typeはsize_tで受けてよいのか?

問題になるとは思えないがどこかで保証してる?

### Item 7

* `{}`のメリット
    * 適用可能なケースが多い
    * 間違えてnarrowingしてしまう可能性が低い
    * Most Vexing Parse(変数宣言が関数のプロトタイプになってしまうやつ)の心配がない
* `()`のメリット
    * C++98の文法と互換
    * initializer_listに関する問題がない

templateの中で一次オブジェクトを作るとき、`{}`と`()`のどちらを使っているかドキュメントに書かないと利用する人はわからないことがある。
[item7-3.cpp](https://github.com/herumi/emcjp/blob/master/src/item7-3.cpp)

### Item 8
整数型とポインタのオーバーロードは止めよう

### Item 9

typedef vs. using
* usingはtemplate化が可能。使うときは`typename`は要らない。`::type`も要らない。
    * typedefのときはdependent type
    * usingはnon-dependent type
    * C++14ではremove_constなどに_tバージョンが追加された。
    * [本家eratta 1/25/15 tyk All](http://www.aristeia.com/BookErrata/emc++-errata.html)にtypedefよりalias templateがよい例が載ってる。

Q. alias templateは特殊化できないけどどうなんだ。→ templateクラスのラップ
* 元のクラスの特殊化と区別できないからしかたない?
* 使う人からするとそんなことを気にせず特殊化したい?

### Item 10
enum Class使え。

Visual Studioでは
```
enum X {
   x = 1234567890123ull
};
```
は32bitを超えた部分が切り捨てられる。
```
enum class X:long long {
    x = 1234567890123ull
};
```
ならOK。

### Item 11
* 代入演算子を使えなくするにはprivateじゃなくて`= delete`の方がいい。
* `= delete`はprivateじゃなくてpublicに書く方がいい。
    * コンパイラはdeletedかどうかより先にaccessibilityをチェックするため。

普通の関数でも`= delete`は使える。
templateでdefault deleteにして使いたいのだけ特殊化、あるいはオーバーロードする。
```
#include <stdio.h>
template <typename T>
void f(T) = delete;

void f(int) { puts("int"); }

int main()
{
    f(3);
    f('a'); // deleteがないとintにmatchしてしまう。
}

```
* char32_tなどは予約語なので`std::`は要らない(erattaに連絡済み)。

### Item 12
override使う。
gccでは-Wall, -Wextraの他に-Woverloaded-vitrualが必要

メンバ関数のreference qualifierのメリットの例

`a + b = 3`を防げる。
```
struct X {
    friend inline X operator+(const X&, const X&) { return X(); }
    X& operator=(int) { return *this; }
    X& operator=(const char*) & { return *this; }
};

int main()
{
    X a, b;
    a + b = 3;   // 通ってしまう
    a + b = "a"; // &がついてるとエラーにできる
}
```

reference qualifierをつけまくったときでgccがambigiousでエラーになるのは4.9.1から直ってました。
[item12-1.cpp](src/item12-1.cpp)

### Item 13
C++03はconst_iteratorは冷遇されていた。C++11からはcbegin, cendを使う。
C++14からはグローバル関数のcbegin, cendを使う。
### Item 14
スタックの巻き戻しに必要なコードを減らせるため可能ならthrow()よりもnoexceptを使う。
```
#include <exception>
#include <stdio.h>

struct A {
    A() { puts("cstr"); }
    ~A() { puts("dstr"); }
};

void f_noexcept() noexcept {
    A a;
    throw 1;
}

void f_nothrow() throw() {
    A a;
    throw 1;
}

int main()
{
#ifdef NOEXCEPT
    puts("noexcept");
    f_noexcept();
#else
    puts("throw()");
    f_nothrow();
#endif
}
```
throw()版なら
```
terminate called after throwing an instance of 'int'

noexcept
cstr
dstr
```
dstrが呼ばれてからstd::unexpected()が呼ばれ、それがstd::terminate()になっている。
noexceptならdstrを呼ばずに即座にstd::terminate()を呼ぶ。
```
terminate called after throwing an instance of 'int'

noexcept
cstr
```
ただし今のところこの最適化はgccのみ。clang(-3.7)では変わらない。

### Item 15
* 可能ならconstexprを使え。
* メンバ関数のconstexprにはconstもつけること(C++14で挙動が変わる)。
* setterにconstexprをつけるとよいことがある。

```
#include <utility>
#include <stdio.h>

struct A {
    static constexpr std::pair<int,int> p{3,5};
};

int main()
{
    printf("A::p.first=%d\n", A::p.first);
}
```
がgccでエラーになるのはバグ? コードが悪い?
=> 多分バグ
