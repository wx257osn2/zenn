---
title: "結局std::simdってどうなのさ"
emoji: "🎰"
type: "tech"
topics: ["cpp", "simd"]
published: true
---

:::message
この記事は[C++ Advent Calendar 2025](https://qiita.com/advent-calendar/2025/cxx)の20日目の記事です(遅刻)．
19日目は…[kizul](https://qiita.com/kizul_v2)さんの『[LLVMとLLDをwasmで組み込んで、LLVM系コンパイラの全ビルドから実行までをブラウザ上で完結させよう](https://qiita.com/kizul_v2/items/acc6d8ab7982e9d88704)』です．
:::

[I(`@wx257osn2`)](https://twitter.com/wx257osn2)です．

今年も遅刻なんだ．すまない．
ちょっとバタバタしてて感想ツイートをいい感じにリアルタイムで投稿する余裕もなかったのと，Twitterくんが改悪されまくっておしまいになっているので，今年も昨年に引き続き感想文はまとめてZennに投げる予定です．

# 結局 `std::simd` ってどうなのさ

巷で話題にしてる人が全然いないんだけど，C++26で `std::simd` が入るそうですよ．
というわけで本稿では `std::simd` ってどんな感じなのかとか，参考実装の状況とか，実際に参考実装を使ってみての評価とかをやっていきます．

## そもそも: SIMDって何？

SIMDは"Single Instruction, Multiple Data"の略です．
フリンがコンピューターアーキテクチャを2x2の4種に分類したもの(フリンの分類)の1つとなります:

![flynn.png](/images/std_simd_bcap8fnapwe987/flynn.png)

現代のCPUには大抵SIMD命令というものが実装されています．
つまり，複数のデータに同じ計算をするような命令です．
アイデアとしては，「複数のデータに対して同じ処理Aを要素毎に行う(e.g. 4つの浮動小数点数の絶対値を取る)」を「単一のデータに処理Aを行う(e.g. ある浮動小数点数の絶対値を取る)」のデータ数倍より短い時間(e.g. 前者が2サイクル，後者が1サイクル)で実行できれば，同一の処理を複数のデータに適用する際には高速に(e.g. $\frac{4}{1} / \frac{2}{1} = 2$ 倍で)処理できますね．
そして画像や音声といったマルチメディアには複数のデータに対して同じ計算を適用することが多々ある(データ並列な処理な)ので，SIMD命令を使うと実行速度がグンと速くなります．
他には行列計算などもデータ並列なためSIMD命令が有効です．
現代ではLLMをはじめとした機械学習は大体行列演算ですから，SIMDな操作の重要性は増していると言えます．

そんなSIMD命令ですが，CPUの命令なのでISA(Instruction Set Architecture, 命令セットアーキテクチャ)毎に異なる命令を持ちます．
どれくらい異なるのか，具体例を挙げてみましょう．
x86_64だとAVX2やAVX-512など，aarch64だとAdvanced SIMD(NEON)があります．
x86_64の例として挙げた2つは拡張命令セットなので，CPUによって実装状況はまちまちです．
一方aarch64のNEONはARMv8の基本命令セットに含まれているので，ARMv8のCPUでは常に使うことができます．
AVX2は命令セット上256bitのSIMDレジスタを16個使えることになっています．なっていますが，実装上は128bit SIMDレジスタを2本使って処理する実装がありました(AMD Zen)．gather/scatterが使えますが，deinterleave load/interleave store命令は無いです．masked load/storeも無いです．
AVX-512は命令セット上512bitのSIMDレジスタを32個使えることになっています．なっていますが，実装上は256bit SIMDレジスタを2本使って処理する実装がありました(AMD Zen 4)．gather/scatterが使えますが，deinterleave load/interleave store命令は無いです．masked load/storeを持っています．
NEONは命令セット上128bitのSIMDレジスタを32個使えることになっています．gather/scatterは使えませんが，deinterleave load/interleave store命令があります．masked load/storeは無いです．
ちなみに「NbitのSIMDレジスタをM個使えることになっています」と述べましたが，これはISA側で扱えるレジスタの数で，物理レジスタは必ずしもM本とは限らない点に注意です．多くの場合M本より多い^[Nbit未満のレジスタを複数本使ってる実装が実在する以上，M本未満のCPUも成立はしそうですけど…だいぶ遅そうなのであんまり現実的ではない気がします]物理レジスタを持っており，レジスタリネーミングをかけるようになっているはず．

とまぁこんな感じでCPU・ISA毎に状況はまちまちで，intrinsics(CPUの命令と直接対応するコンパイラ組み込み関数)とかアセンブリ直書きとかで書いてしまうと特定のISAしか対応できなくなってしまいます(例えばAVX-512のintrinsicsでコードを書くとAVX2までしかサポートしてないx86_64 CPUとかaarch64のCPUでは動かせないプログラムになる)．
というかそもそもSIMDのコードを処理系毎に書くのはしんどいという話があります(それはそう)．
一方で，より可搬性の高い方法としてはコンパイラのauto vectorization(自動ベクトル化)に委ねたり，特定処理を複数環境向けに(e.g. AVX2とNEONそれぞれのintrinsicsで)書かれたライブラリを使ったり，SIMD命令のintrinsicsをラップしたライブラリを使うなどがありますが，auto vectorizationは複雑なコードをうまくSIMD化できなかったりしますし，特定処理のライブラリは使いたい処理が無かったり，あるいは処理をカスタマイズしたいときに融通が利かないことが多いです．

SIMD命令をラップしたライブラリならSIMDな操作であることはコンパイラに対してもコードの読み手に対しても明示できるし，環境差異を吸収できるのでいいとこ取りのように見えます．
では現実はどうなのでしょうか．

### SIMD命令統一は険しい道のり

これまでに数多の人類がオレオレSIMD大統一ライブラリを提案し続けてきました:

- [EVE](https://github.com/jfalcou/eve)
    - アクティブなものとしては一番有力な気がする
- [XSimd](https://github.com/xtensor-stack/xsimd)
    - アクティブ．EVEよりちょっと歴史が長い．WASMとかRVVとかもサポートしてるのが強みか
- [Vc](https://github.com/VcDevel/Vc)
    - libstdc++に載ってる `std::experimental::simd` のベースになったライブラリ．もうmaintenance mode
- [libsimdpp](https://github.com/p12tic/libsimdpp)
    - この中では最古か．MIPSをサポート
- [Generic SIMD Library](https://github.com/genericsimd/generic_simd)
    - ACM SIGPLANの並列プログラミング系のトップカンファPPoPP(Principles and Practice of Parallel Programming)のワークショップWPMVP(Workshop on Programming Models for SIMD/Vector Processing) 2014で発表されたライブラリ(なので[論文](https://raw.githubusercontent.com/genericsimd/generic_simd/master/docs/Generic.SIMD.Library.WPMVP2014.pdf)付き)．発表以後更新が無い．ちなみにWPMVP 2014にはBoost.SIMDも出てました
- [UME::VECTOR](https://github.com/edanor/umevector)
    - ISC High Performance 2017で発表されたライブラリ．Expression Templateを使えばオーバーヘッドが抑えられるんじゃ，みたいなことを主張している．ここ数年更新が無い
- [NSIMD](https://github.com/agenium-scale/nsimd)
    - おそらくBoost.SIMDの後継．ここ数年更新が無い

しかし，おそらく皆さんどれもそんなに聞き馴染みがないのではないでしょうか．

正直なところ，どうもSIMDライブラリというのは流行らない傾向にあります．
そもそもSIMD命令を使う動機は，高速なソフトウェアを書きたいからです．
となると当該のSIMD命令でサポートされるあらゆる命令を駆使して全力で速いコードを書きにいきたいわけです．
ところが，SIMD命令間でサポートしている命令がまちまちなので，片や1命令ですむものが他の命令セットでは9命令くらいに膨れたり，そもそもSIMD命令ではどうしようもないのでスカラ命令でちまちまやらないと再現できなかったりします．
前者は比較的遅い(後段のSIMD命令で十分ペイする)とかで済んだりしますが，後者だとそこだけで遅すぎて話にならず，結局全部スカラ命令で処理したほうがマシ…なんてこともザラです．
そうなってくると環境毎にSIMD命令を使った方がいい場合とそうでない場合が出てきたりしてしまうのですが，それ故かこういった「特定の環境では十分速い，そうでない環境でも遅いが動く」みたいなライブラリって意外と少なくて，大抵はいい感じの抽象化，つまりサポートしたいSIMD命令セットの積集合程度の機能に留まる印象があります．
まぁ仮に存在したとしても，一見整った見た目でありながら特定環境では遅くて仕方ないみたいなのは中々扱いにくいですしね…^[一方で初手の移植としてはとりあえず動いてくれるとたすかるみたいな話もあり，このあたりはユースケースによって要求がまちまち(だからなおのこと一般化されたライブラリというのは難しい)]

そういうわけで，結局高速化をやっている人々は各環境向けにintrinsicsやインラインアセンブラを用いて高速な実装を書いてライブラリにする，みたいなことをしがちです．
精々が「当該プロジェクトで必要な範囲の小さいSIMDライブラリを書く」ぐらいで，どうしても汎用SIMDライブラリは採用されにくい傾向にあります．

## `std::simd`

さて，そんな中C++26で採用されたのが `std::simd` ，つまり標準ライブラリでSIMDを統一しようという試みです．
サードパーティ製ライブラリとは異なりある程度コンパイラマジックを前提とできる点^[もしかするとunsized Arm SVEを上手くサポートできるかもしれない．まぁそうしたら今度 `std::simd::vec<T>` のサイズはどうなるんだ問題が出てくるのでそんなに簡単でもないとは思うが…]，標準に入っているのでどんな環境でも使える(少なくとも動きはする)ことが期待できる点，あたりは強みかなと思います．
一方で，私としては本当に十分な速度を得られるのかやや懐疑的ではあります．
というか，上述のように私はintrinsics書けばいいやと思っているのでそんなに必要性を感じていないというところがある(この辺りは[同じくC++26で入った `<linalg>` についてもBLAS有識者は似たようなこと考えてそう](https://168iroha.net/blog/article/202412020035/#idx-1-e37d85b08738f1b4e26ab1a35876fdab9f59b0b8733ac1a4482226986ab07462)な気がします)．

まずは提案文書の方から軽く歴史を追いかけてみましょう．
`std::simd` は多数の提案文書が関わっているので議論を追うのが非常に面倒なのですが([C++26に向けてマージされたものだけで10件](https://github.com/cplusplus/papers/issues?q=is%3Aissue%20label%3Aplenary-approved%20label%3Asimd%20label%3AC%2B%2B26))，とりあえずメインの提案文書は[P1928R15](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p1928r15.pdf)です．リ，リビジョンが多い…
Introductionに記載の通り，当該文書は[2018年に発行のPrallelism TS 2](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/n4793.pdf)に対して[P0214R9](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0214r9.pdf)^[ちなみに著者はP1928R15と同じ人]によって導入された `std::experimental::simd<T>` をベースとして，C++26導入に向けて2019年から足掛け5年議論が続けられてきた大作です．
当初は `std::experimental::simd<T>` ベースで提案されたP1928ですが，5年間に15回もの加筆修正が加えられた上にP1928以外にも多数の提案が加えられた結果かなりインターフェースが変わってしまっています．
具体的にはcppreference.com辺りを参照するとわかりやすいのですが，[`std::experimental::simd<T>` のドキュメント](https://cppreference.com/w/cpp/experimental/simd.html)に対して，[最終的にC++26で入った `<simd>` ヘッダのドキュメント](https://web.archive.org/web/20251222105131/https://cppreference.com/w/cpp/numeric/simd.html)は名前空間もベクトル型の名前も仕様も，連続メモリ空間からベクトル型に起こす方法・連続メモリ空間へ値を書き込む方法も全部違います．
特に `std::datapar` 名前空間は[P3287R3](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3287r3.pdf)によって導入された変更となります．

しかしこう提案文書が多いと最終的な内容がわかりませんね…というわけで[working draftの `[simd]`](https://web.archive.org/web/20251222120128/https://eel.is/c++draft/simd)を読んでいきましょう．

`datapar` がおらんやんけ

---

実はP3287R3の後に[P3691R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3691r1.pdf)がマージされており，命名についてはこれが大凡の最終稿という感じ．
というわけでベクトル型は `std::simd::vec<T>` となりました．
今のところこれを反映したドキュメントサイトは見当たらないので，コードを書く場合はworking draftを読みに行く以外手が無いです．がんばれ^[まぁ私は今回そんなに突っ込んだ機能まで触るに至ってないので，なんとなく雰囲気で書きましたが…]

## 参考実装

さて，入ることも決まったので触ってみたいところです．
`std::experimental::simd<T>` の参考実装はlibstdc++に同梱されていますが，C++26の `std::simd::vec<T>` の参考実装はどこにあるのでしょうか．
`std::simd` などのワードで検索するとパッと見つかるのは[`VcDevel/std-simd`](https://github.com/VcDevel/std-simd)ですが，これは `std::experimental::simd<T>` の参考実装なのでお目当てのものではありません．

実を言うと検索で辿り着く方法は未だにわからないのですが(本当に出てこない)，結論としては提案文書の著者である[Matthias KretzのGitHubアカウント](https://github.com/mattkretz)から辿れる[`GSI-HPC/simd`](https://github.com/GSI-HPC/simd)がお目当てのものとなります．
READMEにはどの提案文書まで実装されてるかが記載してあり…

> ## [Implementation status](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)
>
> | [Feature](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) | [Status](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) |
> | ------- | ------ |
> | [P1928R15 std::simd — merge data-parallel types from the Parallelism TS 2](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) | [done (exceptions see below)](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) |
> | [P3430R3 simd issues: explicit, unsequenced, identity-element position, and members of disabled simd](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) | [done](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) |
> | [P3441R2 Rename simd_split to simd_chunk](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)                           | [done](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)        |
> | [P3287R3 Exploration of namespaces for std::simd](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)                   | [done](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)        |
> | [P2933R4 Extend ⟨bit⟩ header function with overloads for std::simd](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) | [done](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)        |
> | [P2663R7 Interleaved complex values support in std::simd](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)           | [not started](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) |
>
> ### [Design approved, but not in the WD yet](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)
>
> | [Feature](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) | [Status](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) |
> | ------- | ------ |
> | [P3319R4 Add an iota object for simd (and more)](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)                    | [done](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)        |
> | [P2664R9 Proposal to extend std::simd with permutation API](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)         | [in progress](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) |
> | [P2876R1 Proposal to extend std::simd with more constructors and accessors](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) | [not started](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status) |
>
> ## [Missing features](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)
>
> - [`simd<std::float16_t>` is currently disabled / not implemented](https://github.com/GSI-HPC/simd/blob/18ab4f7747bff4c5dcb642ec2154adbe217f7ad7/README.md#implementation-status)

P2663R7(複素数サポート), P2664R11(permutationやgather/scatterのサポート)^[部分的に実装されてるっぽい？時間がないので試せてない], P2876R3(コンストラクタなどの拡張), P3480R6(`std::simd::vec<T>` にrange accessサポートを追加する提案)^[読み取り専用のrange accessサポートは提供されているが，P3480R6では書き込みも可能にするよう書いてある], P3691R1(`std::datapar` 名前空間を消すやつ)の5つは未サポート扱いです．
ここで致命的なのはP3691R1でしょう．
他のは最悪無くても不便なだけで規格に反したコードは生まれませんが，P3691R1については現時点で当該ライブラリを使ってコードを記述しようものなら確実に規格と反した，存在しないはずの名前空間のエンティティを扱うコードとなってしまいます．
困りましたね．

困ったので，[P3691R1を適用済みのものをご用意しました](https://github.com/wx257osn2/GSI-HPC-simd)^[絶妙に面倒な作業で大変だった．あと気づいたのが21日とかで :innocent: って感じだった(それまで何も気づかずに `std::datapar::simd<T>` を使ってコードを書いていました…)]．
提案文書に反して変更量が多いのでちゃんと動くか自信がないのですが，とりあえずCIは通ってるので一旦これを使うことにしましょう．
あとついでに `-Wpedantic` も潰しておきました． `-pedantic-errors` 民もこれで安心！

## 使ってみた

というわけで `std::simd::vec<T>` と適当なintrinsicsやコンパイラのauto vectorizationなどの実行時間を比較してみようと思うのですが，時間が足りないのでsaxpyと適当単精度密行列積の2つで見てみます^[本当は画処理とかも見てみたかったんですが，pack/unpack周り(permute/blend周りとbit_castとかでも代替は効きそうだが)がしっかりサポートされてないとかなりしんどそうな気がしている．bit_castはまだ提案が通ってないしな…他の処理だとgather/scatterほしいし…]．
実行環境はRyzen9 9800X3D，GCC 15.2に `-O3 -march=znver5 -mavx -mavx2 -mavx512f` で挑みます．

### saxpy

saxpyは"Single precision A X Plus Y"，つまり単精度浮動小数点数で $aX + Y$ ($a$ はスカラー値， $X$ および $Y$ はベクトル値)を計算するコードです．
雑に書くなら以下のような感じ:

```cpp:naiveなsaxpy
void saxpy_naive(float a, const float* x, const float* y, float* z, std::size_t n){
  for(auto i : std::views::iota(0uz, n))
    z[i] = std::fma(a, x[i], y[i]);
}
```

基本的にメモリネックなので，「ちゃんとSIMD化できてるのか」「変なオーバーヘッドが載らないか」みたいなのを軽く眺める感じでしょうか．
これを `std::simd::vec<T>` を用いてSIMD化してみます:

```cpp:std::simdでパッと書いてみたやつ
void saxpy_std_simd0(float a, const float* x, const float* y, float* z, std::size_t n){
  std::size_t i;
  for (i = 0; i + std::simd::vec<float>::size() <= n; i += std::simd::vec<float>::size()) {
    const std::simd::vec<float> xs = std::simd::partial_load(x + i, std::simd::vec<float>::size());
    const std::simd::vec<float> ys = std::simd::partial_load(y + i, std::simd::vec<float>::size());
    const std::simd::vec<float> zs = std::simd::fma(a, xs, ys);
    std::simd::partial_store(zs, z + i, std::simd::vec<float>::size());
  }
  if (i < n) {
    const auto remainings = n - i;
    const std::simd::vec<float> xs = std::simd::partial_load(x + i, remainings);
    const std::simd::vec<float> ys = std::simd::partial_load(y + i, remainings);
    const std::simd::vec<float> zs = std::simd::fma(a, xs, ys);
    std::simd::partial_store(zs, z + i, remainings);
  }
}
```

マスクほど高級ではないにせよ，部分読み書きのAPIが生えてるのは好印象です．
また，定数 `a` を直接 `std::simd::fma` に渡すことができています^[当然ですが関数のインターフェースとしてこれをサポートしているだけで，AVX-512に落ちる段階で一度 `vbroadcastss` かなにかで `zmm` レジスタに持ち上げられています]．
この辺りもArm SVEみがあって個人的には好みです．
というわけでこれをコンパイルして実行してみると:

```
array size: 1000003, 1000 times
saxpy_naive:     0.260703ms
saxpy_std_simd0: 0.655166ms
```

無事遅くなりました．なんでや！

結論から書くと，実際に読み込む値が `std::simd::vec<float>::size()` であろうとなかろうと(少なくとも参考実装では) `std::simd::partial_load` を使うとその時点でメチャクチャ遅くなります^[一応AVX-512のmasked loadまでは使ってくれてそうなのですが，masked storeのタイミングでテンポラリバッファに一度普通のstoreで書き出した後，内部でbit maskを作って1bitずつ書き込むかどうか判定してスカラな書き込みをする処理になってるっぽい？なんでだよ]．
だから端数部の処理以外では `std::simd::unchecked_load` を使う必要があったんですね．

```cpp:std::simd難しいなぁ
void saxpy_std_simd1(float a, const float* x, const float* y, float* z, std::size_t n){
  std::size_t i;
  for (i = 0; i + std::simd::vec<float>::size() <= n; i += std::simd::vec<float>::size()) {
    const std::simd::vec<float> xs = std::simd::unchecked_load(x + i, std::simd::vec<float>::size());
    const std::simd::vec<float> ys = std::simd::unchecked_load(y + i, std::simd::vec<float>::size());
    const std::simd::vec<float> zs = std::simd::fma(a, xs, ys);
    std::simd::partial_store(zs, z + i, std::simd::vec<float>::size());
  }
  if (i < n) {
    const auto remainings = n - i;
    const std::simd::vec<float> xs = std::simd::partial_load(x + i, remainings);
    const std::simd::vec<float> ys = std::simd::partial_load(y + i, remainings);
    const std::simd::vec<float> zs = std::simd::fma(a, xs, ys);
    std::simd::partial_store(zs, z + i, remainings);
  }
}
```

```
array size: 1000003, 1000 times
saxpy_naive:     0.260703ms
saxpy_std_simd0: 0.655166ms
saxpy_std_simd1: 0.282943ms
```

…まだnaive実装より遅いんじゃが？
試しにpeel loopを素書きしてみます:

```cpp:端数部の処理をpeel loopで書く
void saxpy_std_simd2(float a, const float* x, const float* y, float* z, std::size_t n){
  std::size_t i;
  for (i = 0; i + std::simd::vec<float>::size() <= n; i += std::simd::vec<float>::size()) {
    const std::simd::vec<float> xs = std::simd::unchecked_load(x + i, std::simd::vec<float>::size());
    const std::simd::vec<float> ys = std::simd::unchecked_load(y + i, std::simd::vec<float>::size());
    const std::simd::vec<float> zs = std::simd::fma(a, xs, ys);
    std::simd::partial_store(zs, z + i, std::simd::vec<float>::size());
  }
  for (;i < n; ++i)
    z[i] = std::fma(a, x[i], y[i]);
}
```

```
array size: 1000003, 1000 times
saxpy_naive:     0.260703ms
saxpy_std_simd0: 0.655166ms
saxpy_std_simd1: 0.282943ms
saxpy_std_simd2: 0.241794ms
```

う，うーん…？とりあえず `partial_load` は遅そうなことはわかった．
しかし，SIMD化されてるようにも見えない…試しにAVX2実装を生やしてみます:

```cpp:AVX2実装
void saxpy_avx2(float a, const float* x, const float* y, float* z, std::size_t n){
  static constexpr std::size_t vector_count = 256uz / CHAR_BIT / sizeof(float);
  const __m256 as = _mm256_set1_ps(a);
  std::size_t i;
  for (i = 0; i + vector_count <= n; i += vector_count) {
    const __m256 xs = _mm256_loadu_ps(x + i);
    const __m256 ys = _mm256_loadu_ps(y + i);
    const __m256 zs = _mm256_fmadd_ps(as, xs, ys);
    _mm256_storeu_ps(z + i, zs);
  }
  for (;i < n; ++i)
    z[i] = std::fma(a, x[i], y[i]);
}
```

AVX2では `_mm256_fmadd_ps` に直接 `float` は渡せないので，事前にSIMDレジスタに持ち上げておく必要があります．
で，結果はというと:

```
array size: 1000003, 1000 times
saxpy_naive:     0.260703ms
saxpy_std_simd0: 0.655166ms
saxpy_std_simd1: 0.282943ms
saxpy_std_simd2: 0.241794ms
saxpy_avx2:      0.0794373ms
```

3倍ちょい速い．なのでやはりメモリ読み込みからしてスカラで動いてそうな雰囲気です．
しばらく頭を抱えていたのですが，以下のコードは十分に高速に動作しました:

```cpp:std::simd速い実装
void saxpy_std_simd3(float a, const float* x, const float* y, float* z, std::size_t n){
  std::size_t i;
  for (i = 0; i + std::simd::vec<float>::size() <= n; i += std::simd::vec<float>::size()) {
    const std::simd::vec<float> xs = std::simd::unchecked_load(x + i, std::simd::vec<float>::size());
    const std::simd::vec<float> ys = std::simd::unchecked_load(y + i, std::simd::vec<float>::size());
    const std::simd::vec<float> zs = a * xs + ys;
    std::simd::partial_store(zs, z + i, std::simd::vec<float>::size());
  }
  for (;i < n; ++i)
    z[i] = std::fma(a, x[i], y[i]);
}
```

```
array size: 1000003, 1000 times
saxpy_naive:     0.260703ms
saxpy_std_simd0: 0.655166ms
saxpy_std_simd1: 0.282943ms
saxpy_std_simd2: 0.241794ms
saxpy_std_simd3: 0.0774773ms
saxpy_avx2:      0.0794373ms
```

待て待て待て，なんで `std::simd::fma` を呼ぶとSIMD化が阻害されるんだ．
流石に無茶苦茶すぎるので，試しに[アセンブリを眺めてみます](https://godbolt.org/z/bjqP483cP)．
すると， `saxpy_std_simd0` から `saxpy_std_simd3` までauto vectorizationが有効だと `zmm` に対する `vfmadd213ps` (=AVX-512のfused multiply add)が呼ばれているのに対して，auto vectorizationが無効だと `saxpy_std_simd0` から `saxpy_std_simd2` までは `vfmadd213ss` (=スカラのFMA)が呼ばれていることがわかります．
どうもGCC15.2の最適化器と `std::simd::fma` の相性があまり良くないらしく，なにかあるとスカラ命令に落ちてしまうみたいです．
というかたぶん `std::simd::fma` が最適化器によるauto vectorizationを期待したスカラ実装になってたりするか…？(ちょっと調査不足なのでよくわかってませんが)
とりあえず， `float{} * vec<float>{} + vec<float>{}` のような形でもちゃんとFMAに落としてくれることも確認できましたし，こちらを使った方が良いかもしれませんね．
また，いずれにせよ `saxpy_std_simd0` は非常に長いスカラ命令が吐かれており，やはり `partial_load` は使うべきではなさそうです．
ちなみに，普通に `float{} * vec<float>{} + vec<float>{}` がFMAに落ちてることからもわかるかとは思いますが， `a` を事前に `std::simd::vec<float>` に持ち上げても結果は変わりません．

実際の計測プログラムでは，複数の配列サイズを定数で渡していたためconstant propagationで実装が配列サイズ毎に特殊化されており，その結果なんかGCCの機嫌を損ねてしまったらしく `std::simd::fma` の呼び出しがSIMD化されなかったようです．
これは参考実装の問題のような気もしますし，GCCの最適化器の実装の問題のような気もします…

ところで，AVX-512が使える環境ではデフォルトでAVX-512を使うようです．
試しにZen2の環境で `-march=native` するとちゃんとAVX2までに留まりました(アセンブリで確認済み)．
一応 `vec<T, 8>` のようにしてやればAVX2の利用を強制することができるはずなのですが，時間が足りんので確認してません．各位の宿題とします^[and You!!]．

### matmul

$M \times K$ の行列 $A$ と $K \times N$ の行列 $B$ を掛けて $M \times N$ の行列Cに結果を入れます．
ナイーブなコードだとこんな感じ:

```cpp:naiveな密行列積
void matmul_naive(const float* a, const float* b, float* c, std::size_t m, std::size_t n, std::size_t k){
  for (auto i : std::views::iota(0uz, m))
    for (auto j : std::views::iota(0uz, n)){
      auto c_ij = 0.f;
      for(auto l : std::views::iota(0uz, k))
        c_ij = std::fma(a[i * k + l], b[l * n + j], c_ij);
      c[i * n + j] = c_ij;
    }
}
```

これをSIMD化していくのですが，密行列積の高速化はかなり奥が深く沼なので^[ループの順番，ブロッキングのサイズ，ループアンロールは何段？など考えるべきことは無限にあるし，プロセッサ毎に最適解は異なるのでみんなは大人しく適当なBLASを叩こう！]，浅瀬でチャプチャプするに留めます．

まず，ベンチマーク的にauto vectorizationさせます．
適当に調べたところ[どうやら最内を `j` の(`N` に対する)ループにする必要があるらしい](https://www.katsuster.net/static/2023/202302.html#20230228)ので，そのように変更します:

```cpp:ループを並び替えてauto vectorization
void matmul_auto_vectorize(const float* a, const float* b, float* c, std::size_t m, std::size_t n, std::size_t k){
  for (auto i : std::views::iota(0uz, m))
    for (auto j : std::views::iota(0uz, n))
      c[i * n + j] = 0.f;
  for (auto i : std::views::iota(0uz, m))
    for(auto l : std::views::iota(0uz, k)){
      const auto a_il = a[i * k + l];
      for (auto j : std::views::iota(0uz, n))
        c[i * n + j] = std::fma(a_il, b[l * n + j], c[i * n + j]);
    }
}
```

```
array size: 3038x3034x3046, 10 times
matmul_naive:          21143.7ms
matmul_auto_vectorize: 1145.71ms
```

さすが密行列積，SIMD化の効能がよく出ます．
先のsaxpyでも確認しましたが，auto vectorizationでは使える範囲でAVX-512，残りに対してAVX2，更に残りに対してSSE4，それでも余ればようやくスカラを使っていく作りになっています．
また，[アセンブリを確認する](https://godbolt.org/z/jr7hhTef5)と分かる通り，2段でループアンロールもかかっているように見えます．気合入ってんね．

それでは早速上記の実装をSIMD化してみましょう:

```cpp:std::simd実装
void matmul_std_simd0(const float* a, const float* b, float* c, std::size_t m, std::size_t n, std::size_t k){
  for (auto i : std::views::iota(0uz, m))
    for (auto j : std::views::iota(0uz, n))
      c[i * n + j] = 0.f;
  for (auto i : std::views::iota(0uz, m))
    for (auto l : std::views::iota(0uz, k)){
      const auto a_il = a[i * k + l];
      std::size_t j;
      for(j = 0; j + std::simd::vec<float>::size() <= n; j += std::simd::vec<float>::size()){
        const auto bs = std::simd::unchecked_load(b + l * n + j, std::simd::vec<float>::size());
        const auto cs = std::simd::unchecked_load(c + i * n + j, std::simd::vec<float>::size());
        const auto result = a_il * bs + cs;
        std::simd::unchecked_store(result, c + i * n + j, std::simd::vec<float>::size());
      }
      for(; j < n; ++j)
        c[i * n + j] = std::fma(a_il, b[l * n + j], c[i * n + j]);
    }
}
```

今回は `std::simd::fma` ではなく，最初から演算子呼び出しにしました．
結果，auto vectorizationの有効無効にかかわらずちゃんとAVX-512のFMAが呼び出せているようです．
ただし，peel loopの部分はauto vectorization次第でAVX2→SSE4→スカラ実装かループアンロールされたスカラコードのいずれかとなるようですね．

```
array size: 3038x3034x3046, 10 times
matmul_naive:          21143.7ms
matmul_auto_vectorize: 1145.71ms
matmul_std_simd0:      1228.37ms
```

流石に勝てませんね．ループアンロール分でしょうか．

次は `j` のループを外に出してみます．

```cpp:jループを最外に追い出す
void matmul_std_simd1(const float* a, const float* b, float* c, std::size_t m, std::size_t n, std::size_t k){
  std::size_t j;
  for(j = 0uz; j + std::simd::vec<float>::size() <= n; j += std::simd::vec<float>::size())
    for(auto i : std::views::iota(0uz, m)){
      auto cs = std::simd::vec<float>{0.f};
      for (auto l : std::views::iota(0uz, k)){
        const auto a_il = a[i * k + l];
        const auto bs = std::simd::unchecked_load(b + l * n + j, std::simd::vec<float>::size());
        cs = a_il * bs + cs;
      }
      std::simd::unchecked_store(cs, c + i * n + j, std::simd::vec<float>::size());
    }
  for(; j < n; ++j)
    for(auto i : std::views::iota(0uz, m)){
      auto c_ij = 0.f;
      for(auto l : std::views::iota(0uz, k))
        c_ij = std::fma(a[i * k + l], b[l * n + j], c_ij);
      c[i * n + j] = c_ij;
    }
}
```

```
array size: 3038x3034x3046, 10 times
matmul_naive:          21143.7ms
matmul_auto_vectorize: 1145.71ms
matmul_std_simd0:      1228.37ms
matmul_std_simd1:      1384.78ms
```

少し遅いのですが，この実装に対してブロッキング^[ローカリティを上げるために小さいループを内側に作るテク．キャッシュミスを減らしてメモリアクセス効率を上げる効果がある]を行います．
ブロッキングサイズを真面目に吟味してる暇は無いので今回は適当に64でやります^[適切なブロックサイズはプロセッサ毎に異なるので，ちゃんとやるならそのマシンに最適なブロッキングサイズを同定すべき]:

```cpp:64でブロッキング
void matmul_std_simd1_blocked(const float* a, const float* b, float* c, std::size_t m, std::size_t n, std::size_t k){
  static constexpr auto block_size = 64uz;
  for(auto i_block = 0uz; i_block < m; i_block += block_size)
    for(auto j_block = 0uz; j_block < n; j_block += block_size)
      for(auto l_block = 0uz; l_block < k; l_block += block_size){
        const auto m_block = std::min(i_block + block_size, m);
        const auto n_block = std::min(j_block + block_size, n);
        const auto k_block = std::min(l_block + block_size, k);

        std::size_t j;
        for(j = j_block; j + std::simd::vec<float>::size() <= n_block; j += std::simd::vec<float>::size())
          for(auto i : std::views::iota(i_block, m_block)){
            auto cs = std::simd::unchecked_load(c + i * n + j, std::simd::vec<float>::size());
            for (auto l : std::views::iota(l_block, k_block)){
              const auto a_il = a[i * k + l];
              const auto bs = std::simd::unchecked_load(b + l * n + j, std::simd::vec<float>::size());
              cs = a_il * bs + cs;
            }
            std::simd::unchecked_store(cs, c + i * n + j, std::simd::vec<float>::size());
          }
        for(; j < n_block; ++j)
          for(auto i : std::views::iota(i_block, m_block)){
            auto c_ij = 0.f;
            for(auto l : std::views::iota(l_block, k_block))
              c_ij = std::fma(a[i * k + l], b[l * n + j], c_ij);
            c[i * n + j] += c_ij;
          }
      }
}
```

```
array size: 3038x3034x3046, 10 times
matmul_naive:             21143.7ms
matmul_auto_vectorize:    1145.71ms
matmul_std_simd0:         1228.37ms
matmul_std_simd1:         1384.78ms
matmul_std_simd1_blocked: 577.327ms
```

勝ちました．多分これが一番速いと思います^[そんなことはない/それはそれとして今回の実装の中ではこれが最速です]．
ちなみにアセンブリを見てみると `vmulps` と `vaddps` に分かれていて，FMAになっていません．
これはおそらく `cs` に対する依存が伸びまくるのを防ぎたい(`a_il * bs` の部分だけでも独立して計算可能な状態にすることでOoOのスケジューラーの負荷を下げたい)みたいな話かな～と予想していますが，詳しいことはわかりません．

それではAVX-512およびAVX2に移植していきます．結果は以下です:

```
array size: 3038x3034x3046, 10 times
matmul_naive:             21143.7ms
matmul_auto_vectorize:    1145.71ms
matmul_std_simd0:         1228.37ms
matmul_std_simd1:         1384.78ms
matmul_std_simd1_blocked: 577.327ms
matmul_avx512_0:          1233.34ms
matmul_avx512_1:          1396.26ms
matmul_avx512_1_blocked:  679.948ms
matmul_avx2_0:            1100.04ms
matmul_avx2_1:            2650.26ms
matmul_avx2_1_blocked:    1222.49ms
```

やはり計算が重きを占めてくるとレジスタ長が長いSIMD命令が本領を発揮しますね．
アセンブリからも分かる通り， `_fmadd_ps` で記述してあるAVX-512/AVX2 intrinsicsで記述された実装はちゃんと `vfmadd231ps` に落ちていました．
もしかすると上述のようにこれが原因で `matmul_std_simd1_blocked` と実行時間に差がついてしまっているかもしれません．
ただ今回は実際の実行バイナリでは(constant propagationの結果として)どのように振る舞ってるのか逆アセンブルして調べる時間が取れてないので，もしかすると思いもよらない理由で遅くなったりしてる可能性もあります．
これについても読者のみなさまの課題とします．

最後に，リポジトリを置いておきます: https://github.com/wx257osn2/std_simd_sample

# まとめ

というわけで `std::simd::vec<T>` の歴史的変遷をざっくり紹介し，参考実装を使ってみてAVX512のintrinsicsを使った場合とどのように違うのかを軽く見てみました．
うーん，とりあえず現時点での `std::simd::partial_load` (ともしかすると `std::simd::partial_store` も？)と `std::simd::fma` の振舞いはかなり罠なのと，デフォルトで環境で最も長いbit長のSIMDレジスタを使う命令を発行しようとしてしまうので，環境によってはちゃんと明示的にAVX2/SSE4を使うように `vec` の第二引数に値を入れてあげないとサポートはしてるけど遅い命令を発行されてしまって速度低下の原因になりそうです．
まぁ想像してたよりは使えそうな雰囲気はある反面，やはりpermutation・shuffle・pack/unpackなどの操作や，型の拡張・縮小操作，デインタリーブ入力・インタリーブ出力などがどのようなインターフェースで提供される・あるいはされないのか，そしてそれらの性能はどんなものなのかは気になるところです．
コンパイラにちゃんと実装されるまでにはまだまだ時間がかかりそうですが，一足先に使用感を体感したければ私の用意した実装を使うとよいでしょう．

---

21日は[マホウ(`@tyanmahou`)](https://twitter.com/tyanmahou)さんの『[静的リフレクションでニコニコしよう^^](https://qiita.com/tyanmahou/items/2f80f3315dd500e87ee5)』です．
