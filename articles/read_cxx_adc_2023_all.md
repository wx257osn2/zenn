---
title: "C++ Advent Calendar 2023 感想文"
emoji: "🔙"
type: "tech"
topics: ["cpp"]
published: true
---

2024年，今年も年の瀬ですが皆様いかがお過ごしでしょうか．
年末の風物詩といえばAdvent Calendarですが，今年も[C++ Advent Calendar 2024](https://qiita.com/advent-calendar/2024/cxx)にはたくさんの方にご参加いただきました．
皆様お疲れ様でした！
あとは私が今年の記事を全部読んで感想を書けばおしまいですね．

ところで，何か忘れているような…

> [AdC、とりあえず自分のを書き終えたら読みます](https://x.com/wx257osn2/status/1736039352722190823)

> [年末バタバタする等の理由で年が明けてからゆっくり読む予定です．](https://zenn.dev/wx257osn2/articles/try_to_make_a_try-saiudhleghbaowlb)

> [GWの目標、C++ AdC 2023感想文全部書くで…対よろ…](https://x.com/wx257osn2/status/1781318385969815769)

> [ところでGWは転と生活をしていたら終わったし転は普通に失敗したわけだが，AdCの感想文はいつ書くんでしょうか](https://x.com/wx257osn2/status/1813045512532291743)

> [ところで夏休みなので宿題があります，こいつは未だにやってなかった去年の冬休みの宿題](https://x.com/wx257osn2/status/1832328353992356322)

そう！[去年のAdvent Calendar](https://qiita.com/advent-calendar/2023/cxx)の感想を書いていないのである！！
というわけで，2024年も終わりかけですが2023年のC++ Advent Calendar全日程の感想文をここに記します．
例年([2021](https://x.com/wx257osn2/status/1466118644522708993), [2022](https://x.com/wx257osn2/status/1587328330537500672))はTwitterにクソ長ツリーを作っていたのですが，API制限が厳しすぎてまともにツイート取得できないサービスをそういう用途で使うのは厳しいのと，リアルタイムに追いかけるならまだしも1年も経ってからSNSでやってもな…という気持ち，2023年は例年にも増して感想が長大になる記事が多かったことから，Zennでの記事執筆という体を取ります^[C++ AdC 2024感想文も同様にZennでの公開を予定していますが，来年以降どうするかは未定．リアルタイムにやれるなら[お空](https://bsky.app/profile/wx257osn2.template.typena.me)とツイで同時公開とかも検討中(お空の方がAPI liimtが緩そうなので後から追いかけるときには楽かなと思っている)]．
また，本稿に相当量の技術的話題が含まれることから，Tech記事としています．

:::message
というわけでこの記事は[C++ Advent Calendar 2023](https://qiita.com/advent-calendar/2024/cxx)の23日目が空いているので1年後に枠に埋めたものです(本来のAdC記事ではない)．
一応書いておくと22日目は[ラクラムシ(@raclamusi)](https://twitter.com/raclamusi)さんの「[C++ コンパイル時「出力」で画像ファイル生成](https://qiita.com/Raclamusi/items/fd9c5b52c514a6420e2c)」でした．
:::

# C++ Advent Calendar 2023 感想文

## Day1 「[C++の環境構築に関して](https://qiita.com/exli3141/items/17f6682916909e13914d)」

1日目，vckpgとCMakeとpkg-configを組み合わせてMSVC環境でGtkを使う話．
実際のところ，Rustの体験の良さの5割くらいはCargoが担ってるとこあります[要出典]し，ビルドシステムとパッケージマネージャというのは現代では非常に重要なところです．
一方でC++は古くから存在する言語ですし，処理系も複数あってコンパイラオプションも様々，ライブラリも各々多様なビルドシステムを使用，と中々いい感じの抽象化をするのが難しいのが現実でもあります．
私自身[C++のパッケージマネージャを弄っている](https://zenn.dev/wx257osn2/articles/current_poac-iuartb123ir2)身ですし，他のプロジェクトの現状を把握しておくべきでしょう．

ふむふむ，なるほど．
vcpkgが裏でpkg-config自体を抱えるんですね．
力技だけどたしかにそれなら既存ライブラリも上手いこと扱えるかもしれん．
どれ，試しに「[実演：vcpkgの力を借りたVisual Studio上でCMakeを走らせてのGtkスケルトン作成](https://qiita.com/exli3141/items/17f6682916909e13914d#%E5%AE%9F%E6%BC%94vcpkg%E3%81%AE%E5%8A%9B%E3%82%92%E5%80%9F%E3%82%8A%E3%81%9Fvisual-studio%E4%B8%8A%E3%81%A7cmake%E3%82%92%E8%B5%B0%E3%82%89%E3%81%9B%E3%81%A6%E3%81%AEgtk%E3%82%B9%E3%82%B1%E3%83%AB%E3%83%88%E3%83%B3%E4%BD%9C%E6%88%90)」を手元で試してみるか…え？サンプルリポジトリが無いんですか！？

### 実際にやってみた

ということで，Day1の記事内容の追試を行います^[このようなことをやっているから時間がかかったという側面が多分にあります(言い訳)]．
見せてもらおうか，vcpkgとやらの実力を！

成果は[ここ](https://github.com/wx257osn2/cxx_adc_2023_day1_example)にあります．

#### Visual Studioを入れる。

手元のVS2022だとCMakeは入っていたがvcpkgが入ってなかったのでインストーラーを起動してインストールして…再起動しないといけないの！？めんどくさいなぁ…

確認方法は， `x64 Native Tools Command Prompt for VS 2022` を起動して

```console
**********************************************************************
** Visual Studio 2022 Developer Command Prompt v17.11.5
** Copyright (c) 2022 Microsoft Corporation
**********************************************************************
[vcvarsall.bat] Environment initialized for: 'x64'

C:\Program Files\Microsoft Visual Studio\2022\Community>where cmake
C:\Program Files\Microsoft Visual Studio\2022\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe

C:\Program Files\Microsoft Visual Studio\2022\Community>where vcpkg
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\vcpkg\vcpkg.exe

C:\Program Files\Microsoft Visual Studio\2022\Community>where ninja
C:\Program Files\Microsoft Visual Studio\2022\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja\ninja.exe

```

となればOK． `情報: 与えられたパターンのファイルが見つかりませんでした。` と返ってきたものがあったら足りないのでインストールしましょう．

#### プロジェクトの作成

> なおある不具合の関係から、ソースフォルダはCドライブ直下に入れておく。

確認した．パスが長すぎるとちょうどgtkのビルドのタイミングで `CreateProcess` の長さ制限に引っかかってコケる．もう令和やぞ，なんとかしてくれ．

> [この文字列の最大長は 32,767 文字](https://learn.microsoft.com/ja-jp/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa)

ちなみに何故コケるかというと，gtkのビルドに際し依存関係すべて(それこそフォントレンダリング用に[`freetype`](https://freetype.org/)とか，ほんとに全部だ)がリンカのコマンドラインに並ぶわけだが， `C:\Users\user\AppData\Local\vcpkg` とかいう無駄に深いディレクトリにファイルが配置されているため，凄まじい勢いで文字数を消費するのである．
プロジェクトのディレクトリが短いとかろうじて足りる．
というか私は大丈夫だったけど，ユーザー名が長いとその時点でダメな可能性ある．
その場合は短いユーザー名のユーザーを作ってやるしかなさそう．
上述のように， `CreateProcess` の仕様があんまりな気もするし，vcpkg側もMicrosoft謹製なんだからもうちょっと配慮してディレクトリ名を決めて欲しい．
ともかく，記事に記載の通り，どこかのドライブの直下に8文字ぐらいのディレクトリ名で置いておけば問題なく通りそうである．

#### CMakePresets.jsonの設定

宇宙猫である．聞いたことがないワードが飛び出してきた． `CMakePresets.json` is 何…？

こういうときは[公式ドキュメント](https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html)を頼るに限る．というわけで読んでみたのですが，どうも `cmake` 実行時に大量の `-D` を渡すのは良くないよね，ということでJSONでいい感じに指定できるようにしたものっぽい．はえ～，現代はこんなことになってるんですねぇ． `CmakePresets.json` はプロジェクト固有の記述を， `CMakeUserPresets.json` はユーザー固有の記述をするらしい．なるほど．
それでは記事からコピペを…

> CMakePresets.jsonの共通項部分。

動くファイルが置いてない！！！！！！というわけで上記ドキュメントを参考に自力で書きます．
動きました．

#### vcpkgの設定

`builtin-baseline` を空にして `vcpkg install` を実行する．

```console
C:\gtkmmtes>vcpkg install
A suitable version of git was not found (required v2.7.4) Downloading portable git 2.7.4...
Downloading git...
https://github.com/git-for-windows/git/releases/download/v2.43.0.windows.1/PortableGit-2.43.0-64-bit.7z.exe->C:\Users\user\AppData\Local\vcpkg\downloads\PortableGit-2.43.0-32-bit.7z.exe
Downloading https://github.com/git-for-windows/git/releases/download/v2.43.0.windows.1/PortableGit-2.43.0-64-bit.7z.exe
Extracting git...
the top-level builtin-baseline () was not a valid commit sha: expected 40 hexadecimal characters.You can use the current commit as a baseline, which is:
        "builtin-baseline": "6f1ddd6b6878e7e66fcc35c65ba1d8feec2e01f8"
note: updating vcpkg by rerunning bootstrap-vcpkg may resolve this failure.
```

これを `vcpkg.json` に書いておけばいいらしい．
これはなんなのかというと，[`microsoft/vcpkg`](https://github.com/microsoft/vcpkg)のコミットハッシュである．
vcpkgはそのツール本体とパッケージレジストリを同じリポジトリで管理しており，パッケージレジストリとしての `microsoft/vcpkg` の最低バージョンをコミットハッシュで指定するのが `builtin-baseline` らしい．
`builtin-baseline` より古いパッケージは自動で更新されるそうな．

##### あらかじめのインストール

```
R:\gtkmmtes>vcpkg install
R:\gtkmmtes\vcpkg.json: error: $.name (a package name): "CMake-gtkmm" is not a valid package name. Package names must be lowercase alphanumeric+hypens and not reserved (see https://learn.microsoft.com/vcpkg/users/manifests for more information).

Extended documentation available at 'https://learn.microsoft.com/vcpkg/users/manifests'.
```

とのことで，少なくとも2024年末時点では大文字パッケージネームは許されていないらしい．
まぁ，大文字小文字を区別しないOSもサポートすることを踏まえれば妥当な判断だと思う．
というわけで **今後記事内の `CMake-gtkmm` をすべて `cmake-gtkmm` に変更する．**

修正したら再度 `vcpkg install` を実行する．
先述のようにgtkを動かすのに必要な依存関係のすべてをビルドするため，初回だと死ぬほど待たされる^[1Gbpsのまともな光回線とPCIe4x4 M.2SSDとRyzen9 3950X環境でも1時間弱かかった(WindowsとかいうファイルI/OすべてにフックをかけてWindows Defenderとかいうやつが監視をかけているOSだからという話もありそうだが)．その間断続的にファイルダウンロードが発生するし，途中でファイルダウンロードに失敗などすればそこで一時中断となるため，インターネット回線が安定した場所でゆっくり腰を落ち着けて作業できるタイミングで実施したい]が，まぁ待っていれば終わる．
ちなみにこのときログを見ていればわかるが，vcpkgのディレクトリに最低限のmsys2環境を構築しているようだ．
vcpkgが扱っているのとは別に普段遣いのmsys2環境を持っている側からすると中々もにょる挙動だが，まぁ確かにmsys2があればpkg-configが動くのも納得である．

##### pkg-conigの有効化

記事に記載のとおりにパスを見つけてくる．

#### CMakeListsの設定

ユーザー固有の設定である `pkg-config` のパスを `CMakeLists.txt` に書くのはだいぶ良くなくて^[バージョン管理システムの管理対象のファイルにユーザー毎に固有の変更が入ると一生差分扱いで出続ける]，こういうのは `CMakeUserPresets.json` に書いたほうが良さそう．
それ以外は概ねコピペできる内容なので，適宜コピペしていく．
「[コンソール窓を開かせない方法](https://qiita.com/exli3141/items/17f6682916909e13914d#%E3%82%B3%E3%83%B3%E3%82%BD%E3%83%BC%E3%83%AB%E7%AA%93%E3%82%92%E9%96%8B%E3%81%8B%E3%81%9B%E3%81%AA%E3%81%84%E6%96%B9%E6%B3%95)」の内容も同時に適用する．

#### スケルトンのソースを構築。

> プロジェクトを一通り完成させたのちにビルド

`CMakePresets.json` を使った場合のビルド方法は，

```console
> cmake --preset <ここに `CMakeUserPresets.json` で指定したプリセット名>
> cmake --build out/build/<ここに `CMakeUserPresets.json` で指定したプリセット名>
```

である．
無事ビルドに成功し，実行できればOK．
はろーわーく．

#### まとめ

というわけで，記事に記載の内容について追試を行い，多少の変更は必要ながらも実行できるところまで持っていくことができました．
再掲しますが，動くリポジトリは[ここ](https://github.com/wx257osn2/cxx_adc_2023_day1_example)．
`CMakePresets.json` とか知らんかったので勉強になりましたね．
まともな記述が公式ドキュメントしか無かったので軽く触ろうとするにはややヘビーでしたが…

vcpkgの感想ですが，うーん…いや，思想としては正しいと思うんですよね．
同じCMake integratableなC++パッケージマネージャとして[Conan](https://conan.io/)と比較すると，Conanはprebuiltパッケージが基本なのに対して，vcpkgはすべてをローカルでビルドします．
この挙動自体に私は賛成です．
ですが，あまりにも依存解決の時間が長すぎる．
どうもパッケージの解決を1個ずつやってるような挙動に見えます．
ダウンロード，ビルド，ダウンロード，ビルド…という流れを延々繰り返す．
これが非常に長い．1時間待ちになったのには流石にびっくりしました．
Cargoってたぶんこの辺を並列にやってるはずで，まぁ色々前提が違うという話はあれど，まともなパッケージマネージャ体験，Cargoってやっぱすごいんだなって再認識しました^[わからん，vcpkgもなんかすれば並列に動くかもしれん．いやデフォルトでやってくれ]．

とはいえvcpkgも中々えらい．
Windowsでpkg-configが必要ならなんとかpkg-configに依存せずに扱えるようにするみたいな方法ではなく，Windows上でなんとかpkg-configを扱えるようにする，というのは一見脳筋にも見えるけど，pkg-configに依存する任意のパッケージに直接対応できるわけで，現実的でもあります．
またvcpkgはマルチプラットフォームですから，pkg-configが素直に使える環境では使ったほうが良いという話もあります．
そうした判断の結果として，現在ちゃんと使えるパッケージマネージャになっているのは素直にすごいことです．

vcpkgは(パッケージのマニフェストが)中央集権ですが，追加のリクエスト自体は `microsoft/vcpkg` へのPRとして普通にユーザーでもできます^[知らんうちに[自作ライブラリが登録されてました](https://github.com/microsoft/vcpkg/pull/33102)]．
みなさんもパッケージマネージャに興味が湧いたらこういった形から貢献してみると良いかもしれません^[Cabinもはよこういう形に持っていきたい]．

## Day2 「[レガシーAPIとstd::unique_ptr](https://qiita.com/yohhoy/items/c186bbf8fdf36ba9e984)」

2日目， `std::out_ptr` の話．
記事内の「レガシーAPI」，身近なところで行くとCUDAなんかが該当します．

> [`__host__ ​ __device__ ​cudaError_t cudaMalloc ( void** devPtr, size_t size )`](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__MEMORY.html#group__CUDART__MEMORY_1g37d37965bfb4803b6d4e59ff26856356)

大体この手のは `template<typename T> std::expected<std::unique_ptr<T, cuda::deleter>, std::error_code> cuda::malloc(std::size_t)` みたいなラップをしがちなのでユーザーコードで直接的な恩恵を受けるわけではないようにも思いますが，ライブラリコードがまともになるのは素直にうれしいです．
というかライブラリコードを用意しないといけないのがこれまで `std::out_ptr` が無かったからみたいなところもあり，これからはユーザーコードで直に扱っても大きくは困らないかも．

## Day3 「[Thrustの非同期実行](https://qiita.com/sukeya/items/1b5998764b9326973644)」

3日目，Thrustでデバイス上同期処理をいい感じにやる記事．
実は9/16に読んでいたのですが，その2週間後に[最新版](https://sukeya.github.io/articles/thrust-async/)が公開されていたようなので…一応最新版に沿って話をしましょう^[読み直した]．

> Thrustには `event` と `future` があります

一見「何故？」という気持ちになりますが，おそらく `event` はCUDAのeventをラップしたもので， `future` は `std::future` と同等のそれな気がします．
Thrustちゃんと使ってないので詳しくないですが， `reduce` は処理結果をホストに返す気がします(これはある種当然で，Thrustは基本的にデバイス上の処理をホストから(ある程度)隠蔽する類のライブラリですし，1要素ならデバイスでさらに使うとしても引数で再度値渡しすればいいし，デバイス上の1要素のアドレスを返されても困るので)．
それ故ホスト側で同期を取る標準的なインターフェースに合わせこんでいる．
一方デバイス側の同期処理は `std::future` に合わせこむのはちょっと無理があります．
その時点でホストに返すもの `T` が無いからです．
こうした理由から同期機構が2種類載っているものと考えられます．

また，上記の推測が正しければ `host_event` が存在しないのはラップすべきeventが無いので当然…かとも思いましたが，

> `thrust::when_all` 関数で複数の `event` と `future` を一つの `event` にまとめることもできます。

なので， `future` を `event` にできるならなんかどうにかならんか？という気持ちになってきました．
いやどっちみちoneTBB向けに非同期APIが生えてないとかっぽいので一朝一夕にどうにかできるもんでもない気がしますが…

> Thrustのリポジトリをご覧になるとわかるのですが、なんとアーカイブにされてます。

えっ．知らんかった…

---

というわけで[置いてあるコード](https://github.com/sukeya/ThrustAsyncProgram)をビルドして動かしてみると…ビルドが通りません．
直して動かしてみると…動きません．
といったあたりを直したのが[こちら](https://github.com/wx257osn2/sukeya-ThrustAsyncProgram)に，PRは[これ](https://github.com/sukeya/ThrustAsyncProgram/pull/1)になります．

## Day4 「[「//このコメントを消したら動かない」は大体Shift_JISの2バイト目が原因で発生する](https://qiita.com/shirokuma89dev/items/1e3a01aa8071507850cd)」

4日目，コンパイラのソース文字セットがShift-JISないしCP932でないときに当該文字コードのファイルを食わせると場合によって行末バックスラッシュ扱いになって論理行結合が起こり，これによって行コメントが継続してプログラムが意図と異なる挙動をする話．
そうはいってもCP932使ってるのなんて日本語Windows環境ぐらいだろうし，Windows環境ではソース文字セットをCP932にできるコンパイラが大半なのでそんな困らなくない…？という気持ちもある．特にLinuxとかMacの環境でCP932触ること，ある…？まぁあったんやろな…^[教授が授業用にWindowsで書いたコードを配布した，とか．いやそれなら大人しくWindowsで扱いなさいよという気もしないでもない(逆に授業としてLinuxやMacを想定してそれなら教授側が悪いが)]

> そんな問題何を今更→UTF-8が出てから生まれたからです。

実はUTF-8初出が1993年で，このくらいの生まれだとまだこの問題に出くわした経験のある人はいそうな気がする，などと野暮なツッコミを入れてみる(マジレスするとこれは「UTF-8が十分に普及して後にこの世界に飛び込んだ」ぐらいの意味合いでしょう．…そら私も歳取るわな…(遠い目))．

> Shift_JISさん悪く言ってごめんなさい。確かにバイト数が少なく済むのはいいと思うのでがんばってね

EUC-JPっていうのがあってぇ…(インターネット老人会)^[日本語が大体2バイトで表現できるのでバイト数ケチりたいだけならこっちで良くない？の意．いや半角カナ多用するとかだとまぁShift-JISの方が縮むけど…(どちらかというとShift-JIS(というかCP932)が使われてるのは日本語Windowsでのサポート度合いの意味合いがつよいと思うのですよね)]

> Shift_JISやめましょう

それはそう．現代なら大人しくUTF-8を使うのが良いでしょう．
だがUTF-8 with BOM，テメーはダメだ．というかUTF-8にbyte orderとかいう概念を生やすな．

ところでUTF-8はあんなにまともな体系だったのにどうして[こんなクソ仕様入れちゃった](https://ufcpp.net/blog/2021/12/regional-indicator/)んですかねぇ… 

## Day5 「[C++ をclang で解析するときに情報をvcxproj から取得する方法](https://uyamae.hatenablog.com/entry/cppadventcalendar2023)」

5日目，vcxprojをいい感じに解析する方法．
msbuild，こんな感じで拡張できるんですね．精々が `UserMacro` 止まりだったので知りませんでした．あとはC++/CLIで書けばより良い記事になりましたね～，などと思いながら徐ろにVS2022を起動，ガチャガチャやったところ…

```
'native,Version=v0.0' を対象とするプロジェクト 'ClassLibrary1' に関して、パッケージ 'Microsoft.Build.Utilities.Core.17.12.6' の依存関係情報の収集を試行しています
依存関係情報の収集に 623 ミリ秒 かかりました
DependencyBehavior 'Lowest' でパッケージ 'Microsoft.Build.Utilities.Core.17.12.6' の依存関係の解決を試行しています
依存関係情報の解決に 0 ミリ秒 かかりました
パッケージ 'Microsoft.Build.Utilities.Core.17.12.6' をインストールするアクションを解決しています
パッケージ 'Microsoft.Build.Utilities.Core.17.12.6' をインストールするアクションが解決されました
インストールに失敗しました。ロールバックします...
パッケージ 'Microsoft.Build.Utilities.Core.17.12.6' はプロジェクト 'ClassLibrary1' に存在しません
パッケージ 'Microsoft.Build.Utilities.Core.17.12.6' はフォルダー 'R:\ClassLibrary1\packages' に存在しません
NuGet の操作の実行に 686 ミリ秒 かかりました
パッケージ 'Microsoft.Build.Utilities.Core 17.12.6' をインストールできませんでした。このパッケージを 'native,Version=v0.0' を対象とするプロジェクトにインストールしようとしていますが、そのフレームワークと互換性があるアセンブリ参照またはコンテンツ ファイルがパッケージに含まれていません。詳細については、パッケージの作成者に問い合わせてください。
経過した時間: 00:00:03.3114763
========== 終了 ==========
```

`Microsoft.Build.Utilities.Core` ，C++向けには提供されてないらしく，NuGetで持ってこようとするとコケます．ぐぬぬ．

本題からは外れますが， `Directory.Build.props` が自動で読み込まれる話，機能的に欲しかったけど調べてもわからなかったので泣く泣く普通のproperty sheetを手動でimportしてなんとかしたやつなのでもっと早く知っときたかった…(🤖<GWぐらいで読んどけば使えたのに)

## Day6 「[VSCode + MinGW-w64 via MSYS2 で WIN32API の Unicode ビルド](https://aquasoftware.net/blog/?p=1982)」

6日目，MinGW-w64でUnicode(非マルチバイト)設定のWinAPIコードをビルドする話．
MinGW特有の(Windows特有の話のg++向け)オプションは他にも `-mbig-obj` などがありますね．
これはMSVCで言うところの[`/bigobj`](https://learn.microsoft.com/ja-jp/cpp/build/reference/bigobj-increase-number-of-sections-in-dot-obj-file?view=msvc-170)に相当します．

> [テンプレート ライブラリを多用するコードでは、より多くのセクションを保持できる .obj ファイルが必要になる可能性があります](https://learn.microsoft.com/ja-jp/cpp/build/reference/bigobj-increase-number-of-sections-in-dot-obj-file?view=msvc-170#remarks)

ありました．Boost.Spirit.X3でパーサー書いたときに．

## Day7 「[Deducing thisで子クラスにも繋がるメソッドチェーン](https://qiita.com/tyanmahou/items/2e9a7350f52e2da18043)」

7日目，deducing thisを使えば子クラスの型を取れるのでメソッドチェーンのために自身の参照を返す際に適切な参照を返せて親クラスのメンバ関数から子クラスのメンバ関数を繋げられる話．
うーん，天才．[deducing thisは絶対なんか面白いことに使えると思っていました](https://x.com/wx257osn2/status/1600368823835975683)が，実用的な例を自力で思いつけなかったのでたすかります．

関連してその他のdeducing this有用情報として，[ラムダ式で使うと自己再帰ラムダがいい感じに書ける](https://youtu.be/EIqEEZywdi4?t=884)話なども2023年のトピックでした．

## Day8 「[浮動小数点数と整数の変換](https://qiita.com/eli_/items/7f305eb022eead1329f8)」

8日目，浮動小数点数型から整数型に `static_cast` する際，整数型側で表現不可能な入力を与えるとUBになる問題について．
UBを避けるためには注意深くNaNを避けたりsaturationしたりする必要があります．
一方RustはLLVMにいい感じにsaturationしたりする命令をぶち込んで組み込み型変換をsafeにした．

んー，どうも浮動小数点数周りは闇深ですねぇ…そういえば浮動小数点数周りで最近おもしろい話を聞いたのですが…まぁこれは別の機会にしましょう．

> 今年の6 月にTokyo C++ Meetupで会った方にアドベントカレンダーイベントを紹介してもらいました

私だこれ(ご参加ありがとうございました！)

## Day9 「[大学の C++ 講義で使っているオンラインコンパイラ](https://zenn.dev/reputeless/articles/cpp-online-compiler)」

9日目，オンラインコンパイラの紹介．
昔はideoneしかなかったのに現代は便利になったのぉ(老人並みの感想)
実際環境構築は最新の状況が移ろうので書籍で学びにくい・追従しないと情報が陳腐化する・頻繁にやらないので忘れがち，と難しい要素が多い割に，本質的に学びたいことのために必要な事項だがそれそのものではないのでしんどさがあり，初心者つまづきがちポイントの1つです^[rustupみたいなのがあればよかったのですが…まぁC++も最新機能を使いたいみたいなこだわりがなければ一応OS標準のパッケージマネージャで何かしらは入れられるだろうからいいか．…え？Windows？]．
オンラインコンパイラはそうした点を避けてプログラミング・プログラミング言語の学習そのものに取り組むことができるので，非常に良い…と思う一方で，現代でC++を使う用途の1つとしてスクリプト言語の遅い部分をC++実装してbindingする，みたいなのがあると思っていて，こうしたことをやろうとすると真面目に取り組まざるを得ないのですよね…

## Day10 「[EnTTを使って大量のオブジェクトを扱う](https://qiita.com/naritancorp/items/0377004000388ea695da)」

10日目，ECSの紹介とその一実装である[EnTT](https://github.com/skypjack/entt)によるサンプル．
現代だとコンポーネントベースな設計とかいうのが流行っているらしく，その上でECSがあるっぽいのですが…コンポーネントベースな設計でどうやってゲームを作るのか全くわからん．
私が想像するゲームのコードって(無理にコンポーネントに寄せるなら)こんな感じ:

```cpp:昔懐かしタスクシステム的な
struct TransformComponent {
	float _x, _y, _z;

	TransformComponent(float x = 0, float y = 0, float z = 0) : _x(x), _y(y), _z(z) {}

	void update() {
		_x += 1.0f;
		_y += 1.0f;
		_z += 1.0f;
	}
};
struct AnotherComponent;

struct GameObject {
	virtual void update() = 0;
	virtual ~GameObject() = default;
};

struct SomeObject : GameObject {
	TransformComponent transform;
	AnotherComponent another;
public:
	SomeObject(const TransformComponent& t, const AnotherComponent& a) : transform{t}, another{a}{}
	SomeObject(const SomeObject&) = default;
	SomeObject(SomeObject&&) = default;

	void update() override {
		transform.update();
		another.update();
		// 他にオブジェクト固有の処理があるならここに書く
	}
};
struct OtherObject : GameObject { /* ... */ };
struct AnotherObject : GameObject { /* ... */ };

std::vector<std::shared_ptr<GameObject>> objs;  // ← こいつにいろんなオブジェクトを突っ込む

```

(現時点の私にとって)重要なのは「オブジェクトそれぞれが自前の `update` を持っている」ことです．
細々したデータと処理をcomponentに切り出すのはまぁいいんですけど^[処理が共通化できるので/それはそれとして無理に寄せてこうしましたが私がゲーム作ってた頃はこういうことはしていませんでしたね… `SomeObject` が `TransformComponent` と `VelocityComponent` のメンバ変数相当を直に持って自分の `update()` で直接操作して…みたいな感じだった]，それを汎用の `GameObject` に持たせても…どうしようもなくない？という疑問が払拭できません．
例えば全く同じデータメンバを持つプレイヤーのオブジェクトと敵のオブジェクトの処理ってどうやって書き分けるんでしょう？
もしかして他のcomponentをガチャガチャやるデータメンバ持たないcomponentでも用意するのか…？無駄が多すぎる気がするが…(本当にわからない)

さて，その上でECSなのですが，ECSの利点を大きく2つに分けると

1. 設計の簡素化
    - 継承は管理しきれなくなるのでそれ以外の設計を導入したい
    - データと振る舞いも分離したい
1. 実行性能の向上
    - ECSに実装を隠蔽すると比較的容易にAoSからSoAへの移行が可能
        - 同じ型のオブジェクト群についてメモリ連続性が確保できるのでキャッシュヒット率が向上する

に集約されそうです．
前者は上記の通り前提の時点でなーんにもわからなかったので，後者についてもう少し真面目に考えてみます．

上記の私のコードだと，最終的に `GameObject` が収まるのは `std::vector<std::shared_ptr<GameObject>>` でした．
したがって，データの実体はメモリ連続な領域に配置されず，オブジェクト毎に飛び飛びの領域に格納されることが予想されます．
これは記事内のコンポーネントベースな設計だとより顕著で， `std::vector<std::vector<std::shared_ptr<Component>>>` 相当なので各 `GameObject` はメモリ連続ですがその `_components` の中身(`std::shared_ptr<Component>` が置いてあるメモリアドレス)の時点でぐちゃぐちゃです．そらオーバーヘッドもデカくなる．
一方，EnTTは内部的には `TransformComponent` の配列を持つことになります．
あとはこれをiterateするだけ．
他のcomponentが増えても，配列をsparseに扱うだけで飽くまで配列走査の形にするようです．
そりゃ速そうではある．が…単にメモリ連続にするだけなら別にコンポーネントベース(というかAoS)でもうまくやれば(やや無理がありますがなんとか)できそうです．

というわけで，記事の後段にある実行時間計測のコードを拡張して計測してみます．
拡張の内容は:

- 時間を標準出力に吐くのではなく関数で返すように
    - 時間の加工がしやすくなるので
- 時間は `float` で持つように
    - ミリ秒は粗いのでもう少し下の桁も欲しい
- warmupの追加
    - 動的メモリ確保などを含む場合に実行時間計測するならとりあえずwarmupしときたさある
    - とりあえず1回空回しする
- medianを取るように変更
    - 一応ブレを抑えたいので
    - debug実行で5回，release実行で11回回して中央値を取る
- 実装の追加
    - 上記の「私が考えるゲームのコード」なども比べたい

ということで，リポジトリは[こちら](https://github.com/wx257osn2/cxx_adc_2023_day10_performance_comparison)になります．
記事に記載のあったgame object版とentt版に加えて，以下の3実装を追加してあります:

1. improved gameobject: 元の記事の実装に僅かに手を加えたもの
    - bodyが空の特殊メンバ関数を `default` 指定
    - `push_back` を `emplace_back` にしたり `std::move` を書いて不要なコピーコストを削減
1. static-polymorph game object: 静的多態やメモリ連続性を導入して極力速くしてみたもの
    - 多態: `std::variant` でやる
    - メモリ連続性: `boost::small_vector` でほぼメモリ連続にする
1. old style game object: 上で書いた「Iさんが思うゲームのコードってこうじゃない？」のやつ

まずは手元の環境でのRelease実行の結果を見てみましょう．

```
---
original game object
GameObjectの作成とコンポーネントの追加: 175.251ms
コンポーネントの変更にかかった時間: 14.1389ms
---
entt
エンティティの作成とコンポーネントの追加: 75.3356ms
コンポーネントの変更にかかった時間: 1.471ms
---
improved game object
GameObjectの作成とコンポーネントの追加: 103.287ms
コンポーネントの変更にかかった時間: 13.3253ms
---
static-polymorph game object
GameObjectの作成とコンポーネントの追加: 39.3652ms
コンポーネントの変更にかかった時間: 3.3549ms
---
old style game object
GameObjectの作成とコンポーネントの追加: 47.3202ms
コンポーネントの変更にかかった時間: 7.9202ms
```

大筋記事に記載の時間に近そう…いやenttのコンポーネント作成時間はちょっと遅いか．
ともかく，ここで触れたいのは追加した実装から見える以下2点です:

- improved game objectがoriginalのgame objectに比べてぼちぼち速くなっている
    - 高速な実装を相手取って比較するなら比較対象も十分に高速にするべき
        - 流石にコピーコストの削減くらいはしておきたい
    - > ちゃんと計測するのが大事。

      それはそう．でもいくら計測しても速くする手段を知らないと速くはできないのですよね…
- static-polymorph game objectはそれなりに健闘しているがまだまだ遅い
    - ちなみにこの実装 `std::vector<boost::small_vector<std::variant<TransformComponent>, 1>>` 相当になっているので，サイズとvariantのタグが間に挟まるぐらいである程度は `TransformComponent` がメモリ上近い位置に置いてあるなんですが…やはり分岐も入りますし完全に連続している時ほどの速度にはなりませんね．
        - コンポーネントの種類が増えれば `boost::small_vector` の静的確保サイズが増えるようになってるので，どんどん遅くなりそう．それはそれとして1種類しかcomponentが無いのもECS側に過剰に有利になってそうな気もしないでもない
    - ここから(性能面での)enttの効能は十分見てとれる

EnTT，言うだけあって中々速いですね！
では次に，Debug実行の結果を見てみます．

```
---
original game object
GameObjectの作成とコンポーネントの追加: 3601.27ms
コンポーネントの変更にかかった時間: 105.737ms
---
entt
エンティティの作成とコンポーネントの追加: 4670.88ms
コンポーネントの変更にかかった時間: 135.34ms
---
improved game object
GameObjectの作成とコンポーネントの追加: 2903.64ms
コンポーネントの変更にかかった時間: 96.4827ms
---
static-polymorph game object
GameObjectの作成とコンポーネントの追加: 1345.7ms
コンポーネントの変更にかかった時間: 152.371ms
---
old style game object
GameObjectの作成とコンポーネントの追加: 440.225ms
コンポーネントの変更にかかった時間: 37.221ms
```

記事に記載の通りenttが遅い結果となりました．
そしてそれ以上に遅いのがstatic-polymorph game objectで，最も速いのはold style game objectです．
ここで記事で触れられていた考察を引用します:

> debugでECS版が遅かったのは
> - 最適化されていない状態ではキャッシュヒット率が低下している
> - includeしているentt.hppが大きい
>
> が主な原因だと思われます。(全然違っていたらコメントください)

思うに，だいぶ違うと思います．ということで，何がどう違うのか，何故Debug版がこのようになるのか述べます．

> - 最適化されていない状態ではキャッシュヒット率が低下している

これは全く違うはずです．理由は2つ．

1. enttにビルドプロファイルによってデータ配置を変更する動機が無い
    - 時間がなくてenttの深堀りまでできていないのですが，わざわざDebug版とRelease版でコードを分けてデータ配置を変更する動機は無さそうに思います
    - データキャッシュのヒット率は実際のデータ配置に依存しますから，Releaseと同じデータ配置ならDebug実行でキャッシュヒット率がそこまで極端に落ちるとは考えにくいでしょう
1. ここではキャッシュヒット率は「高速な理由」足り得るが，「低速な理由」足り得ない
    - もちろんRelease buildでは最適化によって消え去るはずのメモリアクセスなどによってキャッシュがある程度汚される可能性は十分にあるのでdebug buildでキャッシュヒット率が落ちている可能性はあります
    - それでもまるででたらめなランダムアクセスを強いられる他の実装と条件はトントンといったところ．他実装より顕著に遅くなる理由足りえません

> - includeしているentt.hppが大きい

これは微妙な話で，直接の原因ではないですが結果的に影響していそう，といったところでしょうか．
当たり前ですが入力ソースが巨大であることを直接の原因として(特にRelease buildの)実行バイナリの性能が悪化するということは(出力バイナリが巨大になって命令キャッシュから溢れるようになる，みたいな話ぐらいが関の山で)原則ありません^[現に今回の例ではRelease buildで高速な実装ほどDebug buildで遅くなっています．単純にコードが多くてバイナリがでかいほど遅くなるならRelease buildでも遅くなるはず]．
ただし，一般に大規模なコードは短いコードに比べて関数がたくさん書いてあったり標準ライブラリをたくさん使うことが多いです．
そうすると，Release buildではinline展開されるはずだった関数のfunction callが残ってしまったり，標準ライブラリ内で呼び出している `assert` の条件分岐などが短いコードより多く含まれえます．
また，単純に入力ソースコードが増えればコンパイラが出力するデバッグ用の(余分な)コード出力も当然増えます．
コードの量が多いから遅いというのは一見無理のある主張に聞こえますが，Debug build特有のオーバーヘッドがよく整頓された大規模なコードほど載りがちなので，Debug buildに限って言えば影響はありそうです．

そして実際に遅くなっている主要因がこのデバッグ情報によるオーバーヘッドとinline展開の阻害によるものです．
EnTTやBoostなどはヘッダオンリーのライブラリなのでコンパイラから見ると一見ユーザープログラムが長大なように見え，その随所にデバッグ情報を仕込んでしまいます．
加えて(特に)Boost.ContainerはDebug build時にはassertやコンパイラ拡張によるデバッグ命令を大量に仕込んであります．
そしてそもそも，EnTTや `boost::small_vector` ， `std::variant` などはinline展開されることを念頭にtemplateパラメータ毎の挙動の差の吸収などに大量の関数を用いて実装されています．
Debug buildでは大量の関数がそのままの形で残っており，呼び出し^[具体的にはrange-based forの度に `begin()` と `end()` が，そしてiterationの度に `operator++()` や `operator!=()`も呼ばれます．場合によってはそれらの関数からさらに他のメンバ関数が呼ばれていくわけです]の度にcall命令が走るので非常に大きなオーバーヘッドになっています^[実際にasmを見るとえらいことになってる．実測と同程度にasmを眺めるのも大事です]．
というわけで，以下の設定変更を加えて再度実行時間を確認してみましょう:

- inline展開などを阻害するのでデバッグ情報の形式をプログラムデータベースに変更(`/Zi`)
- 不要なデバッグ情報の削除
    - プリプロセスマクロに `NDEBUG` を追加
    - コンパイルオプションから `/JMC` と `/RTC1` を削除
- `/Ob2` で関数のinline展開を有効化

```
---
original game object
GameObjectの作成とコンポーネントの追加: 1846.25ms
コンポーネントの変更にかかった時間: 56.7655ms
---
entt
エンティティの作成とコンポーネントの追加: 1043.6ms
コンポーネントの変更にかかった時間: 23.4641ms
---
improved game object
GameObjectの作成とコンポーネントの追加: 1590.36ms
コンポーネントの変更にかかった時間: 59.0381ms
---
static-polymorph game object
GameObjectの作成とコンポーネントの追加: 213.77ms
コンポーネントの変更にかかった時間: 36.9301ms
---
old style game object
GameObjectの作成とコンポーネントの追加: 231.559ms
コンポーネントの変更にかかった時間: 22.9041ms
```

この程度の変更であっという間にgame object実装より速くなりました．
上記のリポジトリでは `DebugNoAssert` というconfigurationで上記の設定が動作するので確認してみてください．

## Day11 「[UBとEB](https://onihusube.hatenablog.com/entry/2023/12/11/000000)」

11日目，新概念EB(Erroneous Behaviour)の紹介．
不正なビットシフトのEB，はよ入ってほしい．
というのも，過去に64bit非負整数型を64bit左シフトしてしまい，GCCとClangで挙動が違くて無事死亡した^[GCCはまぁ見逃してくれることも多いんですがClangは容赦なくoptimizeに使ってくるので結果が意味不明なことになりがち]ため…

ところでCとC++の違いについてなのですが，実はCだと非 `void` 戻り値型の関数であっても呼び出し側がその戻り値を受け取らなければ `return` 文無しで脱出してもvalidなんですよね([N3220](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3220.pdf) `6.9.2.13`)^[当たり前ですがそういうことをするなら大人しく戻り値型を `void` にしておくべきです]．

> Unless otherwise specified, if the `}` that terminates the function body is reached, and the value of the function call is used by the caller, the behavior is undefined.

一方，C++だと(N4950だと[`[stmt.return]/4`](https://timsong-cpp.github.io/cppwp/n4950/stmt.return#4)など)に記載の通り

1. (cv修飾外すと) `void` を返す関数
    - 暗黙の `return;` が書いてあると見做される
1. コンストラクタ
    - 暗黙の `return;` が書いてあると見做される
1. デストラクタ
    - 暗黙の `return;` が書いてあると見做される
1. `main` 関数 ([`[basic.start.main]/5`](https://timsong-cpp.github.io/cppwp/n4950/basic.start.main#5))
    - 暗黙の `return 0;` が書いてあると見做される
1. promise型に `return_void` が定義されているコルーチン ([`[stmt.return.coroutine]/3`](https://timsong-cpp.github.io/cppwp/n4950/stmt.return.coroutine#3))
    - 暗黙の `co_return;` が書いてあると見做される

の5つを除く関数・コルーチンで `return` 文が無い場合，定義した時点でUBです．
というわけで， `return` 文忘れ周りのEB化も提案されているようですが，個人的には(代入演算子はさておき一般の関数については)大人しくill-formedで良いのではないかなぁと思っています．検出も簡単だしね．

## Day12 「[範囲for文の歴史](https://qiita.com/tetsurom/items/47a4a7af7a4ffa66d243)」

12日目，範囲for文の歴史．

> C++17では、イテレーターの型と異なる型の終端イテレーター(番兵ともいう)を許容するようになりました。

えっ，sentinelの概念ってC++20からじゃなかったの！？
C++17コードではsentinelが使えないと思い込んで頑張って無効なiteratorを定義して `end()` で返していたのに…(おそらくいつものC++標準仕草で，言語機能に入れた変更は次の規格からライブラリに実装され始めるやつで^[まぁ元々議論が紛糾して遅れてたのもありますが]C++20からRange周りでsentinelが多用されたのを見て勘違いしていたやつ)

## Day13 「[メモリ領域の再利用は結構気を付けたほうがよい件](https://qiita.com/43x2/items/b07894b2db77d744e937)」

13日目，生存期間と記憶域期間について．
記憶域の上にオブジェクト生成と破棄を行うので，基本的には記憶域期間の方が生存期間以上^[変数を普通に定義して普通に使う分には記憶域期間と生存期間が一致する]の長さになる，はず…
C++23以降だと `std::construct_at()` と `std::destroy_at()` が出てきたのでplacement newやデストラクタ呼び出しに比べれば幾分か見た目がマシにはなった気がします．
まぁよほどの強い要求が無い限りは原則この手のhackyな手法は使わずに大人しく別の記憶域にオブジェクトを生成する，どうしても必要な場合は適切に抽象化して頻繁にこの手のコードを触らずに済むようにする，あたりが無難なところな気はしますが，ところでメモリが足りないとかキャッシュが足りないとかあーだこーだ言われると…やるしか無いのだよなぁ．

## Day14 「[C++23便利機能の紹介：byteswap関数](https://qiita.com/yohhoy/items/2341166e978aacdf13c6)」

14日目， `std::byteswap` の紹介．
なんかC++20以降エンディアン周りのサポートが急激に向上しましたね．
いまさら？という気もせんでもないですが，まぁついこの間まで実際の数値表現が2の補数かどうか保証無かった言語だし…

`std::endian::native` が `std::endian::little` でも `std::endian::big` でもない値を取ることがある，という話は聞いたことはあったけど，PDP-11，そういう並び方するんだ…そうはならんやろ^[なっとるやろがい！]

## Day15 「[constevalの紹介](https://qiita.com/tyanmahou/items/45229a33e71bd25844e6)」

15日目，constevalの紹介．
`if consteval` とinline asmを併用することで実行時性能を落とすことなくコンパイル時にも呼び出せるconstexpr関数が書けちまうんだ！^[これ自体はC++20の `if(std::is_constant_evaluated())` 時代から書けはしたが]というわけで個人的にはうれしいやつです．

## Day16 「[2023年のコンパイル時レイトレーシング](https://in-neuro.hatenablog.com/entry/2023/12/16/233332)」

16日目，constexprレイトレの話．
[2022年の私の記事](https://zenn.dev/wx257osn2/articles/constexpr_variant-jdbnlwiberawejbf)でざっくり書いた「constexprの進化」を詳述してくれる記事でもあり，C++23時点でのconstexprの強力さを実例で示す記事でもあります．
処理系毎に異なるコンパイル時計算のキャッシュ戦略についてはかの中３女子も頭を悩ませており，Sprout.Darkroom(C++11コンパイル時レイトレーシング)はGCCの方がだいぶ速いみたいな話とか(聞いた覚えがあるが出典が見当たらないので勘違いかもしれない)，逆にSprout.Compost(コンパイル時音声処理)は[GCCだとメモリが結構キツい](https://boleros.hateblo.jp/entry/20121207/1354888090)^[それはそれとして(当時の)GCCは巨大な配列を作れないという別の問題もあり，Clangでしか動かなかった]などの話があったはずです．
C++26の数学関数 `constexpr` 化，当初はC++0xぐらいでlibstdc++は対応してたのに規格側で「処理系によってコンパイルが通ったり通らなかったりするのは良くないので標準で `constexpr` 付けてない関数を勝手に `constexpr` にするのは禁止！」という提案が通ってしまい，処理系側が後から `constexpr` を剥がしたという歴史があり，15年経ってようやく公式に `constexpr` になったのかと感慨深い気持ちです．もしかすると(先述の通り)これはC++20から `constexpr` 関数内でinline asmが書けることで実行時性能を落とすことなくコンパイル時計算ができるようになったことが一因かもしれません^[実際の処理系はコンパイラマジックでどうとでもできるので別にC++11の頃から問題なく `cosntexpr` にできたんだが？]．
また，C++26ではついに明に `void*` を取り回すことができるようになり， `std::variant` のみならず `std::any` や `std::function` に相当するユーティリティすらも `constexpr` に実装できるようになります．

## Day17 「[それって本当にdynamic_castじゃないとダメ？ static_castにして高速化](https://qiita.com/tyanmahou/items/74bd86e766eec10bab7d)」

17日目， `dynamic_cast` を減らす記事．
RTTIを使わずに自前で実行時型情報を構築するのはLLVM全体で使われてますね．
そもそもなんで世間の処理系のRTTIってあんなに重いんでしょうね…？そっちを高速化すれば済む話なような気もしますが…(無知)^[これは例外ハンドラにも言える．例外ハンドラを自前のものと差し替えることで組み込み環境でも十分例外を扱えるようになる，といった話を何処かで小耳に挟んだ覚えが…]

## Day18 「[C++ プログラミングの生産性を少し改善する Visual Studio の機能（2023）](https://zenn.dev/reputeless/articles/cpp-vs2022-tips)」

18日目，MSVCの最新機能の話．
[C++ MIX #9](https://cppmix.connpass.com/event/305699/)でも[このテーマでお話されていました](https://www.youtube.com/watch?v=eP-rm5_Rl7E)ね．
やはり特筆すべきはメモリレイアウトの可視化でしょう．
アライメントに合わせて横幅が変わったり，unionにすると変数が重なったり，かなり芸が細かい．
ただ唯一問題点があって，このアライメント表示を出すためのサイズ・アライメントツールチップ，型名が短いとマウス動かしたらすぐに表示が消えてしまうので1文字型名相手だとほとんどまともに表示できないのですよね…

## Day19 「[C++ コンパイル時パスワード認証　〜コードを不正コンパイルから守ろう！〜](https://qiita.com/Raclamusi/items/89b9b7df0cd61e285a54)」

19日目，コンパイル時パスワード認証の話．
_おまえは何を言っているんだ_ (画像略)

今回は `static_assert` で落としているのであっさり防がれていますが，SFINAEがコケるように仕向けたり，その結果が無いとうまくコンパイルが通らないように何重にも入力パスワードが必要なコードを書けば一応使えなくはなさそうなのがまた…いやもちろんコードが公開されている以上は限界はあるのですけれども．

また，この記事でも `/dev/stdin` を使っているように，CPPはファイル入力が可能です．
ファイルシステムは拡張可能ですから，うまくやればインターネット上のファイルを読んできたりすることもできるわけですね^[実際compiler explorerでは[そのようなことができます](https://godbolt.org/z/AXAe8K)．おそらくこれはCPPとはまた別の前処理を挟んでいそうな気もしますが]．
コンパイル時処理はファイルI/Oができないなどと言われていますが，プリプロセス時処理ならネットワーク通信もできちまうんだ！^[ネットワーク通信をしているのはCPPではなくファイルシステムでは？というツッコミまで含めて昔からよくあるネタ]

## Day20 「[Try to make a try !](https://zenn.dev/wx257osn2/articles/try_to_make_a_try-saiudhleghbaowlb)」

20日目，私です！

## Day21 「[C++erですがCOMに翻弄されています: 再入との戦い](https://qiita.com/yumetodo/items/f961bc96594b9b5619c2)」

21日目，COMの再入の話．
今回の場合は値の書き換えというよりはコンテナへの要素追加・削除が問題のようです．
であるならば，(大人しくMTAにすればいいという話を置いておくなら)私なら補足説明の欄にある通り「削除対象を事前に持っておいて削除は一括一箇所まとめてやる」の方向で進めそうな気がします．
10日目の記事の感想で `std::vector<std::shared_ptr<GameObject>> objs;` のようなコードを私が書きましたが， `objs` 内のオブジェクトが他のオブジェクトを削除する・追加する，といった操作はこのようにしないとiteratorが壊れてループが狂いますからね．

でもたぶん大人しくMTA使ったほうがいいよ(堂々巡り)

## Day22 「[C++ コンパイル時「出力」で画像ファイル生成](https://qiita.com/Raclamusi/items/fd9c5b52c514a6420e2c)」

22日目，コンパイル時出力の話．

> コンパイル時処理はファイルI/Oができないなどと言われていますが，プリプロセス時処理ならネットワーク通信もできちまうんだ！

> _**誰ができないって決めたんですか？**_

何ィ！？

どうもアセンブリに `.print` という命令があり，これを使うとアセンブル中にターミナルに文字列を吐けるらしく，これを使って標準出力にファイルの中身を吐き，それをリダイレクトしてやればコンパイラでもテキストファイルなら吐けるもん！ということらしい．

…プリプロセス時にファイルを読んで，アセンブル時にファイルを吐いているのだからコンパイラは何もしていないのでは？

## Day23 「国際規格断片コンパイル」 → 本稿

23日目は無かったので当記事で代替としました(近場においておいたほうが何かと便利なので)．

kaizen_nagoya先生の[次回作](https://qiita.com/kaizen_nagoya/items/d9c69d79d5ad4aa73a91)にご期待ください．

## Day24 「[std::variantを使ってシミュレータでオートプレイさせる](https://h-sao.com/blog/2023/12/25/aboutsimulatorandusestdvariant/)」

24日目，共通の操作を持つ複数の型のオブジェクトを `std::variant` を使っていい感じにまとめたコードにしたい話．

> `struct GetCharactorId` や `struct ExecuteOrder` を定義する必要があるものの

実は必ずしも必要がなくて，単に `[](auto&& order){return order->ExecuteOrder();}` とか，もうちょっと制約するにしても `[]<typename T>(const std::shared_ptr<T>& order){return order->ExecuteOrder();}` みたいなラムダ式にしてしまえば事足りたりします．
さらにこれらであれば今後技のクラスが増えても関数オブジェクト側は変更する必要がありません^[さも良いことのように書いていますが，あえてそこは手で管理したい！みたいな場合はこれだと困るのでユースケースに依る]．

## Day 25 「[契約プログラミング機能のこれまでとこれから](https://onihusube.hatenablog.com/entry/2023/12/25/220134)」

25日目，契約プログラミングについて．
C++，早く僕と契約を！

契約プログラミングは言語サポートの一段手前に意識的な実践があって，要は言語サポートが無くても事前・事後条件と不変条件を意識してプログラムを書くと安全なコードが書けていいよね，みたいな話があり，私も10年近く前から意識してコードを書いています．
それはそれとして動くドキュメントとしての事前事後不変条件の記述はさっさと入って欲しい言語機能第1位みたいなところがあり…おっC++20に入ったな…なんか消えたな…いつになったら入るんだ…？
というわけでC++20策定ちょっと前くらいからの契約プログラミングの変遷と，執筆時点での最新状況についてのまとめ記事です．

> 属性の置ける位置を再利用するため、後置戻り値の前（ `override` や `requires` 節の前）に事前条件と事後条件がくる

そもそもの話として `requires` 節って後置できたんですね…

> 異なる翻訳単位の最初の宣言における契約について 

これってODRに載せればなんとかなったりせんの？と思ったけど，One Definition RuleであってOne Declaration Ruleではないのだった…

> オーバーライドする関数とされる関数の契約について

これが変えられてしまうとオーバーライドとは…という気持ちになるが，まぁ継承が継承を目的として使われないこともある(というか現代C++における継承って大体mixinとかのための機能だ)しな…

## 最後に

ということで全25-1日分の記事の感想でした．
参加者の皆様，1年遅れとなりましたがご参加いただきありがとうございました！

24日目の記事は[遥佐保(@hr_sao)](https://twitter.com/hr_sao)さんの[std::variantを使ってシミュレータでオートプレイさせる](https://h-sao.com/blog/2023/12/25/aboutsimulatorandusestdvariant/)です．