---
title: "Private namespace idiomは合法なのか"
emoji: "🤫"
type: "tech"
topics: ["cpp"]
published: true
---

:::message
この記事は[C++ Advent Calendar 2022](https://qiita.com/advent-calendar/2022/cxx)の2日目の記事…ということになりました．詳細は[こちら](https://twitter.com/wx257osn2/status/1598719220069847041)．
昨日は[lewisacid(`@acd1034`)](https://twitter.com/acd1034)さんの「[メンバ関数の新しい書き方、あるいは Deducing this](https://zenn.dev/acd1034/articles/221117-deducing-this)」でした．
:::

[I](https://twitter.com/wx257osn2)です．今年もC++ Advent Calendarをよろしくお願いいたします．というか枠がメチャクチャに余っているので，各位…よろしく頼む…

# Private namespace idiomは合法なのか

本稿ではPrivate namespace idiomの合法性について解説していきます．なお，突貫で書いたので規格の文面を間違って解釈しているかも知れません．間違いがあれば是非ご指摘お願いします．

## そもそも: Private namespace idiomとは

_Private namespace idiom_ ，あるいは _名前空間を隠蔽するテク_ ^[いずれもIさんが勝手にそう呼んでるだけで，一般に広く知られた名称は無さそう？]とは，C++において名前空間を外から参照できないようにする技法です．
詳細は[C++ Advent Calendar 2015 19日目の記事](https://qiita.com/Chironian/items/0848dd301ac358bc29c6)にて解説されているのでそちらに任せるとして，概要としては以下のような形になります．

```cpp
namespace test{

namespace{  // 無名名前空間内で

namespace detail{  // 適当な名前空間を切り，適当に定義していく

struct type{
  int member_variable;
};

constexpr type f(){
    return {.member_variable = 42};
}

}

}

using detail::f;  // 外部から参照したい名前をここでusingしてから

namespace detail{}  // 名前空間を"閉ざす"

}

int main(){
    static_assert(test::f().member_variable == 42);  // usingで外に出しておいたものについてはそちら経由でアクセス可能
    test::detail::type x{.member_variable = 3};  // detail名前空間内の要素にアクセスできない
}
```

[あなたとWandbox，今すぐ動作確認](https://wandbox.org/permlink/C2TVjy3Su8lTrunh)

便利そうですが，何故これで名前空間を隠蔽できるのか，またこれが合法なコードなのかについての細かい記述をあまり見かけなかった^[大体「うまく隠蔽できる」程度の記述]ため，規格からこれが合法であるかどうかを確認してみましょう．

以下では(一応フィックスしたことになっている――なんか無限にDRと遡及修正が出続けていますが――)C++20の規格書…は高いので，そのドラフトである[N4861](https://timsong-cpp.github.io/cppwp/n4861/)を参照します．

## 無名名前空間の振る舞い

そもそも _無名名前空間_ (_unnamed namespace_) とはなんなのかについて今一度確認しましょう．
無名名前空間は

```cpp
namespace {
    /* ... */
}
```

のような構文を持ち，[無名名前空間およびその内部で定義された名前空間の要素は内部リンケージ](https://timsong-cpp.github.io/cppwp/n4861/basic.link#4)となります([`[basic.link]/4`](https://timsong-cpp.github.io/cppwp/n4861/basic.link#4))．
リンケージを変えるための手段として提供されている無名名前空間ですが，その要素にアクセスする際には無名名前空間は考慮せずにアクセスすることができます．この仕組みについて記載されているのが[`[namespace.unnamed]/1`](https://timsong-cpp.github.io/cppwp/n4861/namespace.unnamed#1)です．以下に省略したものを引用します．

> An [_unnamed-namespace-definition_](https://timsong-cpp.github.io/cppwp/n4861/namespace.def#nt:unnamed-namespace-definition) behaves as if it were replaced by
> 
> ```cpp
> /*略*/ namespace unique { /* empty body */ }
> using namespace unique ;
> namespace unique { namespace-body }
> ```
> 
> (中略) and all occurrences of _unique_ in a translation unit are replaced by the same identifier, and this identifier differs from all other identifiers in the translation unit. (以下略)

邦語訳すると，

```cpp
namespace {
    /* ... */
}
```

は

```cpp
namespace unique {}
using namespace unique;
namespace unique{
    /* ... */
}
```

と同様に扱われ，この `unique` には翻訳単位毎に固有の識別子が与えられる，ということです．

また，[無名名前空間内でexport宣言はできません(`[module.interface]/1`)](https://timsong-cpp.github.io/cppwp/n4861/module.interface#1)．

## Private namespace idiomの振る舞い

無名名前空間が言語仕様上どのように振る舞うかわかったので，もう一度private namespace idiomのコードを見返してみましょう．

```cpp
namespace test{

namespace{  // 無名名前空間内で

namespace detail{  // 適当な名前空間を切り，適当に定義していく

struct type{
  int member_variable;
};

constexpr type f(){
    return {.member_variable = 42};
}

}

}

using detail::f;  // 外部から参照したい名前をここでusingしてから

namespace detail{}  // 名前空間を"閉ざす"

}

int main(){
    static_assert(test::f().member_variable == 42);  // usingで外に出しておいたものについてはそちら経由でアクセス可能
    test::detail::type x{.member_variable = 3};  // detail名前空間内の要素にアクセスできない
}
```

[`[namespace.unnamed]/1`](https://timsong-cpp.github.io/cppwp/n4861/namespace.unnamed#1)から，これは以下のコードと概ね等価です:

```cpp
namespace test{

namespace unnamed{}
using namespace unnamed;
namespace unnamed{

namespace detail{

struct type{
  int member_variable;
};

static constexpr type f(){
    return {.member_variable = 42};
}

}

}

using detail::f;

namespace detail{}

}

int main(){
    static_assert(test::f().member_variable == 42);
    test::detail::type x{.member_variable = 3};
}
```

すると，だいぶ話がわかりやすくなります．まず， `using detail::f;` は `using namespace unnamed;` によって `test` 名前空間上に持ち込まれた `unnamed::detail::f` を `using` 宣言しており，これによって `test::f` によるアクセスが可能となります．
次に， `test::detail::` によるアクセスですが，これは名前空間 `test::detail` に対するqualified name lookupとなりますので， **`test::detail` 名前空間内及び `test::detail` 名前空間に `using` ディレクティブで持ち込まれた名前** の中からその要素を探します([`[namespace.qual]/2,3`](https://timsong-cpp.github.io/cppwp/n4861/namespace.qual))．が， `test::detail` は要素を持たない名前空間であり，またその内部で他の名前空間を `using` ディレクティブで導入してもいないので，結果的に任意のアクセスが失敗するわけです．ここで，もし `test::detail::type` のアクセスに失敗したら `test` 名前空間で `using` ディレクティブによって導入された名前空間の中から `detail::type` に該当する名前を引っ張ってくる，のような再帰的な処理をされていたらこのidiomは成立しないのですが，上述の通りこれは再帰的に処理されないことになっているので，無事にこのようにname hidingできるわけですね．
そして，この読み替え後のコードでは `test::unnamed::detail::type` とすることで(元)無名名前空間内の要素にアクセスできますが，読み替え前のコードではこの `unnamed` に相当する識別子はコンパイラしか知りえない，ユーザーには記述することのできない識別子となるため，晴れて `test::unnamed::detail` 以下へのアクセスを禁じることができました．

## まとめ

というわけで，private namespace idiomはC++規格上合法なコードであることがわかりました．
普段書いているコードの挙動について改めて規格を追ってみると，意外な発見や規格の賢さに出会えます^[私は無名名前空間が `using namespace` で読み替えられる話この記事のために調べるまで知らなかったしqualified lookupの探索範囲の話とかも厳密には知らなかった]．
みなさんも是非チャレンジしてみてください． **そしてC++ Advent Calendar 2022への投稿をよろしくお願いします…**

@[tweet](https://twitter.com/wx257osn2/status/1301148044763697152)

明日(もう今日ですが)は[lewisacid(`@acd1034`)](https://twitter.com/acd1034)さんの「[std::optional のモナド的操作](https://zenn.dev/acd1034/articles/221118-monadic-operation-for-optional)」です．