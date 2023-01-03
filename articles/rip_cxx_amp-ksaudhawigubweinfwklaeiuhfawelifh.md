---
title: "C++ AMPの死について"
emoji: "🪦"
type: "tech"
topics: ["cpp"]
published: true
---

# C++ AMPの死について

C++ AMPが死ぬことが決まった．本稿では，C++ AMPとは何だったのか，直近・今後のC++ AMPの動向，C++ AMPの死，そしてC++ AMPの代替となりうる存在についての検討を，個人の見解として述べる．

## C++ AMPとは

C++ AMP(**A**ccelerated **M**assive **P**arallelism)はMicrosoftが策定したデータ並列プログラミングAPIである^[公式曰く "C++ AMP is a compiler and programming model extension to C++ that enables the acceleration of C++ code on data-parallel hardware" とのことなので，「データ並列プログラミング向けのC++に対するコンパイラ及びプログラミングモデル拡張」と称するのが正しそうだが，面倒なので本稿では単に API と称することにする]．C++ AMPで記述されたプログラムはCPUのSIMD演算器やGPUといったデータ並列に強い(≒SIMDな)プロセッサにオフロードされ，高速に処理される^[CPUバックエンドも存在することから厳密にはヘテロジニアスコンピューティングAPIと呼ぶべきではなさそうだ(し，現に公式では _heterogenous_ ではなく _data-parallel_ という用語を多用している)が，本稿においてはNVIDIAのdiscrete GPUにオフロードする用途で利用しているので，事実上ヘテロジニアスコンピューティングAPIとして運用することを念頭に置く]．C++の名を冠する通り，C++ AMPはC++をベースとしてライブラリとコンパイラ拡張で構成されており，CUDA C/C++のようにデータ並列部分とそれ以外のコードを単一ソースに記述することができる．このあたりは実例を見たほうが速いという読者が多いと思うので，以下にsaxpyのコードを示す．

```cpp
#include<vector>
#include<cstddef>
#include<random>
#include<iostream>
#include<cassert>
#include<amp.h>

std::vector<float> saxpy(concurrency::accelerator_view accv, const std::vector<float>& x, const std::vector<float>& y, float a){
    std::vector<float> z(x.size());
    concurrency::array_view<const float, 1> xv(x);
    concurrency::array_view<const float, 1> yv(y);
    concurrency::array_view<float, 1> zv(z);
    concurrency::parallel_for_each(accv, xv.get_extent(), [xv, yv, zv, a](concurrency::index<1> idx)restrict(amp){
        zv[idx] = a * xv[idx] + yv[idx];
    });
    return z;
}

int main(){
    static constexpr std::size_t size = 10;
    std::mt19937 rand(std::random_device{}());
    std::uniform_real_distribution<float> dist(0.f, 10.f);
    const auto rnd = [&rand, &dist]{return dist(rand);};
    std::vector<float> x(size);
    std::vector<float> y(size);
    std::generate(x.begin(), x.end(), rnd);
    std::generate(y.begin(), y.end(), rnd);
    const auto a = rnd();

    concurrency::accelerator acc{concurrency::accelerator::default_accelerator};
    const auto z = saxpy(acc.default_view, x, y, a);

    assert(z.size() == size);
    for(std::size_t i = 0; i < size; ++i)
        assert(a*x[i]+y[i] == z[i]);
}
```

上記のように， `restrict(amp)` を付された関数を `concurrency::parallel_for_each` でオフロードすることができる．特にラムダ式を用いることでオフロードするコードをその場に記述することができる，というのがC++ AMPの売りの1つだったように思う．また， `std::vector` や連続するメモリ領域に格納されたデータについては `concurrency::array_view` を用いることでH2D/D2Hの転送を明に書かずとも裏でよしなにコヒーレンシを取ってくれたり，色々といい感じにヘテロジニアスなコードを書けるやつである．

