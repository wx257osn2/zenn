---
title: "poacの現況 + 改造して遊んでみた"
emoji: "🐷"
type: "tech"
topics: ["cpp"]
published: true
---

:::message
この記事は[C++ Advent Calendar 2024](https://qiita.com/advent-calendar/2024/cxx)の20日目の記事です(遅刻)．
19日目は[yumetodo(@yumetodo@misskey.dev)](https://misskey.dev/@yumetodo)さんの「[それはCOM STAと並列処理の三体問題との戦いだった ～Optimal Biz Teleworkの機能をOptimal Bizに部分取り込みする～](https://tech-blog.optim.co.jp/entry/2024/12/19/100000)」でした．
:::

[I](https://twitter.com/wx257osn2)です．C++ Advent Calendar 2024，今年はそこそこ状況を見ていたのですが，なんやかんやありつつも(私以外は概ね)ちゃんと記事が公開されてていい感じでしたね．主催が率先して遅刻するのをやめろ．
リアルタイムで感想をツイートすることは今年も叶いませんでしたが，一応今までの記事は一通り目は通しているので感想は年末ぐらいまでには書く予定です．

# poacの現況 + 改造して遊んでみた

C++にはRustにおけるCargoのようなまともなビルドツール兼パッケージマネージャが無いことは皆様周知の事実かと思います[^一応パッケージマネージャだけで言えばConanとかCMake CPMとかもありますが，CMakeに依存してるのがな… / Hunterって生きてるんですか？ / BuckarooはBuckのセットアップが地獄だし(Buck 2はRust製になったそうだが1はJavaとPythonを両方要求される)開発止まってそうだし… / vcpkgは依存解決の時間がクソ遅い，という話は後日しますが…]が，その方向で頑張ってる[^全然WIPなので今すぐproduction ready！みたいな感じではない]プロジェクトとして[poac](https://github.com/poac-dev/poac)があります[^ちなみに読みは「ぱっく」とか「ぽっく」に近く(/pəʊək/)，「ポーク🐷」ではない]．
本稿ではpoacの概要と現状について **私の主観全開で**[^main authorにインタビューなどをしたわけではないので，憶測などを多分に含む．最終的にテーマをpoacに決めたのが20日当日だったこともあり，ちゃんとしたインタビューとかする時間無かった]軽くまとめ，ついでに少し改造して遊んでみようと思います．

## poacとは

poacとは[情報処理推進機構](https://www.ipa.go.jp/)が主催している[未踏IT人材発掘・育成事業](https://www.ipa.go.jp/jinzai/mitou/it/about.html)の[2018年度採択プロジェクトに端を発する](https://www.ipa.go.jp/jinzai/mitou/it/2018/gaiyou_t-2.html)松井健さんの個人開発プロジェクトで，ものすごく乱雑に言えばC++版のCargoを作ろうとするものです[^実際にはCargoと全く同一のものを目指しているわけではないと思いますが，少なくともCLIのUI/UXについては相当Cargoを参考にしてると思う]．
個人開発故，松井さんのライフステージに応じて開発が止まっていた時期などもあり，開発開始から6年程度経とうとしていますが今のところはまだWIPです．

実は私も去年の4月頃までは暇なときにコントリビューションをしていたのですが，なかなか時間も取れず1年半くらいはメールで通知を眺めている程度の感じでした．
開発自体も昨年春から冬くらいまで半年程度停滞していたような印象です．
昨年末辺りから開発が最活発化し，今年1年は(コントリビューターからのPRも含めて)結構活発に開発されていたように思います．
どうもその過程で[大規模に作り直した](https://github.com/poac-dev/poac/pull/799)ようで，以前あった機能が無くなってたり依存関係が変わったりしたようです．

という感じで，メールの中身はそこまで読まず通知の頻度と誰がコントリビューションしてるかくらいの薄ぼんやりしたウォッチを続けていた[^メールボックスの未読を消すタイミングでタイトルと送信者だけを見ていた]ため実態はあまり把握できておらず，久方ぶりに触ってみたらコードは様変わりしてるしCMake[^poac自体のbootstrapビルドに使われていたはず]もBoost[^作り直しの過程で独自パッケージサーバーに対するサポートを一度オミットしたっぽくBoost.Beastが不要になったのと，まともなC++20コンパイラが増えたのが理由？]もninja[^以前はpoacがninjaのAPIを呼び出すような作りだった．現在は `Makefile` 吐いて `make` コマンドをspawnする感じ， `make` に移行するに至った理由とかはよく知らない]も要らないしで結構びっくりしたのですが，ともかく「なんか遊べそう」という雰囲気を感じたので久々に触ってみました．

## poacのセットアップ

`Dockerfile` があるので中を見ればprerequisitesがわかります:

https://github.com/poac-dev/poac/blob/main/Dockerfile

ということで上記の手順で環境構築をするもよし， `Dockerfile` 自体をそのまま使うもよし．
私は最近この手の環境構築には専ら[Singularity](https://github.com/sylabs/singularity)を用いているので，適当に定義ファイルを書いて環境構築をしました．
以前と違って依存関係がだいぶ減ったので環境構築は楽な気がします．
prerequisitesが一通りインストールできたらリポジトリをクローンして `make RELEASE=1 -j$(nproc)` するだけです．
PATHの通ったところにインストールしたければ `make RELEASE=1 PREFIX=/path/to install` するとよいでしょう．
インストールしたくない場合は `poac/build-out/poac` に実行バイナリがあるので直接呼びましょう．
以下では基本的にインストール済みであることを前提とします．

### セルフホスト

poacは `poac.toml` を持つので，一度バイナリをビルドすればpoac自体もpoacでビルドできます:

```console
$ pwd
/path/to/poac
$ build-out/poac build --release
 Compiling poac v0.10.1 (/path/to/poac)
/path/to/poac/src/Manifest.cc: In function 'Profile getProfile(std::optional<std::__cxx11::basic_string<char> >)':
/path/to/poac/src/Manifest.cc:354:15: warning: possibly dangling reference to a temporary [-Wdangling-reference]
  354 |   const auto& table = toml::find<toml::table>(manifest.data.value(), "profile");
      |               ^~~~~
/path/to/poac/src/Manifest.cc:354:46: note: the temporary was destroyed at the end of the full expression 'toml::find<std::unordered_map<std::__cxx11::basic_string<char>, basic_value<type_config>, std::hash<std::__cxx11::basic_string<char> >, std::equal_to<std::__cxx11::basic_string<char> >, std::allocator<std::pair<const std::__cxx11::basic_string<char>, basic_value<type_config> > > >, type_config>((* & manifest.Manifest::data.std::optional<toml::basic_value<toml::type_config> >::value()), std::__cxx11::basic_string<char>(((const char*)"profile"), std::allocator<char>()))'
  354 |   const auto& table = toml::find<toml::table>(manifest.data.value(), "profile");
      |                       ~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
lto-wrapper: warning: using serial compilation of 17 LTRANS jobs
lto-wrapper: note: see the '-flto' option documentation for more information
  Finished `release` profile [optimized] target(s) in 14.98s
$ ls poac-out/release/poac
poac-out/release/poac
```

というわけで， `poac.toml` の書き方などはある程度poacの持つものを参考にすれば良さそうです．

## 現状のpoacのCLI操作

### アプリケーションを作る

適当なディレクトリで `poac new repo_name` とすると `repo_name` というディレクトリが掘られてその下にいくつかのファイルが現れます:

```console
$ pwd
/path/to
$ poac new repo_name
   Created binary (application) `repo_name` package
$ cd repo_name
$ tree -a
.
├── .git
│   ├── HEAD
│   ├── config
│   ├── description
│   ├── hooks
│   │   └── README.sample
│   ├── info
│   │   └── exclude
│   ├── objects
│   │   ├── info
│   │   └── pack
│   └── refs
│       ├── heads
│       └── tags
├── .gitignore
├── poac.toml
└── src
    └── lib.cc

11 directories, 8 files
```

ビルドしてみましょう:

```console
$ poac build
 Compiling repo_name v0.1.0 (/path/to/repo_name)
  Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.27s
$ poac-out/debug/repo_name
Hello, world!
$ cat src/main.cc
#include <iostream>

int main() {
  std::cout << "Hello, world!" << std::endl;
  return 0;
}
```

### 依存関係の追加

では次にパッケージマネージャの重要な機能である，依存関係を追加してみましょう．

```console
$ cat poac.toml
[package]
name = "repo_name"
version = "0.1.0"
authors = [""]
edition = "20"
$ ../poac/poac-out/release/poac add --sys --version ">=9" fmt
     Added to the poac.toml
$ cat poac.toml
[package]
name = "repo_name"
version = "0.1.0"
authors = [""]
edition = "20"

[dependencies]
fmt = {system = true, version = ">=9"}

$ # ついでに src/main.cc も変更
$ cat src/main.cc
#include <fmt/core.h>

int main() {
  fmt::print("Hello, fmt!\n");
}
$ ../poac/poac-out/release/poac build
 Compiling repo_name v0.1.0 (/path/to/repo_name)
  Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.28s
$ poac-out/debug/repo_name
Hello, fmt!
```

上記では `poac add` を使いましたが，直接 `poac.toml` を編集しても問題ありません．
というか私は専らそのようにしています．

現在poacのサポートする依存の種類は2種類あります．

1. system dependency
    - `pkg-conf` が拾ってこれるライブラリを指定します
    - semverによるバージョン指定が可能
    - `libraryname = {version = "semverの条件，比較演算子や&&などで記述", system = true}`
1. git dependency
    - gitで拾ってこれるライブラリを指定します
    - コミットハッシュやブランチ名・タグ名による指定が可能
    - リポジトリは `$XDG_CACHE_HOME/poac/git/src` ないし `$HOME/.cache/poac/git/src` にクローンされて，そこを参照する
    - 現時点ではヘッダオンリーライブラリのみをサポート
    - `libraryname = {git = "gitのURL", hash = "SHA256コミットハッシュ"}` など

ということで，とりあえずシステムインストールなライブラリとgitにあるヘッダーオンリーライブラリの組み合わせであれば一応使うことができそうです．

### 静的ライブラリを作る

`src/main.cc` ではなく `src/lib.cc` が存在する場合，静的ライブラリとしてコンパイルされる機能が[つい先日実装されました](https://github.com/poac-dev/poac/pull/1035)．

```console
$ mv src/main.cc src/lib.cc
$ # 適当にlib.ccを編集
$ cat src/lib.cc
#include <fmt/core.h>

void hello_fmt() {
  fmt::print("Hello, fmt!\n");
}
$ poac build
 Compiling librepo_name.a v0.1.0 (/path/to/repo_name)
  Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.27s
$ ls poac-out/debug/librepo_name.a
poac-out/debug/librepo_name.a
```

ちなみに余談ですが， `poac add --lib package_name` すると `include` のみのディレクトリができてしまい， `src/lib.cc` が生成されません．これは上述のように従来poacは静的ライブラリのサポートが一切無く，ヘッダオンリーライブラリを作ろうとする挙動の名残です．

### その他

`clang-format` や `clang-tidy` ， `cpplint` を予めインストールしておけば `poac fmt` や `poac tidy` ， `poac lint` といったコマンドでformatやlintが実行されます．
`clang-format` や `clang-tidy` はリポジトリ内の設定ファイル(`.clang-format` , `.clang-tidy`)を， `cpplint` は `poac.toml` 内の `[lint.cpplint]` テーブルの `filters` 配列でオプションを並べることで設定できます．

また， `[profile]` テーブルではdebug/release共通の， `[profile.dev]` や `[profile.release]` では個別の設定を行うことができます:

```toml
[profile]
cxxflags = ["-Wall", "-Wextra", "-pedantic-errors"]

[profile.dev]
cxxflags = ["-Og"]

[profile.release]
lto = true
```

ちなみに `debug = true` とか `opt_level = 3` とかの指定ができますが，[記事執筆時点ではこれらは実際には考慮されないので注意](https://github.com/poac-dev/poac/blob/2e0da2cb3aa70ba004fb3a5b2f8f8b0a4fd4ecbb/src/BuildConfig.cc#L605-L616)．

## 改造する

このように，現状のpoac，それっぽく振る舞いそうな部分とそうでない部分(単純に未実装なものから変更の影響で壊れた部分まで)が混在しています．
その中でも，特に以下の2つの機能が不足していると感じました:

1. ローカルパッケージへの依存
    - 現状ではシステムインストールかgitリポジトリへの依存しか張れませんが，実用的には他のローカルディレクトリに置いてあるパッケージへの依存も作りたくなるはずです．最終的な形はさておき，ソフトウェアの開発初期の段階では依存の登録のために毎回リモートリポジトリを作るような運用はちょっと手間が多いです．
1. 依存に対する再帰的ビルド
    - 先日入ったばかりの機能なので致し方ない面もありますが，現状静的ライブラリのビルドに対応したのにgit dependencyはヘッダーオンリーライブラリしか対応していないので，poacで静的ライブラリのパッケージを作ってgitに公開して利用する側のプロジェクトで依存関係を設定しても正しくリンクされません．ちゃんと依存先をビルドして適切にライブラリがリンクされるとパッケージマネージャとしてグッとそれっぽくなります．

というわけで，これらの機能を追加すべくpoacを改造してみましょう．
尤も，ローカルパッケージについては実装が比較的簡易な割に必要性の高い機能だったため，[既に公式リポジトリにPRを出してあります](https://github.com/poac-dev/poac/pull/1048)．こちらはマージ待ちとなります．
再帰的ビルドは真面目に組むと大変なので，今回は雑に「gitかローカルの静的ライブラリパッケージならそのディレクトリで `poac build` して `poac-out` 以下のライブラリファイルをリンクする」みたいな作りにしてしまいました．遊びたいだけだしね．

というわけで改造したものが[こちら](https://github.com/wx257osn2/poac/tree/cxx_adc_2024)になります．
差分は[これ](https://github.com/wx257osn2/poac/commit/82ad3ee9c3cea92ebd449142683c0f4df6f1e0e9)．
変更の内容をざっくり解説すると，依存解決は `poac build` のようなpoacのコマンドをトリガーとして行われ，同じコマンドを依存先で発行してやれば大体似たような構成のビルドが起こるだろう，くらいの雑な考えで， `src` 以下に `lib.cc` があって `poac.toml` を持つgit or ローカルパッケージが依存に書いてあったらそのディレクトリに移動して `main` 関数の `argv` をそのまま実行する，という感じです．
[再帰的なビルドのサンプルも用意した](https://github.com/wx257osn2/cxx_adc_2024_day20_poac-demo)ので，試してみてください．

さて，この改造一見良さそうにも見えますが，普通に多数の問題を抱えています．
まずこの改造はローカルディレクトリ上にすべてのビルド生成物を配置してlockファイルでちゃんと管理，みたいなことは全然していないことに注意が必要です．現在poacのgit dependencyは常にキャッシュディレクトリにのみ展開する仕様のため[^まぁヘッダオンリーだという前提を置くとチェックアウトだけで済むので割と妥当な選択ではある]，この改造はそのままキャッシュディレクトリ上に移動してビルドしてしまいます．ローカルでも同様なので，依存先に `poac-out` ディレクトリが掘られてしまいます．
さらに `clang-tidy` が「実際にビルドしてみてコンパイラが掴んだwarningを出す」という作りをしているせいで， `poac tidy` すると依存解決を行ってしまいます．上述の通り依存解決のタイミングで同じコマンドを発行するので，依存関係も含めて全部 `clang-tidy` がかかってしまい，「依存先はあんまり真面目に `clang-tidy` かけてないけど自分のプロジェクトぐらいは真面目にかけたい」みたいなときに依存先リポジトリの `clang-tidy` の失敗で自プロジェクトの `poac tidy` がコケるようになるので，これはうれしくないですね．
また，サンプルで言うところの `test_application` と `libtest` が同じgit dependencyを異なるハッシュ値で参照するとき，`test_application` の `poac.toml` の `[dependency]` がgit → `libtest` の順の場合， `test_application` 側で指定したコミットハッシュでgitリポジトリのクローンやビルドが行われたあと， `libtest` 側で指定したコミットハッシュで再度gitリポジトリのビルドが走ったのち `test_application` 本体のビルドが走るため， `test_application` から見ると指定したコミットハッシュとは異なるものとなってしまいます．まぁそもそも全く異なるコミットハッシュの同一ライブラリを要求された時点でうまく解決する術は無いのですが[^コンパイルが通っても本質的にODR-use違反を避ける手段が無い]，それを検知するような仕組みを一切導入していないためどこで壊れるかわからないという問題があります．
他にもそれぞれの依存が要求する同一のsystem dependencyのバージョンの範囲が一切被らないかつ各々だけでみれば満たせる(つまりシステム上に当該system dependencyが2バージョンインストールされていて， `test_application` と `libtest` がそれぞれ別のバージョンを向いている)場合なども同様にリンクエラーやODR-use違反などの問題が発生しそうです．
こうした問題を解決するために，依存関係を全部洗って依存ライブラリのバージョンのすり合わせをするというパッケージマネージャの本分みたいな仕事があるわけですが，一番上で起動したpoacが依存先すべての `poac.toml` の依存を読んでネゴる・ビルドもまとめて扱う，みたいになって実装がとても大変なのですよね…(パッケージマネージャなんだから本来そうしたことをすべきなのですが)

そういうわけで，この雑な改造はPRにするつもりはないのですが，少し遊ぶ程度であればこの程度の差分で再帰的なビルドを実現できました．
複数の開発者による様々なパッケージを組み合わせようとするとこの改造では難しそうですが，こうやって設定ファイルに従ってリポジトリを集めてビルドしてくれるとちょっとした個人開発くらいには使えそうですね．

## 感想とまとめ

というわけで，ここ1年以上開発に参加してなかった元コントリビューターから見たpoacの現況と魔改造の記録でした．
「なんか作り直しとる」とか「へー，あの人コントリビューションしたんだ」とか「最近海外の人達がたくさんコントリビューションしてるなぁ」とかをメール通知越しに遠巻きに眺めていただけだったのですが，静的ライブラリをビルドできるようになったのを見て驚いたのが記事を書こうと思ったきっかけです[^締め切り4日前になるまでネタ思いつかなかったのか…]．再開発前には無かった機能でしたが，これがあれば(やりようによっては)ヘッダオンリーでない既存ライブラリの対応を進める目処も少しは立ちます．現状がどの程度実用的なのか，実用的でないとしたらどのような変更が必要なのかを追いかけようと思える変更でした．
そうして追いかけた中で，大規模作り直し以前(私がコントリビュートしてた頃)から見て「おっこれ入ったんだ」と思った静的ライブラリサポート以外の機能としてgit dependencyがあります．poacは長らく独自のパッケージ仕様について進捗が出ず，それ故公式リポジトリも特にメンテはされていなかったのですが，パッケージマネージャの命はパッケージレジストリであり，poacが広く使われるためにはパッケージレジストリの整備は急務だと当時私は考えていました．gitリポジトリをうまく拾ってこれさえすればユーザーがパッケージレジストリを構築することもできますからね．まぁ現状だと特定のコミットハッシュしか拾ってこれないので各依存が要求する複数のバージョンのネゴシエーションとったりはできなくて，そうした夢を実現するにはまだまだ機能が足りないのですが(そしてそれをやろうとすると上述の大変な実装が必要なのですが)，ともあれgitのリポジトリから拾ってくる機能が標準に搭載されたのは大きな一歩だと思います．

C++の体験の良いパッケージマネージャへの道のりはまだまだ遠いですが，みなさんもpoacに興味が湧いたら是非使ってみたりコントリビューションしてみてください．

---

21日分は[マホウ(@tyanmahou)](https://twitter.com/tyanmahou)さんの「[【闇】力技によるメンバ変数名取得](https://qiita.com/tyanmahou/items/aedeb7838a776de118f1)」です．
