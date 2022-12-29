---
title: "本当のコンパイル時variant"
emoji: "🔀"
type: "tech"
topics: ["cpp"]
published: true
---

:::message
この記事は[C++ Advent Calendar 2022](https://qiita.com/advent-calendar/2022/cxx)の20日目の記事です(大遅刻)．
19日目は[ラクラムシ(`@raclamusi`)](https://twitter.com/raclamusi)さんの「[C++ コンパイル時「出力」　～C++にできないことはない～](https://qiita.com/Raclamusi/items/418d08f8e64fa1959850)」でした．
:::

[I](https://twitter.com/wx257osn2)です．忘年会シーズンですが皆さん忘年どうですか？私は数日寝込む規模で体調を崩して忘年どころではありませんでした…
さて，寝込んで9日遅れの大遅刻かましたC++ AdC 2022 20日目ですが，当初書こうと思っていたネタはうまくいきませんでしたし(という話はまた別途書きますが)，その他没ネタもいくつかあったのですがいずれも思ったほど膨らまなかったので，C++20のコンパイル時処理の話でもしようと思います．

# 本当のコンパイル時variant

tl;dr:

@[tweet](https://twitter.com/wx257osn2/status/1552335064943718401)

の話をします．

## `constexpr` decade

みなさん今日もコンパイル時処理してますか？ `constexpr` (定数式)の機能がC++11で導入されて10年ちょっと経ちましたね．今日はちょっと昔話でもしましょう．

C++11以前のコンパイル時処理は `template` を用いたメタプログラミング(TMP: Template Meta Programming)かCPPを用いたメタプログラミング(PMP: Preprocessor Meta Programming)の2択で，これらはいずれも実行時のC++プログラムとはかなり異なる構文や意味を用いるため扱いにはかなり慣れが必要でした．
C++11で導入された `constexpr` は普通のC++コードがコンパイル時に動く，という当時では画期的な機能であり，これによってコンパイル時処理における値の計算にTMPを用いた奇っ怪なコードが不要になり，型の処理は `template` ，値の処理は `constexpr` …という住み分けができるようになりました．

ところで，C++11時代の定数式は **死ぬほど何もできませんでした** ．
具体的には，C++11時代の `constexpr` 関数には以下のような制約が課されます(これでも一部ですが):

1. ユーザー定義デストラクタは使用できない
1. コンストラクタはメンバ初期化子でメンバを初期化する以外のことはできない(bodyを持ってはいけない)
1. 可変参照(非const参照)は取れない
1. `typedef` , `using` , `static_assert` , 単一の `return` 文以外書けない
    - 型の別名は作れるが(変数宣言の形で)値に名前付けできない
    - 一般的な条件分岐・ループ構文(`if` , `for` , `while` など)が使えない
1. `reintepret_cast` やplacement newは使えない

これらのびっくりするほど厳しそうに見える制約はC++14やC++20を経て徐々に緩和され今では殆どが無くなっていますが，ほんの10年前はこの過酷な環境下で「コンパイル時処理楽しい！」と言いながらコンパイラにICEを吐かせていました．
また，C++11ではラムダ式も導入されましたが，[ラムダ式が `constexpr` な文脈で使用できるようになった・ `cosntexpr` 指定を明示できるようになったのはC++17以降](https://cpprefjp.github.io/lang/cpp17/constexpr_lambda.html)です．
つまり，C++11当時は `constexpr` の文脈でラムダ式は使用できなかったんですね．
その結果，[C++11 constexprで使えるラムダ式ライブラリを自作する人](https://qiita.com/wx257osn2/items/72ecebc0fb127d60c8b1)が現れたりしました．一体何者なんだ…

さて，上述のような極めて制約されたC++11環境で `constexpr` はまともに実用し得たのかというと…当時の時点で最早悪用の域に達していました． **だってC++11の時点で概ねチューリング完全だったし…** ^[死ぬほど何もできない(できないとは言ってない)]
関数内では単一の `return` 文しか記述できず，また条件分岐やループなどに `if` や `for` などの構文が使用できないというのは先に述べた通りですが，実際には条件分岐に条件演算子( `?:` )，ループには関数の再帰呼び出しを用いることで普通に(というには一般的なC++コードとはだいぶ見た目が違いましたが――少なくともC++03の `template` ゴリゴリのメタプロに比べれば，普通のC++の見た目に近い形で)プログラミングできてしまったんですね．
ただし，コンパイル時の再帰呼び出しの深度には処理系側で制限が課されていることがほとんどで^[現実的にコンパイル時無限ループを止めるには妥当な制約です，停止性を判定するのは大変なので]，素直にコンパイル時処理を記述すると結構な頻度でこれに引っかかってしまう問題がありました．
そこで，これを解決すべく様々な技法が編み出されたりしたのですが…今日の本筋からは逸れるので，これはまた別の機会に．

## Sprout

近年では随分と名前を聞く機会も減ってきましたが，[Sprout](https://github.com/bolero-MURAKAMI/Sprout)というC++11/14時代の有名な汎用 `constexpr` ライブラリがあります．
作者の故・ボレロ村上氏は上述の処理系の制約を回避しつつ実用的な処理を記述するために倍分再帰などの様々な技法を編み出した人物ですが，その結晶がSproutです．
SproutはC++11 `constexpr` にとっていわばC++03時代のBoostのような，実用上必須な準標準規模ライブラリの立ち位置であり，基本的なデータ構造，アルゴリズム関数を始めとした様々なライブラリ群で構成されています(そしてその殆どがC++11 `constexpr` の制約のもとで実装されている)^[他にもコンパイル時レイトレーシング・コンパイル時音声波形処理・コンパイル時パーサーコンビネーターなど，誰が使うんだよみたいなライブラリもありますが，いずれもC++11の頃から `constexpr` がそれだけ強力な機能であったという証左でもあります]．

### Sprout.Variant ―― 偽りのコンパイル時variant

そしてSproutにはC++11時点では `constexpr` ではなかった `tuple` や，当時から必要性こそ叫ばれていたものの標準入りはC++17にずれ込んだ `optional` や `variant` についても実装されています．しかも全部コンパイル時処理に使えちまうんだ！
では `sprout::variant` の実装を眺めてみましょう:

https://github.com/bolero-MURAKAMI/Sprout/blob/6b5addba9face0a6403e66e7db2aa94d87387f61/sprout/variant/variant.hpp#L35-L74

```cpp
			typedef sprout::tuples::tuple<Types...> tuple_type;
```

```cpp
			tuple_type tuple_;
```

おや？ `variant` が `tuple` で実装されています．これはどうしたことでしょう？

#### `tuple` と `variant`

ここで念のため `tuple` と `variant` とは何なのか説明しておきましょう．
`tuple<Ts...>` は `Ts...` の型の値全てを保持する型です．
全ての型の値を同時に保持する必要があるため， `tuple<Ts...>` のオブジェクトサイズは単純に考えて `sizeof(Ts)...` の和と同程度になることが予想できます^[実際にはパディングとかが入ってくるので全く同じにはならない可能性があります]．
一方， `variant<Ts...>` は `Ts...` のうちいずれか1つを保持する型です．
いずれか1つの値を保持できれば良いため， `variant<Ts...>` のオブジェクトサイズは単純に考えて `std::max({sizeof(Ts)...})` と同程度に抑えることが可能なことがわかります^[実際にはどの型のオブジェクトを抱えているかを識別するためのタグも必要なのでもう少し大きくなります]．

他にも `tuple` と `variant` にはいくつか想定される挙動があるため， `variant` を `tuple` で実装してしまうと `variant` に想定される挙動に対して以下のような差異が出てしまいます:

- オブジェクトサイズが大きくなる
    - 上述の通り， `variant<Ts...>` は `Ts...` の中で最大のオブジェクトのサイズと同程度であることが期待されますが， `tuple<Ts...>` で実装すると `Ts...` のサイズの総和と同程度の大きさとなってしまいます．
- 未使用の型のオブジェクトが `variant<Ts...>` 本体と同じ寿命で生成されてしまう
    - 本来 `variant<Ts...>` では `Ts...` のうち保持していない型のオブジェクトは生成されませんが， `tuple<Ts...>` では当然全ての型のオブジェクトが生成されてしまいます(し，破棄もされます)．
    - これはコンパイル時の文脈ではあまり気にならない性質ですが， `constexpr` なクラスは実行時でも使えてしまうため，コンストラクタやデストラクタで副作用を伴うクラスを非コンパイル時文脈で `sprout::variant` に食わせると一見不可解な挙動を起こします．
        - 尤も，これはコンパイル時の文脈に限って言えばあまり気にしなくて良いため， `constexpr` のための `variant` だから…と言い張ればまぁ逃げれなくもない
- `Ts...` に対して `default_constructible` を要求する
    - 本来 `variant<Ts...>` の `Ts...` は(ユーザーコード上でデフォルト初期化しないのであれば)デフォルトコンストラクタを持たなくても問題ありません．
    - `tuple<Ts...>` で `Ts...` のどれか1つを何かのコンストラクタで初期化する場合，他の型のオブジェクトは現実的にはデフォルト構築するしかありません．そのため， `variant<Ts...>` を `tuple<Ts...>` で構築する場合は `Ts...` (のうち1つ以外)に対して `default_constructible` を要求することになります．

このように `tuple` で `variant` を実装するのはあまり自然に思えません． `sprout::variant` は何故 `tuple` を用いて実装されているのでしょうか？

## `constexpr` な `either` を考える

実際に `constexpr` な `variant` を実装すればこの謎は解明できそうですね．
といっても，任意の数の型を扱おうとするとけっこう大変なので，本節では `constexpr` な `either<T, U>` ，つまり型 `T` または型 `U` のオブジェクトいずれかを保持する型の実装を考えることでこれを代替したいと思います．
2つができればn個もできる．

### 要件

本節では以下のような `template` クラス `either<T, U>` を考えます．

- `constexpr` な文脈で扱える
    - `T` , `U` がそれぞれコンパイル時の文脈で扱える時， `either<T, U>` もまた同程度にコンパイル時の文脈で扱えること
    - 簡単のため，非コンパイル時文脈における使用ついては考慮しないものとする
        - 具体的には `T` および `U` についてリテラル型(死語^[[C++20以降，「リテラル型」という考え方はやめようという話になりました](https://cpprefjp.github.io/reference/type_traits/is_literal_type.html)])であることを仮定する
            - 特に `trivially_destructible` であることを利用する
- `either<T, U>(left, args...)` で `T(args...)` ， `either<T, U>(right, args...)` で `U(args...)` なオブジェクトの初期化が行えること
- `either<T, U>` なオブジェクト `e` に対して， `e.get(left)` で `T` のオブジェクト， `e.get(right)` で `U` のオブジェクトが得られること
    - 保持していない側のオブジェクトを取得しようとしたとき，コンパイル時の文脈ではコンパイルエラーとすること
- オブジェクトサイズは `std::max(sizeof(T), sizeof(U)) + sizeof(bool) + (パディングのサイズ)` となること
    - `T` および `U` のうち大きい方が収まるストレージ + `T` と `U` どちらが保持されているかの `bool` フラグ + コンパイラが挟むパディングサイズ

つまり，以下のようなコードが動くことが期待されます:

```cpp
constexpr either<int, float> f(bool integer){
    return integer ? either<int, float>{left, 42} : either<int, float>{right, 3.14f};  // intまたはfloatを引数に応じて返す
}

int main(){
    constexpr auto x = f(true);   // int, 42
    constexpr auto y = f(false);  // float, 3.14
    constexpr auto z = x;         // int, 42 のコピー
    static_assert(y.get(right) < z.get(left), "");  // 3.14f < 42 -> true
}
```

### 同一ストレージへの複数の型によるアクセス

上述の通り `either<T, U>` はオブジェクトサイズを `T` および `U` のうち大きい方に合わせようとするため，あるオブジェクトストレージに対して `T` および `U` 2つの型でアクセスする必要があります．
これを実現する方法は大まかに2つあり，

- `alignas(T, U) std::byte storage[std::max(sizeof(T), sizeof(U))]` な配列 `storage` を用意し，その先頭アドレスを `reinterpret_cast` を用いて `T*` と `U*` に変換して利用する
- `union{T t; U u;}` な共用体を用意し， `t` と `u` を用いる

の2通りです．このうち前者については `reinterpret_cast` なり `void*` からの `static_cast` なりが必要であり，これらの操作はC++20になっても `constexpr` 文脈では使用できないため今回使うことはできません．
従って，必然的に後者，つまり共用体を用いたオブジェクトストレージの共有を行うことになります^[で，型の数が任意の数あるとこの共用体をいい感じに木構造に落とす必要があって，本稿の本質から離れる割に実装がめんどくさい…ということで `variant` ではなく `either` で説明することにしました]．

ところで，C++03までは `union` のメンバにユーザー定義のコンストラクタやデストラクタを持つクラスを持たせることができませんでした．
そのため，例えば

```cpp
class just_initialization{
    int x;
  public:
    constexpr just_initialization() : x{0}{}
    constexpr just_initialization(int i) : x{i}{}
    constexpr int get()const{return x;}
};
```

のようなクラスを `union` のメンバに持たせることができませんでした．
しかし，[都合よくC++11ではこの制限は緩和されたため](https://cpprefjp.github.io/lang/cpp11/unrestricted_unions.html)，問題なく `union` にクラスメンバを持たせることができます．
というわけで， `union` を用いてパパっと実装してみましょう:

```cpp
#include<type_traits>
#include<utility>
#include<stdexcept>

struct left_t{}static constexpr left = {};
struct right_t{}static constexpr right = {};

namespace detail{

template<typename T, typename U>
union storage_t{
  static_assert(std::is_literal_type<T>::value, "T is not literal type");
  static_assert(std::is_literal_type<U>::value, "U is not literal type");
  T t;
  U u;
  template<typename... Args>
  constexpr storage_t(left_t, Args&&... args)noexcept(std::is_nothrow_constructible<T, Args&&...>::value) : t(std::forward<Args>(args)...){}
  template<typename... Args>
  constexpr storage_t(right_t, Args&&... args)noexcept(std::is_nothrow_constructible<U, Args&&...>::value) : u(std::forward<Args>(args)...){}
  constexpr storage_t(const storage_t&) = default;
  constexpr storage_t(storage_t&&) = default;
  static constexpr storage_t copy(const storage_t& x, bool is_left){
      return is_left ? storage_t{left, x.t} : storage_t{right, x.u};
  }
  static constexpr storage_t move(storage_t&& x, bool is_left){
      return is_left ? storage_t{left, std::move(x.t)} : storage_t{right, std::move(x.u)};
  }
};

template<typename T, typename U>
struct base{
  bool is_left;
  storage_t<T, U> storage;
  template<typename... Args>
  explicit constexpr base(left_t, Args&&... args)noexcept(std::is_nothrow_constructible<T, Args&&...>::value) : is_left(true), storage(left, std::forward<Args>(args)...){}
  template<typename... Args>
  explicit constexpr base(right_t, Args&&... args)noexcept(std::is_nothrow_constructible<U, Args&&...>::value) : is_left(false), storage(right, std::forward<Args>(args)...){}
  constexpr base(const base& other) : is_left{other.is_left}, storage{storage_t<T, U>::copy(other.storage, is_left)}{}
  constexpr base(base&& other) : is_left{std::move(other.is_left)}, storage{storage_t<T, U>::move(std::move(other.storage), is_left)}{}
};

template<typename T>
constexpr T throw_with_cref(const char* str){
    throw std::runtime_error(str);
}

}

template<typename T, typename U>
class either{
  detail::base<T, U> data;
 public:
  constexpr either() = default;
  template<typename LorR, typename... Args>
  constexpr either(LorR x, Args&&... args) : data{x, std::forward<Args>(args)...}{}
  constexpr either(const either& other) : data{other.data}{}
  constexpr either(either&& other) : data{std::move(other.data)}{}
  constexpr const T& get(left_t)const noexcept{
      return data.is_left ? data.storage.t : detail::throw_with_cref<const T&>("not left");
  }
  constexpr const U& get(right_t)const noexcept{
      return !data.is_left ? data.storage.u : detail::throw_with_cref<const U&>("not right");
  }
};

class just_initialization{
    int x;
  public:
    constexpr just_initialization() : x{0}{}
    constexpr just_initialization(int i) : x{i}{}
    constexpr int get()const{return x;}
};

constexpr either<int, float> f(bool integer){
    return integer ? either<int, float>{left, 42} : either<int, float>{right, 3.14f};
}

int main(){
    constexpr auto x = f(true);
    constexpr auto y = f(false);
    constexpr auto z = x;
    static_assert(y.get(right) < z.get(left), "");

    constexpr either<just_initialization, int> s{left};
    static_assert(s.get(left).get() == 0, "");
    constexpr either<just_initialization, int> t{left, 42};
    constexpr auto u = t;
    static_assert(u.get(left).get() == 42, "");
}
```

[あなたとWandbox，今すぐ動作確認](https://wandbox.org/permlink/Dj4pv35X2qzFKNF5)

どうやら上手く動いているようですね．…あれ？では `sprout::variant` は何故 `tuple` を…？

#### 汎用実装の難しさ

実際には，上記の実装は不完全です．以下の型を考えます:

```cpp
struct metamorphose_at_copy{
  bool copied;
  constexpr metamorphose_at_copy() : copied{false}{}
  constexpr metamorphose_at_copy(const metamorphose_at_copy&) : copied{true}{}
};
```

この型は以下のように振る舞います．

```cpp
constexpr metamorphose_at_copy a;
static_assert(s.copied == false, "");  // コピーされていない
constexpr auto b = a;
static_assert(t.copied == true, "");   // コピーされた
```

さて，この型は非trivialなコピーコンストラクタを持っており，そのためムーヴコンストラクタは削除されています．
そのような型であっても `union` はメンバにすることが可能ですが，メンバが非trivialなデフォルト・コピー・ムーヴコンストラクタやデストラクタを持つ `union` は当該コンストラクタ・デストラクタがデフォルトで `delete` されます．
そのため，上記の `either` と組み合わせた以下のコードは `storage_t` のコピー/ムーヴコンストラクタが存在しないため動作しません^[ちなみに単にこのコードを動かしたいだけであれば， `metamorphose_at_copy` にtrivialなムーヴコンストラクタを生やせば `storage_t` のムーヴコンストラクタが生成されるため `base` のコピーコンストラクタが動くようになります]．

```cpp
constexpr either<metamorphose_at_copy, int> a{left};
static_assert(a.get(left).copied == false, "");
constexpr auto b = a;                                // ここでエラー
static_assert(b.get(left).copied == true, "");
```

このように， `constexpr` 文脈で問題なく動作するが `either` のメンバとしては正しく振る舞えない型が存在するため，上記の実装では

> `T` , `U` がそれぞれコンパイル時の文脈で扱える時， `either<T, U>` もまた同程度にコンパイル時の文脈で扱えること

の要件を満たすことができません．
そして，単にプリミティブ型の値，特に整数値を取り回すだけであれば従来のTMPでもできたわけで， `constexpr` はクラスを作って取り回せる点が1つの強みなわけです．
となると，非trivialなコピー・ムーヴコンストラクタに対してサポートが貧弱な現行の `either` では使用感にかなり難があります．

この問題を解決するためには， `base` のコピーコンストラクタから直接 `t` か `u` のコンストラクタを呼び出してやるか，あるいは `storage_t` にユーザー定義のコピーコンストラクタを実装する必要があります．ではどのように実装するのか，というと， **実はC++11では `constexpr` な実装はできません** ．
先述の通りC++11 `constexpr` においてコンストラクタはbodyが持てないため，メンバ初期化子で初期化してやる必要があります．
しかし，初期化子は引数の値に応じて書いたり書かなかったりできません．
従って，フラグに応じて `t` と `u` の初期化を呼び分ける…といったコードは実装できないのです．
従って `base` から直接 `t` や `u` のコピーコンストラクタを呼び出すことはできません(そのため `copy` や `move` といった静的メンバ関数を `storage_t` に実装しておいたのですが…)．
また `storage_t` 側ではコピーコンストラクタ上で `is_left` を読む術がないため，これもやはり無理です．
この時点で，C++11 `constexpr` において `variant` を `union` で実装するのは無理があることがわかりました．

ではC++14ではどうでしょう？C++14ではコンストラクタのbodyが空でなくても良いはずですし， `if` 文も使えます．
`base` のコピーコンストラクタで `is_left` の値に応じて `t` または `u` を初期化するコードをbodyに書いてやればいけそうです．
以下のようなコードになります:

```cpp
  base(const base& rhs) : is_left(rhs.is_left){
    if(rhs.is_left)
      ::new(&storage.t) T(rhs.storage.t);
    else
      ::new(&storage.u) U(rhs.storage.u);
  }
```

…あれ？ **`constexpr` でなくなってしまいました…** コンストラクタを呼び出すためにはplacement newする必要がありますが， **placement newは `constexpr` 文脈で使用できません** ．
C++11の時と同様にメンバ初期化子での初期化はできないため，C++14でも `constexpr` 文脈でメンバコピーが可能な `either` を実装することはできません．

また，C++14では `constexpr` の制限緩和によって `constexpr` 文脈中でのコピー代入・ムーヴ代入が可能になったため，それらについても考える必要があります．
しかし， **C++14の時点では `union` のアクティブメンバは `constexpr` の文脈では変更できないため，代入操作について `constexpr` にすることはできません** ．

そしてこの状況はC++17でも改善されていません．

### 型を捻じ曲げるということ

ここまでで見てきたように，C++17までで `union` を用いて `either` を実装すると，一部の型に対してコピーコンストラクトができなくなること，代入操作全般が `constexpr` にできないことがわかりました．
`constexpr` では種々の操作が制約されてきましたが，とりわけ強く制約され続けてきた操作として「型を捻じ曲げる操作」が挙げられます．
具体的には， `reinterpret_cast` やplacement new， `union` のアクティブメンバ変更が規制されてきたため， **同一オブジェクトストレージを複数の型と見做して扱うことができませんでした** ．
これは処理系の実装の負担を考えると一定程度頷ける制限ではある一方で，実行時の性能を充分に保った上でコンパイル時にも実行可能な処理を記述する際の妨げとなります．

以上のことから， `sprout::variant` の実装が _誤っていた_ わけではありません．そもそもC++17までは **実用上 `constexpr` な `variant` のバックエンドに `tuple` 以外を使うことができなかった** のです．

### C++20 : goal of the decade

結局こうした状況が改善したのはC++20になります．
C++20では指定したアドレス上に `constexpr` にオブジェクトを構築する[`std::construct_at`](https://cpprefjp.github.io/reference/memory/construct_at.html)が導入され，また指定されたアドレスのオブジェクトを破棄する[`std::destroy_at` も `constexpr` に対応](https://cpprefjp.github.io/reference/memory/destroy_at.html)しました．また，デストラクタに `constexpr` が付けられるようになり，非 `trivially_destructible` な型であっても `constexpr` なデストラクタを持っている型については `constexpr` 文脈で扱えるようになりました．
これによってコンストラクタやデストラクタについてのtrivial性に関わらず `constexpr` 文脈で使用可能な型は `either` でも扱えるようになりました．
さらに[`union` のアクティブメンバを `constexpr` 文脈で変更可能](https://cpprefjp.github.io/lang/cpp20/changing_the_active_member_of_a_union_inside_constexpr.html)になり，これによって代入操作も `constexpr` にサポートできるようになりました．

これらを反映したのが以下のコードです^[`swap` の実装が例外安全でない点に注意．面倒なので今回は端折りました．まぁコンパイル時に使う分には困らないし…]:

```cpp
#include<memory>
#include<type_traits>
#include<utility>
#include<stdexcept>

struct left_t{} static constexpr left = {};
struct right_t{} static constexpr right = {};

namespace detail{

template<typename T, typename U>
union storage_t{
 private:
  std::byte dummy;
 public:
  T t;
  U u;
  constexpr storage_t()noexcept : dummy(){}
  template<typename... Args>
  constexpr storage_t(left_t, Args&&... args)noexcept(std::is_nothrow_constructible<T, Args&&...>::value) : t(std::forward<Args>(args)...){}
  template<typename... Args>
  constexpr storage_t(right_t, Args&&... args)noexcept(std::is_nothrow_constructible<U, Args&&...>::value) : u(std::forward<Args>(args)...){}
  constexpr ~storage_t()noexcept{}
};

template<typename T, typename U>
struct base{
  std::int8_t is_left;
  storage_t<T, U> storage;
  constexpr base()noexcept : is_left(-1), storage(){}
  template<typename... Args>
  explicit constexpr base(left_t, Args&&... args)noexcept(std::is_nothrow_constructible<T, Args&&...>::value) : is_left(1), storage(left, std::forward<Args>(args)...){}
  template<typename... Args>
  explicit constexpr base(right_t, Args&&... args)noexcept(std::is_nothrow_constructible<U, Args&&...>::value) : is_left(0), storage(right, std::forward<Args>(args)...){}
  constexpr base(const base& rhs)noexcept(std::is_nothrow_copy_constructible<T>::value && std::is_nothrow_copy_constructible<U>::value) : is_left(rhs.is_left), storage{}{
    if(rhs.is_left == 1)
      std::construct_at(std::addressof(storage.t), rhs.storage.t);
    else if(rhs.is_left == 0)
      std::construct_at(std::addressof(storage.u), rhs.storage.u);
    else
      throw std::runtime_error("rhs is not initialized");
  }
  constexpr base(base&& rhs)noexcept(std::is_nothrow_move_constructible<T>::value && std::is_nothrow_move_constructible<U>::value) : is_left(rhs.is_left), storage{}{
    if(rhs.is_left == 1)
      std::construct_at(std::addressof(storage.t), std::move(rhs.storage.t));
    else if(rhs.is_left == 0)
      std::construct_at(std::addressof(storage.u), std::move(rhs.storage.u));
    else
      throw std::runtime_error("rhs is not initialized");
  }
  constexpr base& operator=(const base& rhs)noexcept(std::is_nothrow_copy_assignable<T>::value && std::is_nothrow_copy_assignable<U>::value){
    if(this == std::addressof(rhs))
      return *this;
    if(is_left == 1)
      std::destroy_at(std::addressof(storage.t));
    else if(is_left == 0)
      std::destroy_at(std::addressof(storage.u));
    if(rhs.is_left == 1)
      std::construct_at(std::addressof(storage.t), rhs.storage.t);
    else if(rhs.is_left == 0)
      std::construct_at(std::addressof(storage.u), rhs.storage.u);
    is_left = rhs.is_left;
    return *this;
  }
  constexpr base& operator=(base&& rhs)noexcept(std::is_nothrow_move_assignable<T>::value && std::is_nothrow_move_assignable<U>::value){
    if(this == std::addressof(rhs))
      return *this;
    if(is_left == 1)
      std::destroy_at(std::addressof(storage.t));
    else if(is_left == 0)
      std::destroy_at(std::addressof(storage.u));
    if(rhs.is_left == 1)
      std::construct_at(std::addressof(storage.t), std::move(rhs.storage.t));
    else if(rhs.is_left == 0)
      std::construct_at(std::addressof(storage.u), std::move(rhs.storage.u));
    is_left = rhs.is_left;
    return *this;
  }
  constexpr ~base()noexcept(std::is_nothrow_destructible<T>::value && std::is_nothrow_destructible<U>::value){
    if(is_left == 1)
      std::destroy_at(std::addressof(storage.t));
    else if(is_left == 0)
      std::destroy_at(std::addressof(storage.u));
  }
};

template<typename T>
constexpr T throw_with_cref(const char* str){
    throw std::runtime_error(str);
}

}

template<typename T, typename U>
class either{
  detail::base<T, U> data;
 public:
  constexpr either() = default;
  template<typename LorR, typename... Args>
  constexpr either(LorR x, Args&&... args) : data{x, std::forward<Args>(args)...}{}
  constexpr either(const either& other) : data{other.data}{}
  constexpr either(either&& other) : data{std::move(other.data)}{}
  constexpr either& operator=(const either& other){
    either<T, U>{other}.swap(*this);
    return *this;
  }
  constexpr either& operator=(either&& other){
    either<T, U>{std::move(other)}.swap(*this);
    return *this;
  }
  constexpr void swap(either& other){
    if(other.data.is_left == data.is_left){
      using std::swap;
      swap(data, other.data);
    }
    else if(data.is_left){
      U u = std::move(other.data.storage.u);
      std::destroy_at(std::addressof(other.data.storage.u));
      std::construct_at(std::addressof(other.data.storage.t), std::move(data.storage.t));
      std::destroy_at(std::addressof(data.storage.t));
      std::construct_at(std::addressof(data.storage.u), std::move(u));
      std::swap(data.is_left, other.data.is_left);
    }
    else
      other.swap(*this);
  }
  constexpr T& get(left_t)noexcept{
      return data.is_left ? data.storage.t : detail::throw_with_cref<const T&>("not left");
  }
  constexpr U& get(right_t)noexcept{
      return !data.is_left ? data.storage.u : detail::throw_with_cref<const U&>("not right");
  }
  constexpr const T& get(left_t)const noexcept{
      return data.is_left ? data.storage.t : detail::throw_with_cref<const T&>("not left");
  }
  constexpr const U& get(right_t)const noexcept{
      return !data.is_left ? data.storage.u : detail::throw_with_cref<const U&>("not right");
  }
};

class just_initialization{
    int x;
  public:
    constexpr just_initialization() : x{0}{}
    constexpr just_initialization(int i) : x{i}{}
    constexpr int get()const{return x;}
};

struct metamorphose_at_copy{
  bool copied;
  constexpr metamorphose_at_copy() : copied{false}{}
  constexpr metamorphose_at_copy(const metamorphose_at_copy&) : copied{true}{}
};

constexpr either<int, float> f(bool integer){
  either<int, float> x{right, 3.14f};
  if(integer)
    x = {left, 42};
  return x;
}

#include<iostream>
int main(){
    constexpr auto x = f(true);
    constexpr auto y = f(false);
    constexpr auto z = x;
    static_assert(y.get(right) < z.get(left), "");

    constexpr either<just_initialization, int> s{left};
    static_assert(s.get(left).get() == 0, "");
    constexpr either<just_initialization, int> t{left, 42};
    constexpr auto u = t;
    static_assert(u.get(left).get() == 42, "");

    constexpr either<metamorphose_at_copy, int> a{left};
    static_assert(a.get(left).copied == false, "");
    constexpr auto b = a;
    static_assert(b.get(left).copied == true, "");
}
```

[あなたと犬小屋，今すぐ動作確認](https://wandbox.org/permlink/lYJZxUOShKXv9GiD)

## まとめ

従来 `constexpr` な文脈において `variant` のインターフェースを充分に満たすためにはバックエンドに `tuple` を用いて余分なオブジェクトストレージを消費せざるを得ませんでした．
しかし過去のC++では不可能だった同一オブジェクトストレージに対する複数の型を用いたアクセスを含む， `constexpr` 文脈における様々な操作がC++20でサポートされた結果，余分なオブジェクトストレージを消費することなく `constexpr` 文脈で動作する `variant` の実装が可能になりました．
これは `std::variant` にも反映されており， `trivially_destructible` であればC++20で，そうでない場合でもC++23から `constexpr` な文脈で `std::variant` を利用可能です^[これ若干怪しい．C++20に対する遡及修正アリのDRとしてこの修正が入っていたらC++20でも行けます(ちゃんと調べてない)]．

この10年で `constexpr` は厳しく制約された環境に対してかなりの緩和が入り，その過程で見出された一部の技法は現代では不要なものとなっています．
しかし，C++23で入る `if consteval` やC++20で入った `constexpr` 関数内での非コンパイル時文脈におけるinline asmの合法化によって，単一のインターフェースで実行時性能を損なうことなくコンパイル時実行可能な処理を記述できる時代がついにやってきました．
みなさんも積極的に `constexpr` していきましょう．

最後にお知らせですが，[昨年](https://twitter.com/wx257osn2/status/1466118644522708993)に引き続き[今年もC++ Advent Calendar全記事への感想ツイートを書いています](https://twitter.com/wx257osn2/status/1587328330537500672)．
記事内容の解説や補足・関連する話題など，AdCをもう一口楽しめるかと思いますので，是非ご覧ください．

明日は[yumetodo(`@yumetodo`)](https://twitter.com/yumetodo)さんの「[std::optionalとの比較演算子に潜む見落としがちな罠](https://qiita.com/yumetodo/items/aac8db2aacb6faf869e9)」です．