上述の通り，C++ AMPはMicrosoftが **策定** したAPIである．[C++ AMPの規格は公開されており](http://download.microsoft.com/download/2/2/9/22972859-15C2-4D96-97AE-93344241D56C/CppAMPOpenSpecificationV12.pdf)，誰でも自由にC++ AMPを実装することができる．「いくら公開されてるからと言ってMicrosoftが策定した言語拡張を誰が実装するんだよ」などと思われるかもしれないが，x265の開発を主導していることで有名な[MulticoreWare社](https://multicorewareinc.com/)がClangベースのヘテロジニアスコンピューティングコンパイラであるclampに実装し，プロジェクト名やら所属やらが転々とした結果今でもAMDのROCmプロジェクトの下に[hcc](https://github.com/RadeonOpenCompute/hcc)として転がっている．

C++ AMPは以下の特徴を備えたAPIであり，これらをオープンな規格とまともな実装を揃えて提供されている例を私は他に知らない．

- 単一ソースで記述できるC++ベースのヘテロジニアスコンピューティングAPI
    - 上記のコードではprimitiveな型を用いたが，ユーザー定義のクラスであってもデバイス上で用いることができる．
        - 例えばテンプレートクラスの `pair` などを定義すれば `argmin` などを自然に定義でき，その結果の `pair` 型オブジェクトをそのままホストでも使える．
    - さらにこうしたコードをヘッダオンリーライブラリとして提供することも可能である．
- 最新のC++の機能が使える
    - これは実装に関する話であるが，MSVCではC++20の言語機能も着々と実装が進んでいる．C++ AMPはC++の言語拡張であるためこれらの言語機能をカーネルの記述に用いることができる．
        - C++17の例になってしまうが，入力型によって処理内容を切り替えるカーネルファンクタを `if constexpr` を用いて実装する，といったことが可能．
- MSVCに搭載されている
    - CUDAやOpenCLとは異なりVisual Studioさえインストールされていれば動作する．
    - Visual StudioがインストールされているWindows開発環境は多いが，CUDA ToolkitやOpenCLのSDKなどが同時にインストールされている環境はそこまで多くない．環境構築は(特に不慣れな人間にとって)難しい作業であり，Windowsユーザーに「今すぐできるGPGPU」として布教する上で最適な環境がMSVCのC++ AMPなのだ．
- グラフィックスAPIとのinteropがスムーズ
    - これまたMSVC限定^[(C++ AMP規格上でも規定されているが拡張扱いであり，HCCは未対応かつC++ AMPのまともな実装がこの2つぐらいしかないので事実上)MSVC限定]の話になるが，MSVCではC++ AMPのバックエンドにDirectComputeを用いており，DirectXのdevice contextからC++ AMPの `accelerator_view` を作ったり，DirectXのtexture(`ID3D11Texture2D` や `IDXGISurface` など)とC++ AMPの `texture` (これは若干制約のある `array` のようなものだと思ってもらえれば良い)の相互変換を行えるなど，DirectXとのinteropが可能だ．
        - これにより，C++ AMPを用いたコンピュートシェーダで作り出した画像をDirectXで描画する，といった処理を無駄なH2D/D2H転送無しに行うことができる．
- 環境に依存しない
    - MSVCの場合，バックエンドはDirectComputeであるため，GPUがNVIDIAであってもAMDであってもIntelであっても動作する．また，CPUバックエンドも存在するため最悪GPUがなくても動作する(当然GPUに比べれば遅いが)．
    - C++ AMPはオープンな規格であり，実際にLinux上ではOpenCLバックエンドの実装であるclampが存在した．
        - 現在ではROCmプロジェクト下に移管されてしまったので，AMDのGPU以外で動くかどうかはちょっと怪しい気がする．
        - また，上記のDirectX interop APIなどを用いない場合の話となる．

### 近年のC++ AMP事情

初登場から10年近く経過したC++ AMPだが，近年では公式から[C++ AMPはメンテナンスモード](https://developercommunity.visualstudio.com/t/question-what-is-microsofts-plan-for-c-amp-do-you/225629#T-N234321)と声明があり，C++ AMPに新たに機能が実装されることは無くなっているが，引き続きコンパイラのバグなどへの対応は行われている．
一方で既存資産を幅広い環境で動かすこと自体には積極的な姿勢を見せており，ちょうど去年の今頃には[Visual Studio 2019 16.8.0 Preview 2.1にてARM64サポートが追加された](https://developercommunity.visualstudio.com/t/Support-C-AMP-on-ARM64/467081#T-N1171839)との報告もある．

…というのが表向きの状況なのだが，MSVCにおいて「メンテナンスモード」であるにも関わらず実際にはろくにメンテナンスされていなかった――Microsoftの都合に寄り添った物言いをすれば，メンテナンスが _追いついていなかった_ のが実態である．
MSVCは近年C++20対応など標準規格準拠に向けたコンパイラへの変更が大量に行われており，その煽りを食っているのが既存の独自拡張の山だ．C++ AMPもその例に漏れず，過去のコンパイラでコンパイルできたコードがコンパイルできない，といったバグはもちろんのこと，そもそもヘッダが過去のMSVCの(規格違反な)仕様を前提として実装されていたためincludeするとビルドエラーになるなど，散々な状態であった．しかも，MSVCは古いバージョンが公式からは配布されてないので一度Visual StudioをC++ AMPが壊れた新しいバージョンに更新するとプロジェクトが詰むという地獄のような状況である．私はこれで詰んでいます．

また，MSVC以外でいうと，HCCは2019年に最終リリースを迎えdeprecatedになっているのでWindowsの外では既に事実上の死を迎えている．

## C++ AMPの死

Visual Studio 2022 17.0 Preview 2のリリースノートにおいて以下の記述があった．
突然の訃報であった．

> [C++ AMP headers are now deprecated. Including <amp.h> in a C++ project will generate build errors. To silence the errors, define _SILENCE_AMP_DEPRECATION_WARNINGS. Please see [our AMP Deprecation links]https://aka.ms/amp_deprecate for more details.](https://docs.microsoft.com/en-us/visualstudio/releases/2022/release-notes-preview#1700-pre20--visual-studio-2022-version-170-preview-2)

リンクの壊れた "our AMP Deprecation links" に飛んだ先の記述が以下である．

> [C++ AMP headers are deprecated, starting with Visual Studio 2022 version 17.0. Including any AMP headers will generate build errors. Define _SILENCE_AMP_DEPRECATION_WARNINGS before including any AMP headers to silence the warnings.](https://aka.ms/amp_deprecate)

そして，実際にVS2022 Previewにおいて `_SILENCE_AMP_DEPRECATION_WARNINGS` を定義せずにビルドした際のエラーメッセージが以下である(ヘッダ名はincludeしたヘッダによって変わる)．

> `#error:  <amp.h> is part of C++ AMP which is deprecated by Microsoft and will be REMOVED. You can define _SILENCE_AMP_DEPRECATION_WARNINGS to acknowledge that you have received this warning.`

斯くして，C++ AMPはdeprecatedに，そしていずれ削除されることになった．1年前はARM対応とかやってたのにどうして…

死因について推測してみると，まぁユーザーが比較的少ない中コンパイラの改変に追従しきれなくなってサポートが限界に来たということなんだろう，たぶん．
個人的にはMFCやATLなんかをサポートしてる暇があったらC++ AMPサポート継続してくれと思って仕方がないのだが，(独自拡張に多少依存していたとしても)ただのライブラリとコンパイラ拡張ではサポートコストが全然違うし，ユーザー数も残念ながらMFCやATLの方が圧倒的に多い気がするのでこれを言うのは無理があるだろう．

### これからのC++ AMP

C++ AMPはdeprecatedになったので，とりあえず死ぬ事自体は避けられないし今後積極的に使っていくべき技術ではないのだが，それはそれとして今後のサポート状況について，以下のような声明があった．

> [we plan to support them at least through VS 2022 lifecycle, which is 5 years after last update + another 5 for extended support](https://developercommunity.visualstudio.com/t/c-amp-headers-are-deprecated-what-is-the-replaceme/1495203#TPIN-N1497575)

ということで，とりあえずVS2022までは引き続きサポートされるらしい^[ほんとか？ちゃんとバグ取るか？ / ちなみにVS2022 Preview2時点では私が報告した[このバグ](https://developercommunity.visualstudio.com/t/C-AMP-doesnt-work-in-lambda-expressio/1448841)が未修正である．先述のヘッダincludeしただけでCE問題は直っていた]．
VS2022のさらに次のバージョンでも引き続きVS2022のMSVCをインストールすれば一応使えるはずだ．かろうじてむこう10年は既存資産がそのまま実行できる，というのが **現状の** 見通しである．

### C++ AMPからの移行先

C++ AMPでいくつかプロジェクトを書いてきたわけだが，これからも使い続けるわけにはいかない．というわけで，C++ AMPと同等の特徴を持ったヘテロジニアスコンピューティングAPIを…と言いたいのだが，これが(おそらく)無いのである．
以下に検討した移行先について表にまとめ，その後それぞれについて詳しく述べる．なお，ここではWindows上で動作するGUIプログラムと連携させる，という用途を念頭に置いている．

|  | 単一ソース | 最新のC++の機能 | グラフィックスAPIとのinterop | デバイス非依存 | 突然死の危険性がない |
| - | :-: | :-: | :-: | :-: | :-: |
| C++ AMP | ○ | ○ | ○ | ○ | × |
| CUDA | ○ | × | ○ | × | ○ |
| OpenCL | × | × | ○ | ○ | ○ |
| HIP | ○ | △ | × | △ | ？ |
| HC C++ API | ○ | × | × | × | × |
| SYCL | ○ | ○ | ？ | ○ | ○ |

1. C++ AMP
    - 必要な要件は概ね満たしているが，だからといって使い続けるかと言われると厳しい．未来がないことが確定しているからだ．
    - あと先述の通り，VS2019のMSVCでC++ AMPを運用するのはあまりにもbuggyが過ぎるので大変厳しい．
2. CUDA
    - NVIDIAのGPUを使うことを念頭に置けば非常に現実的な解だと思う
        - (一応)単一ソースで書ける
        - [DirectXとのinterop APIが生えている](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__D3D11.html)
        - ユーザーもかなりいるため，今更NVIDIAがCUDAを完全に廃止するとは思えない
    - 一方で，デバイスがNVIDIA GPUに縛られること， `nvcc` のC++サポートがありえん遅いので最新のC++機能を活用することが叶わないのはもちろん，最新の標準ヘッダを読ませるとフロントエンドがエラーを吐いてしまうので最新機能を併用するなら事実上シングルソースに出来ないという問題もある．
    - あと単純にCUDA Toolkitのインストールやら更新やらが面倒という話も…
3. OpenCL
    - シングルソースに出来ず，またカーネルの記述は(殆どのデバイスでは)OpenCL Cでの記述が要求されるので，正直厳しい．
    - Microsoftの拡張で[DirectX interop APIは生えている](https://www.khronos.org/registry/OpenCL/sdk/2.2/docs/man/html/cl_khr_d3d11_sharing.html)
    - デバイスは当然非依存だし，OpenCLの規格自体が死ぬことは無いと言ってよいだろう．問題は実装についてだが，
        - KhronosのpresidentであるNeil TrevettがNVIDIAのvice presidentを兼任しているので，彼が任を解かれない限りNVIDIAがコンシューマ向けGPUのOpenCLサポートを辞めるということはないはずだ^[OpenCL未サポートのNVIDIA製品として，Jetsonシリーズがある]．
        - AMDも現状Windows上ではOpenCLを使え，というのが公式見解なので当面は大丈夫だろう．
        - IntelもOpenCL 2.0の策定などに結構口出ししていた印象なのでたぶん大丈夫だと信じている．
        - ところでOpenCLの策定に関わっていたのに近年自社製品ではdeprecatedにしてしまったAppleという会社があって…
4. HIP
    - その実態はちょっとしたプリプロセッサと各コンパイラへの振り分けを行うスクリプト群だったはず．
    - 一応HIP Clangであれば `nvcc` よりはC++対応はマシだろうが，Windowsで動作させるなら現実的には `nvcc` がバックエンドとなるため， `nvcc` の問題はすべて引きずることになる．また，Intel製GPUをサポートできない．
    - 加えて，現状AMDとしては結構HIPに力を入れているように見えるが，これがいつまで続くのかが見えない．後述するように，既に `hcc` を捨てたことがあるので…
        - なんか最近は[OpenMPのGPU Offloadingに力を入れているらしい](https://github.com/wx257osn2/zenn/issues/2)．
5. HC(Heterogeneous Compute) C++ API
    - 単一ソース，C++ AMPを参考によりC++に寄せた記法，という謳い文句で登場し，Clangベースのコンパイラである `hcc` に実装された．が，C++ AMPより後発であったこと，Linux環境上のAMD GPU上でしか動作しないことなどからユーザー数は殆どいなかったのではないだろうか…
        - 今回はWindows上での開発を念頭に置いているのでこの時点で使えない
    - [`hcc` 自体を直接扱うのはとうの昔にdeprecatedになっている](https://github.com/RadeonOpenCompute/hcc#deprecation-notice)上，[去年の時点でHIPのバックエンドとしての役目も外されてしまった](https://github.com/RadeonOpenCompute/ROCm/tree/roc-3.5.0#Upgrading-to-This-Release)．死にゆく言語拡張から死んだコンパイラに移行するわけないだろ！
6. SYCL
    - 大本命．生ける(今なお更新が続く)規格(≒今後のC++の規格への追従に積極的)，単一ソース，複数の実装(≒多様なデバイスのサポート)．
    - さて，問題はWindows上でまともに動く実装があるのかというと，あまり調べていないのでよくわからない．目的に叶う実装でいうと:
        - ComputeCppはWindowsで動作し，多様なGPUで動作しそうに見える．
        - IntelのDPC++が大本命．IntelがoneAPIを打ち出して間もないので，しばらくは死にそうにないという話もある．
        - 今回の目的で言えばtriSYCL，hipSYCL，neoSYCLは考えなくて良い．
    - 懸念点としてはグラフィックスAPIとのinteropがある．ここはSYCL自体はサポートしておらず，おそらくはSYCLのOpenCL interopとOpenCLのグラフィックスAPI interopを繋ぎ合わせてSYCL上にグラフィックスAPIのリソースを持ち込むことになるはず．つまり，グラフィックスAPIとのinteropをサポートしたOpenCLのSDKが既存のSYCL実装のinteropで繋ぎ合わせられなければならないのだが，この辺がどうなっているかは未調査で，実現できるという確証が無い．
    - SYCLの実装については今後暇なときに調査しようと思っているが，時間がないので誰かやってくれるとすげぇ助かる

とまぁこんな感じで，要件を満たしているAPIが存在するとは言い切れず，また仮にSYCLが要件を満たしたとしてC++ AMPに比べてグラフィックスAPIのリソースを持ち込む方法はどうしても迂遠になる．移植コストもそれなりに高くつくであろう．さらに，どう足掻いても「Visual Studioさえあれば使えます！」というヘッダオンリーライブラリにはならないので，どうしても厳しい面が残る^[この点は環境を指定して別個のライブラリとして実装すればよいのだが，一方でグラフィックスAPI(のラッパー)との乗り入れがしにくくなる面があり，従来は1つのライブラリでうまいことできていたのに…どうして…]．

さて，私と同様に[既存のC++ AMP資産が突然のdeprecatedによって死滅することとなった被害者によるfeedback](https://developercommunity.visualstudio.com/t/c-amp-headers-are-deprecated-what-is-the-replaceme/1495203)がある(先程の今後のサポート状況の見通しはこのfeedbackに対する回答である)．彼もまたOpenCL，OpenACC，C++17 Parallel Algorithmsそれぞれについて検討した結果いずれも目的には合致しなかったようである^[特に彼の場合は「VSやMSBuildがサポートしている」「比較的容易に記述できる」あたりが難点か]が，このfeedbackに以下のような返答が記載されている．

> [About the replacement technologies, we currently have plans to add them to the VC AMP documentation page at https://docs.microsoft.com/en-us/cpp/parallel/amp/cpp-amp-overview?view=msvc-160.](https://developercommunity.visualstudio.com/t/c-amp-headers-are-deprecated-what-is-the-replaceme/1495203#TPIN-N1497575)

というわけで，そのうちMicrosoftから移行先についての提示があるかもしれない．尤も，Microsoft自身がC++ AMPが(その特殊な立ち位置故に)如何に貴重な技術であったかを理解していない可能性が非常に高く，おそらくC++ AMPユーザーが満足する移行先が提示されることはないのではないか，という不安が拭えないが…

#### C++標準の並列プログラミング系の機能について

ちゃんと調べてないけどたぶんグラフィックスAPIとのinteropが可能にはならないのではないかと思っていて，端から検討していない．まぁあと今すぐ移行できるわけでもないし…

## まとめ

エデンは滅ぶ．完．

もう少し真面目に書くと，C++ AMPはWindows上で多くのユーザーが使用しているコンパイラに標準で搭載された単一ソースで最新のC++言語機能を用いて記述できるヘテロジニアスコンピューティングAPIである．ビルドされたバイナリはGPUのベンダはおろかGPU・CPUのいずれであっても動作し，またグラフィックスAPIとのinteropによってゲームなどにも使うことのできる機能だ．しかし，Microsoftはこの機能をdeprecatedとすることを決定した．有効な代替先はまだ見つかっておらず，少ないながらもユーザーは悲鳴を上げている状況．

というわけで，C++ AMPの代替となるいい感じのヘテロジニアスコンピューティングAPIを探しております．いい感じの情報ございましたらぜひご連絡いただけますと幸いです．最近のSYCLの状況とかも是非．
~~あとMSVCチーム各位，C++ AMPサポートやりたいので有期でいいから採用してくれ~~
