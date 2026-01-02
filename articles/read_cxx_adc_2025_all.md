---
title: "C++ Advent Calendar 2025 感想文(書きかけ:5日目まで)"
emoji: "🎄"
type: "tech"
topics: ["cpp"]
published: true
---

2026年が始まってしまいました．今年も皆様よろしくお願いします．
さて，昨年も[C++ Advent Calendar 2025](https://qiita.com/advent-calendar/2025/cxx)にはたくさんの方にご参加いただきありがとうございました．
なんだかんだとバタバタしていて12月中には読めていなかったので，年を跨いでしまいましたが感想文を書こうと思います．

# C++ Advent Calendar 2025 感想文

## Day1 『[【C++AdC】std::source_locationで呼び出し元を検知する](https://zenn.dev/plasma_e/articles/35d8527669eca1)』

1日目， `std::source_location` をいい感じに埋め込みたい話．
この手の話だと私は記事とは異なりスタックトレースを使ってしまいがちで，過去には

- [Windows APIのラッパーライブラリ](https://github.com/wx257osn2/will)でネイティブのスタックトレース機構をexpectedや例外に載せることでWin APIの失敗事由をcall stack付きで表示できるようにしたり，
- MSVCに `std::stack_trace` が実装された頃に `std::expected` と `std::bad_expected_access` に無理やり `std::stack_trace` を載せる実験をしたり(非公開)

したことがあります．
ちなみにこれらはいずれも「 `std::expected` でエラーを得た箇所と，それを `std::bad_expected_access` で発火した箇所は異なるので両方のスタックトレースが必要だよね」みたいなのが根底のモチベーションなのですが，まぁ本筋とは関係ないので一旦置いておきます．

さて，記事中では `std::source_location::current()` を呼び出し元からどのようにして受け取るか，という問題で苦心していました．
これは `std::stack_trace::current()` でもある程度同じ問題を抱えているため，わかるな～という気持ちがあります．
可変長引数とデフォルト引数が相性悪いんですよねぇ．
また， `std::expected` などと組み合わせた際は `current()` を取る位置が `expected` の移譲されたコンストラクタだったり `std::expected::value()` 内だったりすることがあり，見栄えが良くないことも多かったです．
一方で，スタックトレースは呼び出し箇所ピンポイントのみならずそこまでの過程も拾えるので，極端な話見栄えを気にしなければ一応使えないこともないです．
`source_location` はその辺りがかなりシビアなので，デフォルト引数をどのように受けるか，という問題はより深刻と言えます．
まぁだからといって引数全部に「デバッグ情報を追加」と明記するのも大変そうなので，なんかもっとこう…アスペクト指向的なアプローチとか，DSLっぽい方策を持ってきたほうが方向性としては良さそうな気がします(具体的になにかアイデアがあるわけではないですが)．

ちなみにIさんはデバッグといえば由緒正しきprintfデバッグです．
ようわからんフレームワークとか使ってるとプロセスが複数いたりスレッドが勝手に生えたりしてようわからんので，なんだかんだいって結局これに限る．

## Day2 『[C++で正規表現](https://qiita.com/K10K10/items/3f2e5cb3c964de45a80e)』

2日目， `std::regex` の話．
正規表現を使う際は `)"` が出てきうるので，私は大抵生文字列リテラルにはprefix/suffix(_d-char-sequence_)を指定しますね．
例えば次のように: `R"re(U"([a-z]*|42)")re"` (これは `U"abc"` や `U"42"` にマッチする正規表現)
prefix/suffixは大抵 `re` を使いますが，これで都合が悪ければ適当に変えます．

> markdown の*強調*(*)をHTMLの\<em>\</em>へ置換する

ちょっとしたサンプルコードなのであんまり言っても仕方のないことですが，これは実用上あんまり上手く行かないですね．
とりあえず提示されたコードは

- 文字列として空白を入力できないので `std::cin >> input;` を `std::getline(std::cin, input);` に変える
- 全体マッチしないと置換してくれないので `regex_match` を `regex_search` に変える

として，例えば `* abc*` というmarkdownを考えてみましょう．
これは記事のコードだと `<em> abc </em>` となりますが，実際には以下のようになります:

* abc*

そう， `<li> abc*</li>` に落ちるべきなのです．
他にも `*abc *` もまた *abc * と一般的なMarkdown処理系では強調されません．
これは[CommonMark SpecのEmphasis and strong emphasisの節](https://spec.commonmark.org/0.31.2/#emphasis-and-strong-emphasis)辺りにも書いてありますが，空白文字が周囲にある場合delimiter runとして扱われなくなりうるからですね．
他にも `*abc* d*` が `<em>abc</em> d*` ではなく `<em>abc* d</em>` とされてしまう問題もありますから，
少なくともこれらについてケアしてみると `R"(\*(\S|\S.*?\S)\*)"` ぐらいの複雑さにはなるでしょうか．
それでも `*abc\*` なども `<em>abc\</em>` として処理されてしまう問題がありますから，ちゃんとした実装にするにはまだまだ遠い道のりです．
[あなたと犬小屋，今すぐ動作確認](https://wandbox.org/permlink/iDZcfdRhR3v5T84f)

ところで上記の `*abc* d*` のために最左最短マッチ(`.*?`)を導入しましたが，これを入れると今度は `*a **bc** d*` などが `<em>a *</em>bc<em>* d</em>` となり壊れます(最長マッチであれば `<em>a **bc** d</em>`)．
Markdownは他にも複雑な仕様をいくつも持っており，こうした仕様を正しく処理するには正規表現ではなく構文解析が必要^[一般に入れ子を正しく扱うには構文解析が必要なことが広く知られています]です．
`std::regex` は入りましたが，皆さんも必要に応じてときには横着せずちゃんと構文解析をしましょう^[Iさんのヒミツ: 実は構文解析が専門だったらしい]．

## Day3 『[RESTful malloc Server × std::mdspan AccessorPolicy](https://qiita.com/yohhoy/items/88b40a4d17d087fdcf6f)』

3日目， `std::mdspan` のアクセスポリシーの抽象化能力がすごいの話．
本来 `std::mdspan` のアクセスポリシーは「オフセットが与えられたときそれを用いてメモリ空間にどのようにアクセスするのか」を扱うクラスです．
例えばアラインされたメモリ空間に対してストライドアクセスしたり，あるいは与えられた多次元配列がその周辺座標にまで拡張されてると見做したり(2次元配列が無限に繰り返される空間であれば剰余によって元の領域にマップするアクセッサを考えられる)，といった形です．
しかし，アクセスポリシーがアクセス時に返す要素への参照型 `reference` は必ずしもアクセス先の要素型 `element_type` に対する参照である必要は無く，そこにプロキシ型を挟むことでメモリ空間へのアクセスのみならず「多次元配列」のように見做せる任意の読み書き操作を扱うことが可能です^[この説明からもわかる通り，そして記事にも記載の通り， `reference` は原則として `element_type&` であることが想定されているはずで，これはライブラリ機能の濫用であるという認識は持っておいたほうが良いでしょう．それはそれとしてできるので…]．
というわけで，メモリ空間の確保・解放・アクセスをHTTP経由で行えるサーバー・クライアントを実装して隠蔽してみた，というのが記事の内容．

元ネタは記事末尾に記載の通りいくつかあって，技術的な部分(`std::mdspan` のアクセスポリシーを濫用する話)はCppCon 2023のLT "Spanny: Abusing C++ mdspan is Within Arms's Reach" (拙訳: Spanny: mdspanの濫用は手の届くところに^[ロボットアームの手が届く範囲にアクセスできる，という話とかけているオシャレタイトル…のはず])です．
このLTではロボットアームに備え付けられたセンサからビール箱の中身を確認し，箱の各スロット内にビールが入ってるかどうかを確認する，というコードを `mdspan` を2重for文で回すようなコードで実現するためにアクセスポリシーを自作する話でした．
そしてもう一方のRESTful malloc serverについては主に私と親しい人々の身内ネタなのですが，これの出自については長くなるため[別記事にまとめました](https://gist.github.com/wx257osn2/4942ce3b8e8f57f037f24d1316b38335)．「低級言語」って何の話？といった疑問をお持ちであれば是非．

## Day4 『[Expression templatesのダングリング対策について](https://sukeya.github.io/articles/expression_templates/)』

4日目，expression templateの中間クラスでメンバにconst lvalue-refを持たせるとdangling referenceが発生しうるのをどう回避するかという話．
私はよくETを用いたライブラリを作っているのですが，後述する理由でETの各ノードのメンバは全部値型で持たせてきました．
値型であれば寿命はオブジェクトと同期するので何も心配しなくてよいわけですね．
しかしETで生成される式なんてどうせ一時オブジェクトしか含まれないやろ，ぐらいの雑な気持ちでいると，たしかに記事のように名前束縛したものを使うことは十分考えられますし，寿命がETの式より長いことがわかっているなら参照で取り回したほうがコピーコストが削減できて嬉しいみたいなことはありうるのですよね．
実際，試しに `BinaryExpr` のメンバを値型で取り回すように書き換えてみると `main` 冒頭で `std::function` のコピーコンストラクタが呼ばれるようになる(`f + g` でコピーを回避できていない)ことがわかります．

さて，実はこの問題標準ライブラリ上で既にアプローチがなされています．
そう， `std::reference_wrapper` です．
意外と知られていないことですが `std::reference_wrapper` は `operator()()` を持っていますので，関数オブジェクトの参照を取り回す際に透過的に使用することができます^[`std::reference_wrapper` の定義ヘッダは `<functional>` であることを思い出しましょう．STLアルゴリズムのcallbackで使うことなどを念頭に設計された部分があることがわかりますね]．
つまり，値セマンティクスとして通常通りコピーしてほしければ `f` ，ムーヴして渡したければ `std::move(f)` ，そして参照として持ち回してほしいのであれば `std::cref(f)` とすればよいわけです．

プログラマが適切に呼び出すことを念頭に置けば^[C++を使うユーザーにはこれを期待してよいでしょう]，やはりETの各ノードのメンバは値型で持たせて良さそうです．
そういうわけで，私は値型で持たせています，という話でした．

## Day5 『[C++コンパイル時コード生成で（Rustみたいに）先を見通す型推論　〜コンパイルは2回〜](https://qiita.com/Raclamusi/items/6a0fe8134e2bba6dbf59)』

5日目，C++にtype inferenceを導入したい！の話．
実現手段として，コンパイルを2回します． _おまえは何を言っているんだ_ (画像略)

実は日本語で「型推論」と呼ばれるものは2種類あります: type deductionとtype inferenceです．
いずれも型の記述を省略できる機能ですが，その内実は少々異なります．
Type deductionは主にC++の用語で， `auto` キーワードや `template` 関数の `template` パラメータなど，直接渡された式からその型を導出する仕組みです:

```cpp:type deductionの例
template <typename T>
auto f(T x) {  // 引数に渡された式からTを導出する
  return x;    // return文に渡した式から戻り値型を導出する
}

auto a = 3.14f;  // 初期化式から変数の型を導出する
```

これらはいずれも右辺から左辺，式で変数や戻り値といったものを初期化^[C++ワードとしては厳密ではない]する際に式の型をそのまま充てるといったものです．
一方type inferenceの対象はより広く，例えば以下のように「使われ方」から型情報を推論することができます:

```cpp:type inferenceの例(実際にはできないが)
void f(double& x);

auto x;
f(x);  // double型の引数として使われているのでxはdouble型
       // (実際にはxの定義時点で型が定まらないのでエラー)

int g(auto x) {
  return x;  // xはint型の戻り値として返されるのでxはint型
}            // (実際には呼び出し毎に異なるgがインスタンス化され，intに暗黙変換可能な場合のみコンパイルが通る)
```

C++はtype deductionはできますがtype inferenceはできません．
これはC++の型システムがボトムアップに型を同定していくからです: primary expressionが型を持ち，compound expressionはその要素のprimary expressionから型が決定されます．
primary expressionの型が不明なとき，そのプログラムはill-formedです．
そしてボトムアップに型を持ち上げる際に記述を省略可能なのがtype deductionです．
一方，Hindley-Milnerベースの型システムでは型の制約を連立方程式のように列挙して各変数の解を求めるように単一化していきます．
これは方向とか無い操作なので，型検査時点で型が明示されていない変数もその全ての利用箇所から総合して型が決定されます．
言うなればHMベースの型システムにおいてtype inferenceとは型検査の帰結として当然起こる事象なのです^[この説明では多相に関しては説明を省いているので，詳しく知りたい方は頑張って型システムについて勉強しましょう]．

前置きが長くなりましたが，6日目の記事はコンパイルを2回行うことでC++上にtype inferenceもどきを実装する記事です．
TeXとちゃうねんぞ！って思ったら記事内でも言及があって笑いました．
項書換え系を複数回回すのと一般的なプログラミング言語を複数回コンパイルするのは話がちょっと違うのでは…という気も一瞬しましたが， `HPA_IGNORE(` を `typefile.inc` 冒頭において問題なく処理されるのは2回目のコンパイルでCPPとかいう項書換え系が走り直すからなので，TeXから着想を得たアイデアで間違いないです(？)
1回目のコンパイル時に `static_assert` を使って2回目のコンパイル時に必要なコードをコンパイルエラーとして生成することでコード全体の文脈を一度得た状態で2回目のコンパイルに臨めるので，これを用いてtype inferenceを実現するというわけですね．

ところで，本来のtype inferenceであれば上述のように関数引数として渡したらその関数引数の型から推論したり，あるいは関数のreturn文の式とその返り値型から推論したりできるはずですが，残念ながらhyper autoではこれは上手く動きません．
また，C++には暗黙変換がありますから，例えば `double` と `int` 両方を渡した場合は `double` として推論してほしいですが，これも同様にhyper autoではできません．
というわけで，これらをサポートしてみましょう．

まず前者については `unknown<ID>` が `T` に暗黙変換された場合に `infer` しないのが原因ですから，以下のメンバ関数を `unknown<ID>` に追加することで実現できそうです:

```cpp
    template <class T>
        requires not_same_as<std::remove_cvref_t<T>, unknown>
    constexpr operator T() {
        infer<std::decay_t<T>>();
        return std::declval<T>();
    }
```

一方後者は非常に面倒です．
というのも，現状hyper autoでは1回目のコンパイルで `template <> struct hpa::hyper_auto<ID> { using type = T; };` のようなコードを出力しているのですが，複数の異なる型で呼ばれると

```cpp
template <> struct hpa::hyper_auto<1> { using type = double; };
template <> struct hpa::hyper_auto<1> { using type = int; };
```

のようになってしまうので特殊化の多重定義でコンパイルエラーになってしまいます．
これを回避するためにはまず `hpa::hyper_auto<ID, Depth>` のように引数を増やして，なんとかして

```cpp
template <> struct hpa::hyper_auto<1, 0> { using type = double; };
template <> struct hpa::hyper_auto<1, 1> { using type = int; };
```

のようにしてやる必要があります．
その上で， `type` の定義を `std::common_type_t<T, typename hpa::hyper_auto<ID, Depth - 1>::type>` のようにしてやればいい感じに複数の型の共通型を取れるはずです．
しかし，コンパイル時に `infer` の呼び出し毎に異なる `Depth` を出力することなど可能なのでしょうか．
実は昔は[コンパイル時に呼び出される度に値が増える関数を実装できた](https://txt-txt.hateblo.jp/entry/2015/05/16/173721)のですが，現在はこれはDRで塞がれてしまっています．

結論から言うと，これは現代でもコンパイラマジックを使えば可能です: `std::__is_complete_or_unbounded()` というbuiltinは直接呼び出した場合はコンパイル時でも呼び出される毎に常に最新の結果を返します．
したがって，以下のコードは多重定義によるエラーとなりません:

```cpp
template <> struct hpa::hyper_auto<1, std::__is_complete_or_unbounded(std::__type_identity<hpa::hyper_auto<1, 0>>{}) + std::__is_complete_or_unbounded(std::__type_identity<hpa::hyper_auto<1, 1>>{}) + std::__is_complete_or_unbounded(std::__type_identity<hpa::hyper_auto<1, 2>>{}) + std::__is_complete_or_unbounded(std::__type_identity<hpa::hyper_auto<1, 3>>{})> { using type = double; };
template <> struct hpa::hyper_auto<1, std::__is_complete_or_unbounded(std::__type_identity<hpa::hyper_auto<1, 0>>{}) + std::__is_complete_or_unbounded(std::__type_identity<hpa::hyper_auto<1, 1>>{}) + std::__is_complete_or_unbounded(std::__type_identity<hpa::hyper_auto<1, 2>>{}) + std::__is_complete_or_unbounded(std::__type_identity<hpa::hyper_auto<1, 3>>{})> { using type = int; };
```

というわけで後はいい感じに辻褄を合わせれば動きます．
[あなたと犬小屋，今すぐ動作確認](https://wandbox.org/permlink/Ai9PSZcM618waxdQ)

## Day6 『[Re: C++は常に進化している！ C++26・C++23の新機能と今後のトレンド](https://qiita.com/wx257osn2/items/e227f01e1bdff2dd7082)』



## Day7 『[C++ 合成関数と逆関数; 艦これのダメージ検証における利用](https://qiita.com/kc-hedgehog/items/f55dd014b49c9467ddae)』



## Day8 『[C++の例外の使い方を少しだけ深く理解する](https://168iroha.net/blog/article/202512080000/)』



## Day9 『[C++26リフレクションでつくる tuple と variant](https://zenn.dev/yaito3014/articles/reflection-tuple-variant)』



## Day10 『[コンパイラ様のご機嫌を取りたかった2025冬](https://zenn.dev/yuk__to/articles/5c397cceed4d5b)』



## Day11 『[C++のテンプレートパラメーター制限](https://qiita.com/K10K10/items/3710f40c74e283ebae1b)』



## Day12 『[vcpkg でライブラリを楽に導入しよう](https://qiita.com/norisio/items/7589fec20a4a70e624bf)』



## Day13 『[(C++23) flat_map: シンプルだが効率的なコンテナ](https://qiita.com/hon_no_mushi/items/ab9b534cac1c729f7b81)』



## Day14 『[C++良すぎる　特にtry文](https://qiita.com/Raclamusi/items/5634c03f1fb07f4d0b54)』



## Day15 『[Visual Studio 2026 18.0 の C++ 新機能を試す](https://zenn.dev/reputeless/articles/cpp-vs2026-1950)』



## Day16 『[std::hive とは](https://qiita.com/shibainuudon/items/89e3058bded69aec41cc)』



## Day17 『[std::hiveを実装する](https://qiita.com/shibainuudon/items/2fbc6c3d8cf9c20034f2)』



## Day18 『[AngelScript を C++ プロジェクトに組み込む](https://zenn.dev/sashi0034/articles/bf06646e0d88ac)』



## Day19 『[LLVMとLLDをwasmで組み込んで、LLVM系コンパイラの全ビルドから実行までをブラウザ上で完結させよう](https://qiita.com/kizul_v2/items/acc6d8ab7982e9d88704)』



## Day20 『[結局std::simdってどうなのさ](https://zenn.dev/wx257osn2/articles/std_simd_bcap8fnapwe987)』



## Day21 『[静的リフレクションでニコニコしよう^^](https://qiita.com/tyanmahou/items/2f80f3315dd500e87ee5)』



## Day22 『[［C++］C++26 Contracts](https://onihusube.hatenablog.com/entry/2025/12/25/093454)』



## Day23 『[【C++】オブザーバーパターンでイベント駆動設計を!](https://qiita.com/tsukino_/items/d8950855b7d5278c3efb)』



## Day24 『[C++20のコルーチンを理解したい](https://qiita.com/tomolatoon/items/eb55f2d534d4869bb63e)』



## Day25 『[]()』



## 最後に

ということで全25日分の記事の感想でした．
参加者の皆様，ご参加いただきありがとうございました！
来年も是非ご参加よろしくお願いいたします！