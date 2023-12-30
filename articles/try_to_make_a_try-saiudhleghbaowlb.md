---
title: "Try to make a `try` !"
emoji: "🦊"
type: "tech"
topics: ["cpp"]
published: true
---

:::message
この記事は[C++ Advent Calendar 2023](https://qiita.com/advent-calendar/2023/cxx)の20日目の記事です(大遅刻)．
19日目は[ラクラムシ(`@raclamusi`)](https://twitter.com/raclamusi)さんの「[C++ コンパイル時パスワード認証　〜コードを不正コンパイルから守ろう！〜](https://qiita.com/Raclamusi/items/89b9b7df0cd61e285a54)」でした．
:::

[I(`@wx257osn2`)](https://twitter.com/wx257osn2)です．

:::details ご挨拶
各位，今年もC++ Advent Calendar 2023にご参加いただきありがとうございます！
今年はなんか数ヶ月単位で労基も真っ青な過酷な労働を強いられてしまい，全くAdCについて気にかける(カレンダーの余白を眺めて宣伝する，記事を読んで感想を書くなどする)余裕がありませんでした…(普通に過労死ラインに乗ってしまったし案の定心療内科通いになっちゃったぐらいなので，ここ暫くは自身の生死や体調・今後しか気にかける余裕がなかった)
そんな中，(本当に全く経時で追えてないので「今日空いてんじゃん，記事書くか…！」していただいた方々も何人かいらっしゃったかもしれませんが)ちゃんとカレンダーが埋まったことはとてもありがたいことです．
皆様本当にありがとうございます．

過去2年間続けてきた感想ツイート([2021](https://twitter.com/wx257osn2/status/1466118976682217472)，[2022](https://twitter.com/wx257osn2/status/1598070064791851008))についてですが，記事執筆がだいぶ遅れてしまったこと，年末バタバタする等の理由で年が明けてからゆっくり読む予定です．
季節イベントなのにリアルタイムで盛り上げられなくて本当にすまない…ちゃんと感想は書くのでゆるして

AdC延長戦も(やってる人がいたかわかりませんが)見つけたら読もうと思います．書いてくれた人は教えてね．
:::

というわけで2023年だしC++23で追加された `std::expected` の話をしようと思います．

# `std::expected`

さて，過去に書いた人もいただろうから解説は任せようと思います．

おらんやんけ

---

`std::expected<T, E>` はC++23から導入されたクラスで正常値 `T` または異常値 `E` のいずれかを保持することができます．
C++17から導入された `std::optional<T>` は正常値 `T` または不正状態を持つことと対比すると，操作の失敗理由を持てるよう拡張されたと見做すことができます．
他言語において対応する概念としてはRustにおける `std::Result` ， Haskellにおける `Either` ^[本来 `Either` はタグ付き共用型(を表現する型クラス)であって `std::expected` のようにエラー処理に特化したものではない(こちらはタグ付きではないものの立ち位置的には `std::variant` に近い)のですが，Haskellでは一般に失敗を伴う関数の結果型を `Either E T` で表現しがち]などがあります．

以下のように使うことができます．

```cpp:expectedの使用例
#include<expected>
#include<iostream>
std::expected<int, std::string> safe_div(int x, int y){  // 正常値int，または異常値std::stringを返す関数
  if(y == 0)
    return std::unexpected<std::string>("y is zero");  // unexpected<E>型の値から暗黙構築可能
  return x / y;  // T型の値から暗黙構築可能
}

std::expected<double, std::string> f(int x, int y){  // 正常値double，または異常値std::stringを返す関数
  const std::expected<int, std::string> v0 = safe_div(x, y);
  if(not v0)  // contextually convertible to bool(正常値でtrue，異常値でfalse)
    return std::unexpected{v0.error()};  // v0が異常値を持っている場合，error()メンバ関数でアクセス可能
  const std::expected<int, std::string> v1 = safe_div(x, 3);
  if(not v1)
    return std::unexpected{v1.error()};
  return *v0 + *v1;  // 正常値を保つ場合，operator*()でアクセス可能
}

int main(){
  const std::expected<double, std::string> x = f(51, 2);
  std::cout << x.value() << std::endl;  // value()メンバ関数で正常値にアクセス
  try{
    const std::expected<double, std::string> y = f(42, 0);
    if(not y)
      std::cout << "actually y has: " << y.error() << std::endl;
    std::cout << y.value() << std::endl;  // value()メンバ関数は異常値の場合例外を送出
  }catch(std::bad_expected_access<std::string>& e){  // std::bad_expected_access<E>例外が送られる
    std::cout << "caught: " << e.error() << std::endl;  // bad_expected_accessもまたerror()メンバ関数でその値を読める
  }
}
```

```:出力結果
42
actually y has: y is zero
caught: y is zero
```

## 冗長な表記， An error propagation operator

さて，上記のサンプルコード，正直記述量が多いです． `f()` について見てみましょう．


```cpp:抜粋(ローカル変数定義の型名はautoに置換)
std::expected<double, std::string> f(int x, int y){
  const auto v0 = safe_div(x, y);
  if(not v0)
    return std::unexpected{v0.error()};
  const auto v1 = safe_div(x, 3);
  if(not v1)
    return std::unexpected{v1.error()};
  return *v0 + *v1;
}
```

値を取って，エラーか確認して，エラーだったらその場で返す，という定形コードが見られます．
なんだかGo言語みたいです．
これをもっとなんかいい感じに…値の取り出しと例外処理をまとめて書くことはできないのでしょうか．

実はRustやHaskellだといい感じの言語機能が生えていて，手続き的に記述しつつ，正常値ならその値が，異常値ならその内容で即時returnする，といったコードが簡潔に記述できます．

```rust:Rustの場合
fn f(x: i32, y: i32) -> Result<f64, String> {
  let v0 = safe_div(x, y)?;
  let v1 = safe_div(x, 3)?;
  return Ok((v0 + v1).into());
}
```

```hs:Haskellの場合
f :: Int -> Int -> Either String Double
f x y = do
  v0 <- safeDiv x y
  v1 <- safeDiv x 3
  Right (fromIntegral(v0 + v1)::Double)
```

いい感じですね．こんな言語機能，C++にも欲しいです．
実はC++でも同様の言語機能である[エラー伝播演算子(`.try?`)](https://wg21.link/p2561)がC++26で導入予定]です．
導入された暁には `f()` は以下のように記述できます:

```cpp:C++26におけるf()
std::expected<double, std::string> f(int x, int y){
  const auto v0 = safe_div(x, y).try?;
  const auto v1 = safe_div(x, 3).try?;
  return v0 + v1;
}
```

これに関しては[昨年のAdCでlweisacidさんが執筆してくださった](https://zenn.dev/acd1034/articles/221118-monadic-operation-for-optional#%E4%BB%96%E3%81%AE%E8%A8%80%E8%AA%9E%E3%81%AE%E5%A0%B4%E5%90%88)ので，より詳しい説明はそちらをご覧ください．

# Try to make a `try` !

$\overset{\scriptsize \text{これ以上}}{\small \text{C++26まで}}$待てるか！！！！！！！！！！！こっちが[何年 `expected` を待った](https://qiita.com/wx257osn2/items/32adec3126b03ede3034)と思っている！！！！！！！！！！！！！！！！！！！！！！！！！！！

ということで，本記事ではC++23時点で使えるエラー伝p…長いので以後歴史的な呼び名であるtry operatorと呼びますが，のようなものをあの手この手で自作してみようと思います．
ここで， `std::expected` の実装に手を加えるわけにはいきませんので，簡単に以下のような実装によってある程度自由に変更の効く `expected` を定義します:

```cpp
namespace adc2023{

template<typename T, typename E>
struct expected<T, E> : std::expected<T, E>{
  using std::expected<T, E>::expected;
};

}
```

## `TRY` マクロ

エラー伝播演算子のペーパーにも記載のある，CPPマクロによる実装．標準範囲内なら，例えば以下のようなものになります:

```cpp:TRYマクロ
#define TRY(name, expected) \
  if(not expected) \
    return std::unexpected{expected.error()}; \
  auto name = *expected;
```

```cpp:TRYマクロを使う
adc2023::expected<double, std::string> f(int x, int y){
  const auto v0_ = safe_div(x, y);
  TRY(v0, v0_)
  const auto v1_ = safe_div(x, 3);
  TRY(v1, v1_)
  return v0 + v1;
}
```

うーん…なんか野暮ったいですね．そもそも式でない．
とはいえこれは(標準の範囲では)致し方ないことで，やりたいことが「 `if` + `return` **文**または**式**」なのでそもそもC++の構文から見ると歪なマクロにならざるを得ません．
また，TRYマクロによって定義される変数を `const` にできない，第二引数を複数回評価してしまうので実用上は上記のように一度変数に束縛しないといけない，など実用性はいまいちです．

---

実際にペーパーに載っているのは上記の問題のいくらかを解消したものになります．ただしGNU拡張(statement expression)を用いているので標準の範囲からは逸脱しますが…

```cpp:GNU拡張利用版TRYマクロ
#define TRY(...) ({\
  const auto e = __VA_ARGS__; \
  if(not e) \
    return std::unexpected{e.error()}; \
  *e; \
})
```

```cpp:GNU拡張利用版TRYマクロを使う
adc2023::expected<double, std::string> f(int x, int y){
  const int v0 = TRY(safe_div(x, y));
  const int v1 = TRY(safe_div(x, 3));
  return v0 + v1;
}
```

これはまぁだいぶマシという感じはします． `-pedantic-errors` が付けられなくなることには目を瞑るとしましょう．

ただ強いて言うなら，**演算子のように前置か後置で済ませたい**ですよね．
その点で `TRY` マクロは関数形式マクロである以上どうしても式の前後を `()` で括らざるを得ません．
これを解消するためには，マクロ以外の方法で解決する必要があります．
マクロ以外で対処する場合，上述の通り「$\overset{\scriptsize \text{if + return文}}{\small \text{条件次第で脱出}}$または式」を言語機能を用いて単一の式内でよしなに表現する必要があります．

## `do_` 関数

繰り返しますが，マクロを用いないのであれば文と式を一体化させるような類のことはできませんから，式の中で条件に応じて脱出するような操作が必要です．
しかし，果たしてそのような機能があったでしょうか…

```cpp:再掲
  try{
    const std::expected<double, std::string> y = f(42, 0);
    if(not y)
      std::cout << "actually y has: " << y.error() << std::endl;
    std::cout << y.value() << std::endl;  // value()メンバ関数は異常値の場合例外を送出
  }catch(std::bad_expected_access<std::string>& e){  // std::bad_expected_access<E>例外が送られる
    std::cout << "caught: " << e.error() << std::endl;  // bad_expected_accessもまたerror()メンバ関数でその値を読める
  }
```

ありました．例外を使えば式から関数外に脱出できます．
というわけで， `try` - `catch` をいい感じにラップした `do_` 関数を作ります．

```cpp:do_関数
namespace detail{

template<typename>
struct is_expected : std::false_type{};
template<typename T, typename E>
struct is_expected<adc2023::expected<T, E>> : std::true_type{};

}

template<typename E, typename F>
requires (!detail::is_expected<std::remove_cvref_t<std::invoke_result_t<F>>>::value)
adc2013::expected<std::invoke_result_t<F>, E> do_(F&& f)try{
  return f();
}catch(std::bad_expected_access<E>& e){
  return std::unexpected{e.error()};
}

template<typename F>
requires detail::is_expected<std::remove_cvref_t<std::invoke_result_t<F>>>::value
std::invoke_result_t<F> do_(F&& f)try{
  return f();
}catch(std::bad_expected_access<typename std::remove_cvref_t<std::invoke_result_t<F>>::error_type>& e){
  return std::unexpected{e.error()};
}
```

あとは， `do_` 関数内で$\overset{\scriptsize \text{失敗したら例外を投げる操作}}{\small \text{value()メンバ関数}}$で値にアクセスするだけです．

```cpp:do_関数を使う
adc2023::expected<double, std::string> f(int x, int y){
  return do_<std::string>([&]{
    const int v0 = safe_div(x, y).value();
    const int v1 = safe_div(x, 3).value();
    return v0 + v1;
  });
}

// あるいは

auto f(int x, int y){
  return do_([&]->adc2023::expected<double, std::string>{
    const int v0 = safe_div(x, y).value();
    const int v1 = safe_div(x, 3).value();
    return v0 + v1;
  });
}
```

:::details おまけ: operator+のオーバーロード

あるいは，よりDSLらしくするなら `operator*` と見た目が似ている `operator+` をオーバーロードして:

```cpp:operator+をオーバーロードする
namespace adc2023{

template<typename T, typename E>
struct expected<T, E> : std::expected<T, E>{
  using std::expected<T, E>::expected;
  decltype(auto) operator+(this auto&& self){
    return std::forward_like<decltype(self)>(self.value());
  }
};

}

// 中略

auto f(int x, int y){
  return do_([&]->adc2023::expected<double, std::string>{
    const int v0 = +safe_div(x, y);  // operator+でエラー伝送+デリファレンスを実現
    const int v1 = +safe_div(x, 3);
    return v0 + v1;
  });
}
```

などとすることも可能ではあります(というのをやったのが[7年前の記事](https://qiita.com/wx257osn2/items/32adec3126b03ede3034#%E3%81%9D%E3%81%AE%E5%89%8D%E3%81%AB)でした)．
尤も，operator overloadまでやるとかなり侵襲的な(少なくとも本記事のように `std` 名前空間以外の箇所に `expected` を自前で置く必要があり，それをユーザーに使用させる必要がある)ので，C++23以降の標準入りした現代ではあまり現実的ではないです．

:::

`do_` 関数内では気軽に `value()` メンバ関数によるデリファレンスが可能な反面，内部機構は例外送出ですから，(現実的に多くの処理系において)実行時性能に懸念が残ります．
処理を中断したり値を返したりするいい感じの構文，無いのでしょうか？

## 前置 `TRY` 演算子

実行時性能は例外ほど悪くなく，処理を中断したり値を返したりする構文，実は近年導入されました．
名を `co_await` 式と言います．そう，コルーチンです．
実はコルーチンを用いた試みは過去にC++ AdCで行われています．
[3年前のgnaggnoyilさんの記事](https://qiita.com/gnaggnoyil/items/6d2c3cf907da0f3c8330)です．
この記事の内容をベースに， `std::expected` 向けにC++23でpromiseを書いてみましょう:

```cpp:expected向けcoroutine
namespace adc2023{

template<typename T, typename E>
struct expected : std::expected<T, E>{
  using std::expected<T, E>::expected;
};

template<typename>
class promise;

template<typename T, typename E>
class promise<adc2023::expected<T, E>>{
  using expected_type = adc2023::expected<T, E>;
  expected_type move()&&{
    auto result = std::move(this->result.value());
    if(this->handle)
      std::exchange(this->handle, nullptr).destroy();
    return result;
  }
  class return_proxy{
    promise& p;
  public:
    explicit constexpr return_proxy(promise& p)noexcept : p{p}{}
    constexpr operator expected_type()&&{
      return std::move(p).move();
    }
  };
  std::coroutine_handle<promise> handle;
  std::optional<expected_type> result;
public:
  constexpr promise()noexcept : handle{}, result{std::nullopt}{}
  constexpr promise(promise&& other) : handle{std::exchange(other.handle, nullptr)}, result{std::exchange(other.result, std::nullopt)}{}
  promise(const promise&) = delete;
  promise& operator=(const promise&) = delete;
  return_proxy get_return_object()&{
    this->handle = std::coroutine_handle<promise>::from_promise(*this);
    return return_proxy{*this};
  }
  constexpr std::suspend_never initial_suspend()const noexcept{return {};}
  constexpr std::suspend_always final_suspend()const noexcept{return {};}
  template<typename... Args>
  requires std::constructible_from<T, Args...>
  constexpr void return_value(Args&&... args){
    this->result.emplace(std::in_place, std::forward<Args>(args)...);
  }
  constexpr void return_value(std::unexpected<E>&& errs){
    this->result.emplace(std::move(errs));
  }
  template<typename U>
  constexpr auto await_transform(adc2023::expected<U, E> e){
    struct awaiter{
      adc2023::expected<U, E> expected;
      std::optional<expected_type>& result;
      constexpr bool await_ready()const noexcept{
        return this->expected.has_value();
      }
      constexpr void await_suspend(std::coroutine_handle<>){
        this->result.emplace(std::unexpect, std::move(this->expected.error()));
      }
      constexpr U await_resume()const noexcept{
        return this->expected.value();
      }
    };
    return awaiter{std::move(e), this->result};
  }
  void await_transform(auto&&) = delete;
  void unhandled_exception()const{
    auto ep = std::current_exception();
    if(ep)
      std::rethrow_exception(ep);
  }
};

}

template<typename T, typename E, typename... Args>
struct std::coroutine_traits<adc2023::expected<T, E>, Args...>{
  using promise_type = adc2023::promise<adc2023::expected<T, E>>;
};
```

これを用いることで，以下のように記述できます．

```cpp:expected向けcoroutineを使う
adc2023::expected<double, std::string> f(int x, int y){
  const int v0 = co_await safe_div(x, y);  // co_awaitでエラー伝送+デリファレンスを実現
  const int v1 = co_await safe_div(x, 3);
  co_return v0 + v1;  // fはコルーチンなのでco_returnを使う必要があることに注意
}
```

最後に以下のマクロによって前置 `TRY` 演算子の完成です！

```cpp:TRY演算子
#define TRY co_await
adc2023::expected<double, std::string> f(int x, int y){
  const int v0 = TRY safe_div(x, y);  // TRY!
  const int v1 = TRY safe_div(x, 3);
  co_return v0 + v1;  // fはコルーチンなのでco_returnを使う必要があることに注意
}
```

:::message
節のタイトルに合わせて `TRY` 演算子まで持っていきましたが， `TRY` のような短い識別子にマクロを割り当てるのは衝突のおそれが大きいし，突然の `co_return` に困惑必死なので当然実用上は `co_await` のまま運用すべきです．
:::

現行の多くの処理系においてコルーチンは若干のオーバーヘッドが乗るものの繰り返し呼び出す分にはそれなりの速度で動作しますので，速度をギリギリまで詰めなくて良い環境では悪くはなさそうです．
一方で `co_await` 演算子の優先順位はだいぶ下なので，例えば `expected<std::vector<int>, std::string>` に対してその中身の `size()` メンバ関数を呼びたい，みたいな話を(一度名前束縛せずに)単一の式でやろうとすると結局 `(co_await e).size()` のように式の前後を `()` で囲まなければなりません．
エラー伝播演算子であれば後置かつ優先順位も高めなので， `e.try?.size()` のように記述できます．
こうしたあたりはやはりまともな言語機能に軍配が上がりますね…

## 性能を比較してみる

[Quick C++ Benchmarkを用いて比較してみました](https://quick-bench.com/q/pzRjZDV-KW45oZmnrCHl5q3TWSg)．
GCCを使うとまたいろいろと変わるのですが， ~~面倒くさいので~~ 簡単のために今回はClangのみを用いて結果を比較します．

![実行時間チャート Clang](/images/try_to_make_a_try-saiudhleghbaowlb/pzRjZDV-KW45oZmnrCHl5q3TWSg.png)

案の定と言いますか，例外を用いた実装では失敗時のペナルティが凄まじいことになっています．
`bench_do_fail` を除いたチャートが以下です．

![実行時間チャート Clang 2](/images/try_to_make_a_try-saiudhleghbaowlb/z5BqePDWwIuCsc7flzvsfMiJGwE.png)

コルーチンの失敗時のコストがやや大きいようです．書き味と比べてこのペナルティを受け入れられるかは要求されるパフォーマンス次第でしょうか．

# まとめ

[あなたと犬小屋，今すぐ動作確認](https://wandbox.org/permlink/QoiJAyocvuxbv8Xe)

というわけであの手この手でtry operatorのようなものを作ってみました．
標準規格準拠・記述の簡潔さ・実行時性能を全て満たすためには，やはりまともな言語機能として生えてきてもらうしかなさそうです．
当座は本記事のような何かで凌ぎつつ，みんなでエラー伝播演算子を待ちましょう…

---

21日分は[yumetodo(`@yumetodo@misskey.dev`)](https://misskey.dev/@yumetodo)さんの「[C++erですがCOMに翻弄されています: 再入との戦い](https://qiita.com/yumetodo/items/f961bc96594b9b5619c2)」です．
