---
title: "C++ Advent Calendar 2024 感想文"
emoji: "🎄"
type: "tech"
topics: ["cpp"]
published: true
---

2024年，今年も年の瀬ですが皆様いかがお過ごしでしょうか．
年末の風物詩といえばAdvent Calendarですが，今年も[C++ Advent Calendar 2024](https://qiita.com/advent-calendar/2024/cxx)にはたくさんの方にご参加いただきました．
皆様お疲れ様でした！
あとは私が今年の記事を全部読んで感想を書けばおしまいですね．

なにかデジャブっぽい感覚とか既にとてつもなくやりきった感とかありますが^[[なんででしょうね．不思議ですね](https://zenn.dev/wx257osn2/articles/read_cxx_adc_2023_all)]，やっていきましょう．

# C++ Advent Calendar 2024 感想文

## Day1 「[std::functionと愉快な仲間たち ～move_only_function, copyable_function, function_ref～](https://qiita.com/yohhoy/items/0d2cec9bebe3f5325009)」

1日目，function系クラスの紹介．
`std::function` は便利だけどちょっと重たかったり問題を抱えていたりしていて，そこをより軽い使用感にしたり用途別に分けたりすることで問題解決を図っているのが `std::move_only_function` `std::copyable_function` `std::function_ref` の3つになります．
適切な実装が生えるのは歓迎な一方で，既存の `std::function` で関数をやり取りしていたインターフェースは(すぐに使えなくなるわけではないとはいえ，より適切なクラスを使うべき，と考えると)移行が大変そうです．
私はと言うと，オーバーヘッドを嫌って `std::function` をあまり使わないので歓迎ムードです．

そういえばこの手のtype erasureなクラスの実装ってなんとなくはわかるけど自力で書いたことなどはないので，今度書いてみたいですね^[何故かは知らないのですが「今は時間無いからやめろ」ってカンペが…]．

## Day2 「[C++のlinalgを使ってみたい(まだ使えない)](https://168iroha.net/blog/article/202412020035/)」

2日目， `<linalg>` の部分実装を通して理解を深める記事．
記事内でも言及されている通り，BLAS使うような層は大体BLASを直に叩けてしまうので使われるのかはだいぶ疑問なのだが^[この辺は `std::simd` にも思うところがあります]，

> iccで `<linalg>` が実装されて、デフォルトでMKLが呼び出され、引数で指定する実行ポリシーでシーケンシャルなBLASかパラレルなBLASを呼び出すかを指定できるようになると使われるようになるのかもしれない

たしかにいたくまともな実装が生えると使われる可能性は無くもない．
いやでもiccではまともな `<valarray>` くんは流行らなかったのだよな…

結局きれいにラップするのはしんどいよね，という話で，まぁそうだよなぁ…という気持ちに．

## Day3 「[拡張浮動小数点数型の変換ランクに関する規定のある一文について](https://onihusube.hatenablog.com/entry/2024/12/03/031917)」

3日目，拡張浮動小数点数の変換ランクに関する話．
拡張浮動小数点数というのは `<stdfloat>` をincludeすると `std::float32_t` とか `std::float64_t` とかが使えるようになるやつです．
また， `float` や `double` はその内部表現について処理系に依存しますが， `std::float32_t` や `std::float64_t` は(定義されていれば)必ずIEEE 754に準拠した内部表現となることが規定されています^[したがって， `std::float32_t` や `std::float64_t` を積極的に使うことで「IEEE 754に準拠してない環境はサポートしないぜ！」という意志を表明できる]．

で，普通の浮動小数点数型と拡張浮動小数点数型の相互変換に用いられる概念が変換ランクで，「 `double` 以上」以上のことが規定されてない `long double` が `double` と同じbit数でも問題なく変換されるようにアドホックな条項が入ってますね，という話でした．

## Day4 「[Niebloid の導入と廃止](https://qiita.com/yaito3014/items/d33e35b49cded4aa3fa8)」

4日目，C++26からNiebloidは消えますの話．
そもそもniebloidって関数オブジェクトじゃなかったんだ…(無知)^[ADLを避ける用途で作られた関数オブジェクトのことをniebloidと呼びます，のような認識だった]

## Day5 「[キャプチャを含むラムダ式を関数ポインタ化](https://qiita.com/Raclamusi/items/8f3fbc68dbb9ae6c14ef)」

5日目，キャプチャありのラムダ式でも静的記憶域期間のストレージにぶち込めば他のラムダ式からキャプチャせずに呼べるようになるので，そのようにして関数ポインタを取り出すことでキャプチャありラムダ式の関数ポインタを間接的に作れるねの話．
[Twitter上でも言及があった](https://x.com/raclamusi/status/1864513414040784975)通りこれはネタ記事なのですが，巧妙にコピーキャプチャしかしてないの笑ってしまう(参照キャプチャはかなり容易に壊れるため)．

> static 変数などはキャプチャせずともラムダ式から使えると cppreference に書いてます

> 規格書からこれに関する記述は見つけられませんでした。どこに書いてあるんだ？

直接的な記述は

- ラムダ式の `{}` 内は `operator()` の定義として扱われる([`[expr.prim.lambda.closure]/13`](https://timsong-cpp.github.io/cppwp/n4950/expr.prim.lambda.closure#13))
    - 関数スコープ内で定義されたラムダ式はlocal classのメンバ関数となるわけですね
- local class内の関数はその親の関数のstatic変数にアクセスできる([`[class.local]/1`](https://timsong-cpp.github.io/cppwp/n4950/class.local#example-1))

の2つ，理解を深める記述としては

- ラムダ式のbodyはキャプチャをすることでラムダ式を囲うスコープのlocal entityにもアクセス可能([`[expr.prim.lambda.capture]/1`](https://timsong-cpp.github.io/cppwp/n4950/expr.prim.lambda.capture#1))
    - コピー/参照キャプチャはlocal entityしか扱えない([`[expr.prim.lambda.capture]/4`](https://timsong-cpp.github.io/cppwp/n4868/expr.prim.lambda.capture#4))

が挙げられます．つまり本来関数外のlocal entityにアクセスできる事自体がおかしく，それを実現しているのがキャプチャ．それ以外の点は普通のlocal classの `operator()` と変わらないので，キャプチャしなくてもアクセスできるわけですね．

ところでlocal entityというのは何かというと，自動記憶域期間変数やそうした変数に対する構造化束縛，または `*this` を総称してそう呼びます([`[basic.pre]/7`](https://timsong-cpp.github.io/cppwp/n4950/basic.pre#7))．
そう，構造化束縛がキャプチャできるのですよね．
これは[C++17からC++20の間で紆余曲折あった](https://x.com/wx257osn2/status/1569694461093105666)のですが…あまりにも記事の本題から外れるのでonihusubeさんの「[構造化束縛の動作モデルとラムダキャプチャ](https://onihusube.hatenablog.com/entry/2019/10/04/122001)」辺りを各位で読んでください^[[当初はこの話題でもうちょっといろいろ書く予定でした](https://x.com/wx257osn2/status/1864567303318458531)が，巻きを催促されているため…]．

## Day6 「[型情報を渡したいときの小技](https://zenn.dev/toru3/articles/90ce75c21fc970)」

6日目， `std::type_identity` を使うことで関数に型情報のみを値として渡せる話．
当日一緒にバーで飲んでいて，話題として「AdCの季節ですよ，そういや今日のAdC記事はどうなったかな…」などとカレンダーページを開いたところ，そこには著者が蒸発した枠が…！
ということで急遽頼み込んで書いてもらった記事になります(ご参加ありがとうございました！)．

`std::type_identity` は古くC++11の時代には `type_wrapper` などの名前で各ライブラリに用意されていたものですが，規格に導入された際は「型推論の無効化」を目的としたものとされました:

```cpp
template<typename T>
T f(T a, T b);

template<typename T>
T g(T a, std::type_identity_t<T> b);

f(1, 3.f);  // Tがintかfloatかわからないのでambiguousと言われる
g(1, 3.f);  // 推論は第1引数のみを用いて行われるのでTはint
```

しかし，古くから使われてきたように `std::type_identity` には他にも多数の用途があります．
その1つが記事内で使われたような関数への値の形での型情報の伝播です．
単に型情報を渡すだけなら(生成コストを度外視すれば)一見その型のオブジェクトを渡すなどすれば良さそうにも思えますが， `std::type_identity` を噛ませることでdecayingを阻害することができ，これによってcv-ref qualifierや配列型を保持することができます:

```cpp
template<typename T>
T f(T a);

template<typename T>
using ident = decltype(f(std::declval<T>()));

static_assert(std::is_same_v<ident<const int>, int>);
static_assert(std::is_same_v<ident<int[4]>, int*>);
static_assert(std::is_same_v<ident<std::type_identity<const int>>::type, const int>);
static_assert(std::is_same_v<ident<std::type_identity<int[4]>>::type, int[4]>);
```

他にも[`std::conditional` で条件によって `::type` メンバ型が定義されない型に対するフォールバックを提供したい時，thenとelseの時点で型が定義されていないとコンパイルが通りませんが，フォールバック側を `std::type_identity` でラップしてやって分岐後に `::type` を取り出すことでうまく動かす，といったテクもあります](https://x.com/wx257osn2/status/1539647083153883136)．

## Day7 「[cmath いろいろconstexprになりました](https://qiita.com/tyanmahou/items/1830aa3e41ebc40242b4)」

7日目，C++23からC++26でだいぶ `constexpr` 対応が進んだ `<cmath>` の話．

> [当初はC++0xぐらいでlibstdc++は対応してたのに規格側で「処理系によってコンパイルが通ったり通らなかったりするのは良くないので標準で `constexpr` 付けてない関数を勝手に `constexpr` にするのは禁止！」という提案が通ってしまい，処理系側が後から `constexpr` を剥がしたという歴史があり，15年経ってようやく公式に `constexpr` になったのかと感慨深い気持ちです．もしかすると(先述の通り)これはC++20から `constexpr` 関数内でinline asmが書けることで実行時性能を落とすことなくコンパイル時計算ができるようになったことが一因かもしれません．](https://zenn.dev/wx257osn2/articles/read_cxx_adc_2023_all#day16-%E3%80%8C2023%E5%B9%B4%E3%81%AE%E3%82%B3%E3%83%B3%E3%83%91%E3%82%A4%E3%83%AB%E6%99%82%E3%83%AC%E3%82%A4%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%B3%E3%82%B0%E3%80%8D)

## Day8 「[C++ で [[lifetimebound]] 属性を用いてダングリング参照の発生リスクを軽減する](https://zenn.dev/reputeless/articles/cpp-lifetimebound)」

8日目，各コンパイラのダングリングリファレンスに対するdiagnosticsの取り組みの紹介．
個人的にはなんもしなくてもちゃんとdiagnostics出してくれるGCCが好みなのですが，ClangやMSVCは `[[clang::lifetimebound]]` や `[[msvc::lifetimebound]]` を導入して検査対象を絞る方針のようです．
まぁ処理系の知らないattributeは無視されるだけなので，とりあえず `[[clang::lifetimebound, msvc::lifetimebound]]` を付けて回れば良さそうに思います．

GCC「 `warning: 'clang::lifetimebound' scoped attribute directive ignored [-Wattributes]` 」

うるさいッ！^[「処理系の知らないattributeは無視される」，まぁコンパイル止めたりはしないんですがGCCやMSVCは知らないattributeがあると(typoなどの可能性もあるので)warningを出すんですよね…ライブラリについては現実的には記事内のように処理系によってattributeを切り替えるのが良さそうに思います．ユーザーコードに関しては `-Wno-attributes` とか `#pragma warning(disable: 5030)` でも付けてやれば上記のをまとめて使ってしまっても良さそう]

## Day9 「[C++を使ったlinuxのコンソールアプリ(CLR)を作る方法](https://qiita.com/takoyaki-death/items/9c8aaa69992e7b114433)」

9日目，Hello, worldのちょっと延長．
C++23以降では `import std;` も `std::print` も生文字列リテラルもあるので以下で良いはずです:

```cpp
import std;

int main() {
  using namespace std::literals::string_view_literals;
  const auto aa = R"(
 /\_/\
( o.o )
 > ^ <
)"sv.substr(1);
  std::print("{}", aa);
}
```

ところで記事に記載のとおりに環境構築をやると `import std;` が使えるコンパイラは降ってこないので `#include <print>` と `#include <string_view>` に置き換えないといけないし， `#include <print>` が使えるのはGCC14以降なのでUbuntuでやるなら24.10必須ですが…

そういえば生でない文字列リテラルでAA書くと `\` が `\\` になるので(そして大量に `/` と `\\` と `_` と `|` が混ざっているような絵だとそういうことを忘れがちなので)，パッと見違和感がすごいのですよね．

## Day10 「[ドメイン仕様に基づいてコンテナクラスを集合論的自作クラスでラップすると幸せになれた話](qiita.com/Hourier/items/d5e4c74983410f5578af)」

10日目，野蛮なCコードをC++でリファクタリングした話…でいいのか…？
たぶん言いたいこととしては「STLコンテナを裸で持つと治安の悪いコントリビューターが破壊してくるので必要な機能だけをインターフェースに出してコンテナ本体はprivateなところに置いといたほうが安全」のようなことに聞こえる．解釈があってるかわからんが…

自分ならどうするだろう，と思うと，たぶん `resize` とかをインターフェースとして出さない気がします．
全部immutableにして初期化時に完全に生成し切る．
まぁその辺りの仕草を阻害しそうな「未鑑定名シャッフル処理」なども見えるのでどの程度できそうかはわからないですが，ともかくやるなら徹底的にやらないと結局壊されうるので…

コードについていくつか気になった点としては，

- 積極的に `std::map` を使うほど `BaseitemKey` や `ItemKindType` の数は多いのか？
    - C++20でもとりあえず `std::unordered_map` を初手で検討しない？の気持ち
- 治安の悪いコーディング環境を想定するなら^[何故そのようなことを言い出すのかは17日目の感想を参照] `create_baseitems_cache` はメンバ関数にしないほうが良さそう
    - 何回も呼ばれるとキャッシュが壊れるため
        - やや無理気味だが塞ぐならキャッシュ作る関数の時点で `static const auto cache = []{ここで作る}();` とかか？
    - そもそもこの辺の関数メンバ関数にする必要ある？ `baseitem-info.cpp` の内部リンケージなグローバル関数にしたほうが外から隠蔽できて良さそうに見える(まぁ一応privateではあるので余程のことをしなければ外からは呼ばれないけど)
- コピーが多めに見える
    - まぁパフォーマンスクリティカルでないなら多少はええかの気持ち
    - 連続している必要があるのでどの程度使えるかはわからないが， `std::span` や `std::subrange` などをうまく使える場所では使っていくと部分集合抽出を低コストに行える

あたりで，大筋納得感ある話に見える．
たぶん記事の内容を本当に理解するには元になった世紀末Cコードを読む必要がありそうだなぁ…

## Day11 「[std::cout << "a very short history of iostream";](https://qiita.com/yohhoy/items/48d14422ba0b5fd03e76)」

11日目， `<iostream>` の `operator<<` は当時としては妥当性のある選択肢だったよ，をBjarne Stroustrupの文献を引用して解説する記事．
なんでか知らないんですけど2024年🤬「文字列表示に `<<` って何だよこれ！」なツイートはまだよくて，「 `std::ostream` で `operator<<` 使ってるの規格作ってるやつがoperator overload楽しくて悦に浸ってただけでしょｗ」みたいなツイートまで無限に出回ってて，前者はたまに[ざっくりとした回答をしたり](https://x.com/wx257osn2/status/1742856716948242902)，後者は見かける度にクソデカため息だったんですが^[どうせまともな代替案も提示できなければ策定に至った経緯も知らんくせに適当なことを言うな]，今後はこのURLを投げつけるだけで良いので大変助かる．特に私は当時の経緯について推察はできるけどちゃんと出典を出すことができていなかったので…

これは何度も言っているのですが， `<iostream>` は本当にろくでもない仕様で，

- フォーマットがしんどい
    - フォーマットを状態として持つので適切にリセットしないとそれ以降の出力が壊れる
    - フォーマットが(`std::printf` 比で)長ったらしい
- ロケールサポートがデフォルトで入ってるので遅い
- 恐怖の菱形継承
- メンバ関数の名前が謎に短縮されていて読めない

などいくらでも叩く余地があり，そんな中でかなり妥当性のある技術選択である `operator<<` / `operator>>` のオーバーロードを取り上げて(しかも「わかりにくい」などではなく「自己満乙ｗ」みたいな中傷じみた)批判をする人の多いこと多いこと…批判するならちゃんと学んでからにして欲しい^[これは自戒でもある]．

## Day12 「[MISRA C++(106), AUTOSAR C++ and so on](https://qiita.com/kaizen_nagoya/items/d9c69d79d5ad4aa73a91)」

12日目，自動車業界でのデファクトスタンダードなコーディング規約MISRA C++について．
2023年にもなってC++17のコーディング規約を出したことを誇らしげにしているの，業界の外から見るとえぇ…みたいな気持ちになるかもしれませんが，そもそも $\overset{\scriptsize \text{事故ったら人死にが出る}}{\small \text{そういう}}$ 業界なので…
もう少し真面目に経緯を説明すると，基本的に業界の出してるコーディング規約に従ってないと出荷できない上に，1個前に出たMISRA C++は2008年のなのでこれまでは(MISRAに従うなら)C++11すら使えなかったという惨状でした．
流石に耐え難かったのかAUTOSARというMISRAとは別の団体がC++14の規約を2017年に出して，ここ7年くらいは業界全体でC++14までは使えてうれし…いやまだ14しか使えんの？なんもできんくない？みたいな雰囲気^[14でゴネてるのは業界の中では一部かもしれん．そもそもC++をまともに使える層が業界の中で一部っぽいので…]でした．
MISRAも危機感を抱いたのか真面目にやる気になって，ついにMISRA C++ 2023が出た，という流れ．
現場からは「これで `if constexpr` が使えるのでSFINAEや特殊化からいくらか解放されます」「 `std::optional` 使って良いんですか！やったー！」「処理系^[AUTOSAR C++準拠変なチップ向けC++コンパイラ，みたいなのがある]がまだ対応してないじゃないすか！やだー！」などの声が聞こえてきております．
大変そうですね(他人事)．

## Day13 「[std::function<bool()> を使った簡易な疑似並行処理](https://zenn.dev/tenka/articles/tiny_act_flow)」

13日目，ゲームループなシングルスレッドプログラムにおける並行処理．
ゲームループというのは一般にGUIの描画やイベントハンドリングなどを司っているので，止まるとGUIが死にます．
それ故ファイル読み込みのような時間のかかる処理^[マルチスレッドで対処することも可能ですがどっちみち `std::promise` からの値の取り出しを待機する処理はループ内で行う必要があります]やアニメーションのような時間方向に一定時間処理が必要なものなどは時間方向に分割して継続して処理をする必要があり，タスクを順列・並行に実行できると便利という話です．
この手のにはコルーチンが最適で，C++23では以下のように記述できます^[ところで `g++` 14.1で `-O3` するとSEGVするんだけど誰か原因わかったら教えて下さい…コルーチンなんもわからん]:

```cpp
#include<generator>
#include<vector>
#include<algorithm>
#include<utility>
#include<functional>

namespace TinyActFlow {

using Act = std::generator<bool>;

template<typename... Acts>
inline Act seq(Acts... acts) {
  for(auto&& xs : {std::ref(acts)...})
    for(bool x : xs.get())
      if(x)
        co_yield true;
      else
        break;
  co_yield false;
}

template<typename... Acts>
inline Act para(Acts... acts) {
  std::vector<decltype(std::declval<Act>().begin())> its;
  its.reserve(sizeof...(acts));
  for(auto&& xs : {std::ref(acts)...})
    its.emplace_back(xs.get().begin());
  while(std::ranges::any_of(its, [](auto&& it){return it != std::default_sentinel;})){
    bool running = false;
    for(auto&& it : its){
      if(it == std::default_sentinel)
        continue;
      running = running || *it;
      ++it;
    }
    if(running)
      co_yield true;
  }
  co_yield false;
}

}

#include<print>
#include<thread>
#include<chrono>
#include<cstdio>

int main(){
  using TinyActFlow::Act;
  using TinyActFlow::seq;
  using TinyActFlow::para;

  bool end_req = false;
  auto endReq = [&end_req] -> Act{
    end_req = true;
    co_yield false;
  };

  const char* str = "hello world!";

  Act mainAct = para(
    [] -> Act{
      for(int i = 0; i < 100; ++i)
        std::println("");
      std::fflush(stdout);
      co_yield false;
    }(),
    [] -> Act{
      while(true){
        std::print("{1:{0}c}\r", 40, ' ');
        std::fflush(stdout);
        co_yield true;
      }
    }(),
    [] -> Act{
      for(auto step = 0u;; ++step){
        std::print("[{:2d}]  ", step);
        co_yield true;
      }
    }(),
    seq(
      [str] -> Act{
        auto step = 0u;
        while(str[step]){
          char buf[] = "            ";
          buf[step] = str[step];
          ++step;
          std::print("{}", buf);
          std::fflush(stdout);
          co_yield true;
        }
        co_yield false;
      }(),
      [str] -> Act{
        auto step = 0u;
        while(str[step]){
          std::print("{2:{0}.{1}s}", step, step, str);
          std::fflush(stdout);
          ++step;
          co_yield true;
        }
        co_yield false;
      }(),
      [str] -> Act{
        for(auto step = 0u;; ++step) {
          std::print("{}", str);
          std::fflush(stdout);
          co_yield step < 16;
        }
      }(),
      endReq()
    )
  );

  for([[maybe_unused]] auto&& _ : mainAct){
    if(end_req)
      break;
    using namespace std::literals::chrono_literals;
    std::this_thread::sleep_for(100ms);
  }
}
```

この手の処理は今回のサンプルには含まれていませんが，手続き的な処理を書くと記事内にあるように `step` に対して `switch` 文で実行コンテキストを保持する必要が出てきて非常に見にくくなりがち．
コルーチンを使えば `co_yield` を書くだけで良いのでスッキリします．
とはいえコルーチンが安定して使える環境もまだ少ないでしょうから^[C++20でも `std::generator` ぐらいは簡単に自作できるのでC++23を待つ必要は無いです，今すぐお手元の環境に導入しましょう]，当面は記事にあるような実装を使うことになりそうな気もします．

> 汎用的に書いたつもりだったのですが、各端末の画面更新頻度を考慮不足で、動きにならない……

これはバッファリングが原因なので，上記のコルーチン版のように各 `printf` の直後に `std::fflush(stdout);` でも書いておけば良さそうです．

また重箱の隅つつきですが， `vec_add` は

```cpp
    template<class VEC, typename T, typename ...Args>
    void vec_add(VEC& vec, T&& t, Args&&... args) {
        vec.emplace_back(std::forward<T>(t));
        vec_add(vec, std::forward<Args>(args)...);
    }
```

の方が良い気がしますね(引数がlvalue-refならちゃんとコピーコンストラクタが走る，argsがperfect forwardされるので不要なコピーコストが発生しない)． `Seq` や `Para` も同様．

## Day14 「[C++20での"bitwise operation between different enumeration types is deprecated"を力技でねじ伏せる](https://qiita.com/hon_no_mushi/items/072f27521c13782b1a04)」

14日目，C++20からenumに対する暗黙の算術変換が非推奨となった話．
enumに対する暗黙の算術変換は非推奨になったけど単項 `+` 演算子による汎整数拡張([`[expr.unary.op]/7`](https://timsong-cpp.github.io/cppwp/n4950/expr.unary.op#7))で `int` にしちまえば怒られねーんだ！

> The operand of the unary `+` operator shall have arithmetic, unscoped enumeration, or pointer type and the result is the value of the argument. Integral promotion is performed on integral or enumeration operands. The type of the result is the type of the promoted operand.

まぁ，ありがちですよね．他に単項 `+` 使う場面としてはキャプチャ無しラムダ式を関数ポインタに起こすときなど．

> 他enumとORedで使う場合は、cast先するべき型がそもそもない

これはたぶん偽で， `std::underlying_type_t<Enum>` にキャストすれば良さそう．双方のunderlying typeが違うとかだと微妙な顔をすることになりそうだが…(それでも一応汎整数拡張でよしなにされるはず)

コメント欄での議論も盛り上がってますが，パッと見た感じ

- C++11以降サポートならC++では普通に正攻法で良くない？
    - C++20対応というのは「それなり」な理由になりそうに思うけど．既存実装の動作を変更するものでもないし
- C APIはどうしようもないので `+` による汎整数拡張しか手が無い
    - `std::underlying_type_t` もoperator overloadも使えないので

という印象．

## Day15 「[TauriにC++を組み込んでGUIアプリケーションを作ろう](qiita.com/kizul/items/33c5b0db42a3475c7109)」

15日目，TauriとC++を組み合わせる話．
TauriというのはElectronのRust版みたいなやつで，ネイティブアプリをバックエンドRust/フロントエンドJSで書けるやつです．
せっかくRustを採用したTauriにRust-C++ interopでC++をぶち込む所業…いいですね！こういうの大好き！

cxxクレートは一時期話題になってたので存在は知ってたんですが，結局どの程度使えるものなのかはあんまりわかってない．
一応 `std::string` をRustの `String` にマップしてくれるらしい，などは聞いたことがあります．
ともかく，せっかくリポジトリも公開されていることですし是非追試したい．
なんかちょうどいい感じのGUIアプリケーションとかあるかしら…

## Day16 「[std::printについて](https://qiita.com/exli3141/items/694d3040ad91087f2887)」

16日目， `std::print` について．


> C++のはろわがどういった機能を使用して記述されているのかを説明するだけでC++の初学者が去るのに十分すぎる

> 多くのメジャーなプログラミング言語たちがそれほど苦しまずに言語機能を知る最序盤で触れる関数によってはろわを実現している

残念ながらこれはあらゆる言語に言えることで，最初に出会いがちな標準入出力が標準ライブラリの中でもだいぶ難しい方というのはありがちな話な気がします．
Cは可変長引数，Rustならマクロといった可変長な入力に対する対処の術に加えて，そもそも標準出力を扱うということがどういうことなのか真面目にやるのはどの言語でも相応に難しい．
加えて， `std::print` を完全理解するのも普通にしんどいのだよな(フォーマット文字列のコンストラクタが `consteval` なのでコンパイル時に処理される話など)．
ただまぁ， `iostream` は見た目が奇っ怪であるが故に興味は引きがちかもしれませんが．

パフォーマンスについては[コメントにもあるように `ostream` の実装に問題がある](https://qiita.com/exli3141/items/694d3040ad91087f2887#comment-75f7ea85e1e40467b0d2)こともあり，いろいろ追試したい(google-benchmarkのセットアップがめんどくさいので後回しにさせてくだち…) → しました．
[ここ](https://gist.github.com/wx257osn2/274ddc7548faab4ed1f475d3760d293e)に置いておきますが，

- `ostream` の実装を修正
- 実行環境
    - Ryzen 9 7950X3D
    - Ubuntu 24.04
    - g++ 15.2

といった感じ．その結果が以下:

```
$ ./bench --benchmark_out=out.txt --benchmark_out_format=console
(中略)
$ cat out.txt
2026-01-01T17:19:55+09:00
Running ./bench
Run on (32 X 4336.32 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x16)
  L1 Instruction 32 KiB (x16)
  L2 Unified 1024 KiB (x16)
  L3 Unified 98304 KiB (x2)
Load Average: 0.09, 0.15, 0.16
-----------------------------------------------------
Benchmark           Time             CPU   Iterations
-----------------------------------------------------
printf           1244 ns         1244 ns       544007
ostream           889 ns          622 ns      1072106
print            1234 ns         1233 ns       560454
```

意外にも，ostreamが速い結果となりました．
ほんとかよと思いましたが `sync_with_stdio(false)` を消せば遅くなるし(それでも他2つと同じぐらいのもの)，他2つに `sync_with_stdio(false)` しても速度に差がほぼ出なかったので，ほんとらしい．

## Day17 「[リテラルの型エイリアスは止めろ、マジで止めろ](https://qiita.com/Hourier/items/795b4ceb6aa798b858b5)」

17日目，リテラル型の型エイリアスに苦しめられた話．
本質的にはリテラル型の型エイリアスが悪いわけではなくて，治安の悪いコードの問題な気がします．
というのも，例えばアイテムの数を表現するのに `std::uint16_t` を直接使った場合，65536個目のアイテムが登場した瞬間にコード上のあらゆる「アイテム数を表現する `std::uint16_t` 」だけを変更しなければならないからです．
それよりは `ITEM_NUMBER` にでもなっていたほうが後から変更するのが容易で良いでしょう．
将来的に変更する可能性があるエイリアスについてはビット幅を気にするのが誤りです(変更しない場合は頑張って覚えよう)．
ただし，これは正しくエイリアスを取り回している前提であって，勝手にエイリアスを剥がして中の型を直接書いたり，同じ変数を別の用途に使い出すような人がいたらそりゃうまく回らないわけですね．
リテラル型の型エイリアスより先にコードレビューで腐ったコードを書かなくなるまで指摘し続けるのが重要な気がします．
既にマージされてるので手遅れみたいですが…
その結果として導入された対策があまりに大規模過ぎる変更で目を覆いたくなるのですが…リテラルの型エイリアスに対する対応のみならず，データがポン置きで随所から操作されまくってる問題にも同時に対応した結果でしょうかね．
~~リファクタリングするより作り直したほうが速いんじゃ…~~

もう少し穏当な変更として，同じ変数を別の用途に使えなくする，という観点ではstrong typedefなども効きそうな気がします．
まぁこれも随所の型が適切に記述されていることが前提ですが…そもそも型もまともに書かれてなかったらそんなコード壊れるに決まってるじゃんね

## Day18 「[C++ glaze JSONライブラリの紹介と条件-値フィルタリング](https://qiita.com/kc-hedgehog/items/ce54f62dddc6d0072168)」

18日目，自称最速のJSONライブラリglazeの紹介とrangeでデータをフィルタリングしていく実例．
glazeはJSONの読み書きとJSONの値を表すクラスを提供するよくあるJSONパーサーライブラリという感じではなく，Rustで言うところのserde_jsonのようにJSONを構造体にマッピングしてシリアライズ・デシリアライズするような実装みたいです．
実行時のオーバーヘッドはだいぶ減らせそうな作りしてますね．

例示されたデータはoptionalな項目が多く一見処理しにくそうにも見えますが，projectionやrangeの機能でいい感じに落とし込めています．
rangeの強力さを示すよい実例ですね．
一方この例からは，glazeに合わせて構造体を作るとオブジェクトメンバの排他の表現が難しそうなことも見て取れます．
本質的には `bonus_data` も `equipment_cond` も `id` なのか `type` なのかの `bool` と実際のデータである `std::vector<int>` で表現可能なはずですが，この辺りは自動生成故の難しさという感じですね…(たぶん `glz::meta` で自前の型に合わせてちゃんと書いてやれば行けるはず/毎回やりたいかと言われるとうーんという感じ)

## Day19 「[それはCOM STAと並列処理の三体問題との戦いだった ～Optimal Biz Teleworkの機能をOptimal Bizに部分取り込みする～](https://tech-blog.optim.co.jp/entry/2024/12/19/100000)」

19日目，COMでSTAとMTAを混ぜたらSTA側でmutex取れなくて詰みかけた話．
呼び出し順序のパターン20通りそれぞれについて動作に問題ないことを証明．お，おつらい…
これもうMTAに全面移行したほうがいいって！と思ったら計画はしてるらしい．
それはそれで大変そうだけど頑張ってほしい．

並列処理が当たり前になった現代ではやや珍しいシングルスレッド並行処理における問題への対処を綴った記事ですが，とはいえ一般に並行処理全般で気をつけた方が良いポイント^[マルチスレッドだと気にしなくても動くことがあるが，気をつけたほうが堅牢な設計になる要点，ぐらいの意味合い．後述する話も，同時に書き込み操作が1箇所しか走らない保証があるならmutableな参照をいくつ持ってても問題ないと言えはするが，だからといってmutableな参照をいくつも取り回すのは脆弱な(いつ誰が壊しても不思議がない)設計だよね，という話]がまとまってるようにも思います．
例えば書き込み操作をSTAに集約する話も，「mutableな参照は1つしか持つべきでない」というRust的な考えを持ち出すと妥当な解決方法でしょう．

## Day20 「[poacの現況 + 改造して遊んでみた](https://zenn.dev/wx257osn2/articles/current_poac-iuartb123ir2)」

20日目，私です！
ちなみに記事内にも追記をしましたが，PoacはCabinに改称しました．
消費期限3日の記事になってしまった．

## Day21 「[【闇】力技によるメンバ変数名取得](https://qiita.com/tyanmahou/items/aedeb7838a776de118f1)」

21日目，18日目で登場したglazeの深堀り．
静的リフレクションが無いのにシリアライザ・デシリアライザを既存構造体にマッピンするなんてどうやってやるんだ？というと，

- 集成体初期化できなくなるまで要素を増やしていくことで集成体のメンバ数を導出
- メンバ数がわかるので構造化束縛してメンバに対するlvalue-refを集めた `std::tuple` に変換
- tupleになったので連番で要素を取得する関数が定義可能．うまいことやるとメンバポインタをtemplate引数に載せられる
- 上記関数の `__PRETTY_FUNCTION__` から一部を切り出してくると変数名が文字列で取得可能

ということで，文字列の変数名と対応するメンバの型がわかるのであとはよしなに，という感じ．
hackyなのももちろんですが， `__PRETTY_FUNCTION__` のような処理系定義の非標準な機能に依存しているが故の力技でもあります．
このような手法ではなくより綺麗なインターフェースで使えて処理系に依存しないリフレクションを提供したいとC++標準規格が考えるのは至極当然で，それ故の静的リフレクション，待ち遠しいですね，という感じなんですが，現時点で動く黒魔術があるならそれはそれで使っていくべきなんだよな…みたいな気持ちもあり．

## Day22 「[oneTBBの使い方](https://sukeya.github.io/articles/how_to_use_onetbb/)」

22日目，Intel oneTBBの使い方について．
並列処理をラップするライブラリですね．
この記事で紹介されているデータ並列の他にも，タスク並列処理を平易に書ける機能などもあります^[私個人としてはこっちの印象がつよい．データ並列は別にOpenMPとかSIMDとかGPGPUとかでいいかなとなる]．
古くはIntel TBB(Threading Building Blocks)という名前のライブラリでしたが，oneAPIの構想に合わせてoneTBBと改称したようです．
工事中なのでそのうちもう少し記事が厚くなりそう．

PoacもといCabinでも使われているので先日少し触ったのですが， `blocked_range` の `begin()` がそのまま数値を返してくるのはあんまり許してません．range-based forで使えんやんけ！

## Day23 「[20年以上前の自作ゲームは再コンパイルしたら動くのか？](https://qiita.com/pierusan2010/items/4c46d31fd2396a59fe88)」

23日目，Win98時代のVC++5.0製ソフトウェアをVS2022でビルドするために修正していく話．
端から見てもだいぶ簡単な修正で動いたようです．すごい．

> const_castを使い文字列リテラルの型 const char* からconstをはがしてやることで対応。
> これが正しい対処かは不明ですが、単にファイルPathを渡しているだけなので問題ないはず。

まぁ動きはするのですが，より適切なのは「 `DDLoadBmp` の第2引数をちゃんと `LPCSTR` に変えてやる」でしょうね．この場合呼び出し側は変更不要です．
`T*` と `const T*` を相互に暗黙変換してしまう古のVC++仕草という感じ．

> 昔は、型を省略するとintとして扱ってくれて下記はOKでしたが、型を明示しないとコンパイルエラーになるようです。

変数の型を省略すると `int` として推定してくれるのはC89までの仕様で，C++は最初から，CもC99からは認められてなさそう．
これもCなんだかC++なんだか曖昧な古のVC++仕草という感じ．

## Day24 「[C++でヒープを使わないSTL代替ライブラリ2選](https://qiita.com/ktetsuo/items/23eba2c11635cbe35fd5)」

24日目，動的確保をしないvector代替ライブラリの紹介．
この手のだとやはりBoost.Containerの `static_vector` を真っ先に思い浮かべるのですが，最近はBoostは流行らないのでしょうか…(まぁBoost.Container使うためにリポジトリ5個引っ張ってこないといけないしなぁ)

## Day 25 「[自作の乱数生成器を乱数分布ライブラリ(std::uniform_int_distribution)と組み合わせる](https://qiita.com/youpong/items/6a5ff842d879eec363f3)」

25日目，乱数生成器を自作する話．
STLのコンテナなどと同様，乱数周りも「特定の操作に対して適切な結果が返ってくるクラス」のような形でインターフェースが切られているので，標準ライブラリのrequirementsに合わせて実装すれば自作の乱数生成器で差し替えられます．

> プログラムは、標準規格 C++20 に準拠している。そのため、新しいバージョンの C++ コンパイラーと C++ 標準ライブラリの実装が必要になる。

実際にリポジトリに置いてあるコードはC++14までの機能しか使われていないので， `Makefile` を編集すれば環境自体は古くてもコンパイルできます．

> Linear Congruential generator

まぁあくまで「自作の乱数生成器で差し替えられる」ということを示すための例示だから良いのですが…現代で線形合同法による乱数生成器を使うことは余程つよい理由がない限り避けたほうが懸命でしょう．
現代ではXorshiftの派生形であるxoroshiroなどを用いるのが，高速かつ小さい実装で良いはずです．

> 型エイリアス RandomEngine::result_type が非負で定義されていることに注意。

> LLVM のサブプロジェクトlibc++ による C++ 標準ライブラリの実装では、 std::uniform_int_distribution のオブジェクトが引き受ける乱数生成器について、非負の乱数を生成することを求めている。

これは規格でそのように規定されています．
`std::uniform_int_distribution` などはrandom number distribution requirements([`[rand.req.dist]`](https://timsong-cpp.github.io/cppwp/n4950/rand.req.dist))を満たす必要があります．
その中で，引数として受容する型はuniform random bit generator requirementsを満たすlvalueである，とあります([`[rand.req.dist]/3.6`](https://timsong-cpp.github.io/cppwp/n4950/rand.req.dist#3.6))．
次にuniform random bit generator requiementsには非負整数を返す関数オブジェクトである，と記載されています([`[rand.req.urng]/1`](https://timsong-cpp.github.io/cppwp/n4950/rand.req.urng#1))．
以上から，(何か特定の標準ライブラリ実装の要件としてではなく)一般に `std::uniform_int_distribution` に渡すような関数オブジェクトはuniform random bit generator requirementsを満たす必要があり，その条件として非負の乱数を生成しなければならないわけですね．
ちなみにC++20以降では `std::uniform_random_bit_generator` コンセプトでこれを確認することができるので，自作の乱数生成器が要件を満たしているかどうかは容易に確認ができます(クラス定義直後に `static_assert(std::uniform_random_bit_generator<MyRandGen>);` とでもしておくとよいでしょう)．

## 最後に

ということで全25日分の記事の感想でした．
参加者の皆様，ご参加いただきありがとうございました！
来年も是非ご参加よろしくお願いいたします！
