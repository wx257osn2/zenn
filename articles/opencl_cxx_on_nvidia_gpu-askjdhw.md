---
title: "OpenCL C++ on NVIDIA GPU"
emoji: "🆑"
type: "tech"
topics: ["cpp"]
published: true
---

:::message
この記事は[C++ Advent Calendar 2021](https://qiita.com/advent-calendar/2021/cxx)の20日目の記事です．
:::

Iです．今年も気づけばAdCの季節．[前回の記事](rip_cxx_amp-ksaudhawigubweinfwklaeiuhfawelifh)を書いた頃はまだ暑かったのに気づけばクソ寒くなってて，どうして…

# OpenCL C++ on NVIDIA GPU

というわけで本記事なんですが，タイトル通りNVIDIA GPU上でOpenCL C++を動かす，というのをやっていきます．
といきなり言われてもなんのこっちゃという人も多いと思うので，まずはOpenCLやOpenCL C++とは何か，という辺りの話から始めて，実際に動かすところまでやっていきます．

OpenCL C++の話だけど，C++って入ってるしC++の話ということで…

## 前提知識

### OpenCL

OpenCLはKhronosという団体が策定しているヘテロジニアスコンピューティング向けのAPI規格です．
KhronosはOpenCLの規格策定までをやっていて，これに基づいて各社が自前のチップで動くようにランタイム等を実装する，という感じになっており，規格と実装は基本的に^[例外として[OpenCL C++ API](https://github.com/KhronosGroup/OpenCL-CLHPP/blob/master/include/CL/opencl.hpp)みたいなポータブルなやつはKhronosで提供していることもある]分離しています^[この規格と実装の分離，という面ではC++と似てます]．
ヘテロジニアスコンピューティング向けのAPIですので，基本的にはOpenCLで動かすのはGPUとかアクセラレータとか，CPUじゃないプロセッサになります^[一応適切なOpenCLのドライバを入れればCPUで動かすことも可能．この場合はSIMD命令を用いて高速に計算を行う]．
ユーザーはOpenCLのAPIに則ってコードを書くと，OpenCLをサポートしている様々なデバイスでプログラムが動きます．
ここで，CPU側のプログラムを **ホストコード** ，GPUやアクセラレータなどOpenCLで動かすデバイス側のプログラムを **デバイスコード** と呼びますが，ホストコードはCのAPIを用いて記述し，デバイスコードの記述にはOpenCL Cという…まぁ概ねC言語を使います．
つまり，

- (概ね)Cでコードを書くと
    - これはCをある程度書けるならば習得が容易という話でもある
    - ちなみにデバイスコード，ホストコード側では文字列として扱うか(環境固有の)precompile済みのバイナリデータを扱います．後者の場合は当然デバイスコードを別ファイルに分離しますし，前者の(実行時にデバイス向けにオンラインコンパイルする)場合以下のような方策が必要ですが，いずれにせよCUDAやC++ AMPとは異なりsingle sourceにはできません．
        - 例えば別ファイルとして書いてファイルから文字列として読み込む
        - 例えば `STRINGIZE` マクロで囲っておいて文字列として埋め込む
- (対応している並列性が高い)いろんなデバイスで動く
    - NVIDIAやAMD，IntelのGPUはもちろんのこと，スマートフォンのGPUや[一般人にはあまり触る機会の少ない変なアクセラレータ](http://tanakh.jp/posts/2016-12-20-rust-pezy-sc.html)(これはOpenCL _のような_ API，とのことですが)などでも(OpenCLをサポートしている限りは)動きます．

→ うれしい！

という，夢のある規格なわけです．

では実態はどうなのかと言うと，言い出しっぺのAppleはMetalに移行するからと3年前にOpenCLをdeprecatedにしやがるし，「どこでも動く」とは言うものの実際に各デバイスでちゃんと速度を出そうとするとデバイスに依存した最適化が必要になってきて結局デバイスごとに別の実装になるし，OpenCLサポートしてるよ！と謳っていても結構細かい所の実装がザルだったりすることもあり(聞いてるかArm，お前らのことやぞ)，まぁ理想とはやや遠い現実がそこにはありますがそれはまた別の機会に…

### OpenCL C++

さて，上述のようにデバイスコードの記述にはOpenCL Cという言語を用いるわけですが，Cなんですね．C++ではない．
もちろん現実的にはデバイスコードで使いたいC++の言語機能というのはそこまで多いわけでもないですが^[例えばリソース管理はデバイスコードの実行前後でホストコード側で行うため，デバイスコード内で細かく行うことは基本的には無くて，RAIIが欲しくなる機会はあまり無いと思います]，例えばオーバーロードやoperator overloadも使えませんし，なによりtemplateが使えないわけです．これは意外と困ります．
というのも，デバイス次第ではありますが分岐がCPUと比べて苦手なデバイスを扱う場合は減らせる条件分岐は減らしたいわけです．
そうなると，一部のパラメータはコンパイル時のパラメータとして受け取っておいて，パラメータ毎に別のバイナリコードを吐き出したい…というような需要が普通にあります．
そこでtemplateが使えないと…みんな大好きプリプロセッサマクロで頑張ることになります．頑張りました．
書いてる分にはいいんですが読む側のことを思うとあまりやりたくはないですね．いやごめん正直書くのもしんどかったわ．

また，templateはGPGPU上で再帰的なコードを記述するのにも役に立ちます．
多くのGPGPU APIでは関数の再帰呼び出しを禁止しており，そのようなコードを書くとコンパイル時にエラーとなります．
これはGPU内でコールスタックのようなものを保持していないことが主な理由(だというのが私の認識)ですが，一方で再帰的なコードを書きたくなることはもちろんあるわけです．
ではこれをどうするかというと，[11日目の記事](https://takkyu.net/ja/polymorphic-recursion/)に出てきたtemplateを用いた有限再帰深度での多相再帰のような方法(以下疑似再帰と呼ぶことにします)で回避します．
C++のtemplateはtemplate引数毎に異なるシンボルになるので，疑似再帰は(同シンボル関数の)再帰呼び出しではなく，したがってコンパイラの再帰呼び出しチェックにひっかかりません．
再帰深度を非型template引数として持ち，一定の再帰深度に到達したらデフォルト値を返す，のようなコードを書けば，容易に再帰的なコードを書くことができます^[尤も，これはカーネルがメチャクチャに肥大化することと同義でもあるので，再帰深度や疑似再帰関数のサイズには気をつけましょう]．

さて，このような状況に対してKhronosが何もしていないかというとそんなことはなくて，Khronosはちゃんとデバイスコード記述用の言語である[OpenCL C++](https://www.khronos.org/registry/OpenCL/specs/2.2/html/OpenCL_Cxx.html)というものを策定してくれています．
ところが，OpenCL C++の話というのは調べてもろくに出てきません(後述する[OpenCL C++ API](https://github.com/KhronosGroup/OpenCL-CLHPP/blob/master/include/CL/opencl.hpp)の話しか出てきません)．

**なぜなら！！！もうお分かりだろう！！！**

**そう！！誰も！！ OpenCL C++を実装しないのである！！！**

…いやそれはそうなんですよ．上述のようにOpenCLのデバイスコードは

- OpenCLランタイムに含めたコンパイラで実行時にオンラインコンパイル
- SDKに含めたコンパイラで実行前にオフラインコンパイル

のいずれかで実行可能な形式に変換していきます．
つまり，OpenCL C++をサポートするということはOpenCLを実装する各ベンダが(OpenCLランタイム上にせよSDK内のバイナリ群にせよ)C++コンパイラを実装することと同義なわけで…まぁ，OpenCL Cさえ実装しておけばOpenCL準拠と言える以上，わざわざOpenCL C++を実装しようとはしないわけです．
斯くしてOpenCL C++は幻の「実装が(少なくとも有名なデバイスにおいて)存在しない絵空事規格」となってしまいました．

### OpenCL C++ API

上述の説明で出てきたOpenCL C++ APIというのは何かというと，OpenCLのホスト側APIのことです．
先に出てきたとおりホスト側のAPIもCで書かなければいけないわけですが，そうなるとやはり面倒なのはリソース管理．
そうした面倒事をいい感じに扱ってくれるのがこのOpenCL C++ APIなわけです．
実装的にも各ベンダの提供するCのOpenCL APIの上にラッパーライブラリの形で実現でき，環境に依存する部分が少ないためか[GitHub上でKhronosが実装を公開しています](https://github.com/KhronosGroup/OpenCL-CLHPP/blob/master/include/CL/opencl.hpp)．

今回の主題であるOpenCL C++とはまるで異なるものですが(OpenCL C++ APIはホストコードを書くためのもの，OpenCL C++はデバイスコードを書くためのもの)，検索で「OpenCL C++」などとすると基本的にはこちらが出てきます．
人によってはOpenCL C++のことを知らないままにOpenCL C++ APIを指して「OpenCL C++」と記述していることもあります．
「OpenCL C++」という文字列を見かけた際はどちらについて話しているのかはよく気をつけて読むと良いでしょう．

## 実際にやってみた

というわけで，OpenCL C++を使うことは現実的には難しいわけですが，今回はOpenCL C++で記述した2次元畳み込み(以下conv2d)のデバイスコードをNVIDIA GPU上で動かす，というのをやっていきたいと思います．
と言っても，別にNVIDIAのOpenCLランタイムではOpenCL C++をサポートしているわけではありません^[そもそももっと言えば彼らのOpenCLサポートは割と貧弱です．それもそのはず，NVIDIA GPU上においてはOpenCLはCUDAと競合する存在であり，彼らとしてはCUDAを使ってほしいわけで，OpenCLをそこまで全力でサポートしてやる気は端から無いわけです．それでもユーザー数がそこそこいるからか最低限の部分はちゃんと動くので，これでも比較的よくできていると言って良いでしょう．世の中にはまじで破綻したOpenCLランタイム実装があって…聞いてるかArm，お前らのことやぞ(2回目)]．
ではどうするのかと言うと…

### CUDAでconv2d

NVIDIA GPUでGPGPUしたい，とくればやはりCUDAでしょう．「OpenCLの記事なのにCUDA…？」と思われるかもしれませんが，まぁそう仰らずに…
ここでは簡単のために，シングルチャネル画像を扱うことにします．ナイーブな実装は以下のようになるでしょう:

```cpp:conv.cu
__global__ void convolution_general(
    const unsigned char* __restrict__ im,
    int width,
    int height,
    const float* __restrict__ kernel,
    int kernel_size,
    unsigned char* __restrict__ output){
  const int x = blockIdx.x * blockDim.x + threadIdx.x;
  const int y = blockIdx.y * blockDim.y + threadIdx.y;

  const int half_k = kernel_size / 2;
  if(y < half_k
  || height - half_k <= y
  || x < half_k
  || width - half_k <= x)
    return;
  float t = 0.f;
  for(int i = 0; i < kernel_size; ++i)
    for(int j = 0; j < kernel_size; ++j)
      t += im[(y+j-half_k)*width+x+i-half_k] * kernel[i*kernel_size+j];
  output[y*width+x] = static_cast<unsigned char>(min(max(t, 0.f), 255.f));
}

void launch_convolution_gpu(
    const unsigned char* __restrict__ im,
    int width,
    int height,
    const float* __restrict__ kernel,
    int kernel_size,
    unsigned char* __restrict__ output){
  assert(kernel_size % 2 == 1);
  const dim3 threads(32, 32);
  convolution_general<<<threads, dim3((width+threads.x-1)/threads.x, (height+threads.y-1)/threads.y)>>>(
    im, width, height, kernel, kernel_size, output);
}
```

カーネル(conv2dのフィルタ)サイズ $K$ に対して2重ループするカーネル(GPU関数)を画像サイズで並列させます．
これらを呼び出すホストコードは長くなるのでひとまず省略します．
ビルドルールは以下です．

```make
conv_cu.o: conv.cu
	nvcc -O3 -c -o $@ $<

conv.o: conv.cpp
	g++ -std=c++2a -Wall -Wextra -pedantic-errors -I./include -I$(CUDA_PATH)/include -O3 -c -o $@ $<

conv: conv.o conv_cu.o
	g++ -lcuda -L$(CUDA_PATH)/lib64/stubs -L$(CUDA_PATH)/lib64 -lcudadevrt -lcudart_static -lrt -lpthread -ldl -lcudart -O3 -o $@ $^
```

ここで「何故ホストコード `conv.cpp` のビルドを分けているんだ？」と思われたかもしれませんが，(少なくともこれ試した環境に入ってた) `nvcc` にはC++20がわからぬ．^[`nvcc` は謎の半導体メーカーのコンパイラである．エラーを吐き，GPUを燃やして暮して来た．けれども拡張子に対しては，人一倍に敏感であった．]から(ホストコードはC++20で書きたかったから)です．[聞いてるかMS，どうしてC++ AMPを…](rip_cxx_amp-ksaudhawigubweinfwklaeiuhfawelifh)

### CUDA Driver APIに書き換える

CUDAといえばシングルソースなGPGPU言語，という印象が強いかと思いますが，実はオフラインコンパイルしたカーネルを実行時にロードして起動することもできます．
上記で使ったの(CUDAと呼ばれて多くの人が思いつくやつ)はCUDA Runtime APIというのですが，今回はCUDA Driver APIというものを使います．
先にビルドルールを示します．

```make
conv_cu.s: conv.cu
	nvcc --ptx -O3 -c -o $@ $<

conv: conv.cpp
	g++ -std=c++2a -Wall -Wextra -pedantic-errors -I./include -I$(CUDA_PATH)/include -lcuda -L$(CUDA_PATH)/lib64/stubs -L$(CUDA_PATH)/lib64 -lcudadevrt -lcudart_static -lrt -lpthread -ldl -O3 -o $@ $^
```

カーネルを記述した `conv.cu` をビルドする際， `nvcc` に `--ptx` オプションを渡すことでNvidia PTXというアセンブリのようなものを吐かせることができます．
この `conv_cu.s` を読み込み，GPUのドライバに向けてアセンブル，カーネルの呼び出し，というのをCUDA Driver APIを使って実行時に行うわけです．
上記で言うところの `launch_convolution_gpu` 関数は削除し， `conv.cu` にはカーネルのみを記述するようにしておきます．
カーネル起動周りのホストコードは以下のようになります．

```cpp:conv.cpp(一部抜粋)
static void cu_launch_kernel(
    ::CUfunction f,
    void** kernel_params,
    unsigned int grid_dim_x,
    unsigned int grid_dim_y,
    unsigned int grid_dim_z,
    unsigned int block_dim_x,
    unsigned int block_dim_y,
    unsigned int block_dim_z,
    unsigned int shared_mem_bytes = 0,
    ::CUstream stream = nullptr,
    void** extra = nullptr){
  const auto err = ::cuLaunchKernel(f, grid_dim_x, grid_dim_y, grid_dim_z, block_dim_x, block_dim_y, block_dim_z,
                                    shared_mem_bytes, stream, kernel_params, extra);
  if(err != CUDA_SUCCESS)
    throw std::runtime_error("cuLaunchKernel: " + cu_get_error_name(err));
}

::CUmodule mod;
::cuModuleLoad(&mod, "conv_cu.s");
::CUfunction func;
::cuModuleGetFunction(&func, data, "convolution_general");
int width = im.get_width();
int height = im.get_height();
void* kernel_args[] = {
    &device_image,
    &width,
    &height,
    &device_kernel,
    &kernel_size,
    &device_output
};
::cuLaunchKernel(func, 32, 32, 1, (width+31)/32, (height+31)/32, 1, 0, nullptr, kernel_args, nullptr);
```

`conv_cu.s` から `convolution_general` というシンボルの関数を拾ってきて `kernel_args` に詰めた引数で呼び出します．
ご覧のように，値渡しの引数であれば `kernel_args` にアドレスを渡すことで自動で `cuLaunchKernel` が転送してくれます(逆に言えば定数であっても一度変数に詰める必要があります…)．
デバイスメモリについては事前に `cuMemAlloc` 関数を使って確保しておき，ハンドル `CUdeviceptr` の変数のアドレスを `kernel_args` に詰めることでカーネルに渡せます．

### カーネルのビルドにClangを使う

実はClangはnvptxを吐けます．というわけで，ビルドルールをClangを使うように書き換えます．

```make
conv_cu.s: conv.cu
	clang -xcuda -S --cuda-device-only --cuda-gpu-arch=sm_60 -O3 -o $@ $<
```

`clang` には以下のようなオプションを渡しています:

- 入力(`conv.cu`)をCUDAのコードとして扱う: `-xcuda`
- アセンブリ出力: `-S`
    - CUDAのコードを処理するとデフォルトでターゲットはnvptxになります．
- デバイスコードのみビルドする: `--cuda-device-only`
- NVIDIA GPUのアーキ指定: `--cuda-gpu-arch=sm_60`
    - 新しいGPUが欲しいですね…(白目)

### ClangでOpenCL Cをビルドする

上記のようにClangでnvptxが吐けることがわかりました．
さて，皆さんご存知のようにClangはフロントエンドとバックエンドをIRで繋ぐ，という構造をとっています．
つまり，CUDA C++フロントエンドに対してnvptxバックエンドを繋いだのが上記なわけですね．
実はClangはOpenCL Cフロントエンドも持っています．
ビルドルールを以下のように書き換えましょう．

```make
conv_cl.s: conv.cl cl_stub.h
	clang -Xclang -finclude-default-header -xcl -cl-std=CL1.2 -S -target -nvptx64-nvidia-cuda -includecl_stub.h -O3 -o $@ $<
```

`-xcl` とすることで入力をOpenCLとして扱ってくれ，さらに `-cl-std=CL1.2` によってOpenCL C 1.2を指定します．
今回はCUDAのコードではないので，ターゲットアーキテクチャは `-target -nvptx64-nvidia-cuda` を明示します．
また，事前に用意した `cl_stub.h` やOpenCLのデフォルトヘッダを強制includeさせます．
`cl_stub.h` の中身は以下のとおりです．

```cpp:cl_stub.h
size_t __get_global_id(uint x){
  if(x == 0)
    return __nvvm_read_ptx_sreg_tid_x() + __nvvm_read_ptx_sreg_ctaid_x() * __nvvm_read_ptx_sreg_ntid_x();
  if(x == 1)
    return __nvvm_read_ptx_sreg_tid_y() + __nvvm_read_ptx_sreg_ctaid_y() * __nvvm_read_ptx_sreg_ntid_y();
  if(x == 2)
    return __nvvm_read_ptx_sreg_tid_z() + __nvvm_read_ptx_sreg_ctaid_z() * __nvvm_read_ptx_sreg_ntid_z();
}

#define get_global_id __get_global_id

float __min(float x, float y){
  return x < y ? x : y;
}

#define min __min

float __max(float x, float y){
  return x > y ? x : y;
}

#define max __max
```

Clang同梱のOpenCLのデフォルトヘッダはnvptxバックエンド向けに作られていないので，必要なOpenCL C関数をマクロで上書きしてやります．
デバイスコードは以下のようになります．

```cpp:conv.cl
__kernel void convolution_general(
    __global const unsigned char* __restrict__ im,
    int width,
    int height,
    __global const float* __restrict__ kern,
    int kernel_size,
    __global unsigned char* __restrict__ output){
  const int x = get_global_id(0);
  const int y = get_global_id(1);

  const int half_k = kernel_size / 2;
  if(y < half_k
  || height - half_k <= y
  || x < half_k
  || width - half_k <= x)
    return;
  float t = 0.f;
  for(int i = 0; i < kernel_size; ++i)
    for(int j = 0; j < kernel_size; ++j)
      t += im[(y+j-half_k)*width+x+i-half_k] * kern[i*kernel_size+j];
  output[y*width+x] = (unsigned char)(min(max(t, 0.f), 255.f));
}
```

ところどころattributeの表記がOpenCLのものになっていますが，大筋はCUDA版と変わりませんね．

### ClangでOpenCL C++をビルドする

というわけでようやく本記事の本題なのですが，実はClangはOpenCL C++フロントエンドも持っています．
`-cl-std=CLC++` とすることでOpenCL C++でコンパイルします．

```make
conv_clcpp.s: conv.clcpp cl_stub.hpp
	clang -Xclang -finclude-default-header -x cl -cl-std=CLC++ -S -target -nvptx64-nvidia-cuda -includecl_stub.hpp -O3 -o $@ $<
```

### templateでループアンローリングを行う

前準備をステップを刻んで進めた結果，なんだかあっけなくOpenCL C++が使えるようになってしまいました．
せっかくOpenCL C++が使えるので，前述の通りGPGPUと比較的相性が良いtemplateを使ってみましょう．
今回はtemplateを用いてループアンローリングを行ってみます．
ループアンローリングはループ継続判定の分岐を削除する高速化手法であり，GPGPUだとそこそこ効果があります^[現代のCPUだと物にもよりますがどちらかというとiCacheに乗り切るようにプログラムサイズを削減する方向に持っていったほうが速くなるという話も聞きます．まぁ最後は実測値が全てなのでちゃんと計測をしよう]．
ところが，例えばカーネルサイズが `3` なら9個の， `5` なら25個の， `7` なら49個の文を職人技で書き連ねる…というのはあまりやりたくありません．
そんなときに便利なのが疑似再帰になります．

```cpp:conv.clcpp
namespace detail{

template<int KernelSize, int J, int I>
static inline float unroll_x(
    const int x,
    const int y,
    __global const unsigned char* __restrict__ im,
    const int width,
    __global const float* __restrict__ kern){
  static constexpr int half_size = KernelSize/2;
  static constexpr int dx = I - half_size;
  static constexpr int dy = J - half_size;
  if constexpr(I == 0)
    return im[(y+dy)*width+x+dx] * kern[J*KernelSize];
  else
    return im[(y+dy)*width+x+dx] * kern[J*KernelSize+I] + unroll_x<KernelSize, J, I-1>(x, y, im, width, kern);
}

template<int KernelSize, int J>
static inline float unroll_y(
    const int x,
    const int y,
    __global const uchar* __restrict__ im,
    const int width,
    __global const float* __restrict__ kern){
  if constexpr(J == 0)
    return unroll_x<KernelSize, 0, KernelSize-1>(x, y, im, width, kern);
  else
    return unroll_x<KernelSize, J, KernelSize-1>(x, y, im, width, kern)
         + unroll_y<KernelSize, J-1>(x, y, im, width, kern);
}

template<int KernelSize>
static inline float unroll(
    const int x,
    const int y,
    __global const uchar* __restrict__ im,
    const int width,
    __global const float* __restrict__ kern){
  return unroll_y<KernelSize, KernelSize-1>(x, y, im, width, kern);
}

}

template<size_t KernelSize>
static void convolution_unrolled(
    __global const unsigned char* __restrict__ im,
    int width,
    int height,
    __global const float* __restrict__ kern,
    __global unsigned char* __restrict__ output){
  const int x = get_global_id(0);
  const int y = get_global_id(1);

  const int half_k = KernelSize / 2;
  if(y < half_k
  || height - half_k <= y
  || x < half_k
  || width - half_k <= x)
    return;
  output[y*width+x] = (unsigned char)(min(max(detail::unroll<KernelSize>(x, y, im, width, kern), 0.f), 255.f));
}

__kernel void convolution_3x3(
    __global const unsigned char* __restrict__ im,
    int width,
    int height,
    __global const float* __restrict__ kern,
    int,
    __global unsigned char* __restrict__ output){
  convolution_unrolled<3>(im, width, height, kern, output);
}
```

あとは呼び出す際に `kernel_size == 3 ? "convolution_3x3" : "convolution_general"` のような形でカーネルを指定してやることで，カーネルサイズが `3` のときのみ特殊化された実装が呼ばれるようになります．

## まとめ

というわけで，NVIDIA GPU上でOpenCL C++を動かしてみました．
C++が動くとtemplateなどが使えてうれしいですね．

…ほんとにうれしいか？といわれると，まぁ，まずオンラインコンパイルができないし，カーネルはOpenCL C++で書いたがホスト側のコードはガッツリCUDA Driver APIという話もあり…いやまぁOpenCLのドライバにOpenCL C++のコンパイラが載ってないんだから仕方がないんですが，じゃあそれCUDAでいいじゃん！と言われると何も言えないですね…😇

上記コードは[このリポジトリ](https://github.com/wx257osn2/opencl_cxx_conv2d_for_nvidia_gpu)に置いてあるので，お手元で試す際は参考にどうぞ．

---

明日は[onihusube9](https://twitter.com/onihusube9)さんの「ExecutorとNetworking TSで起きていること」です．
