---
title: "DPC++はC++ AMPの夢を見るか"
emoji: "🆑"
type: "tech"
topics: ["cpp", "opencl"]
published: true
---

[C++ Advent Calendar 2022](https://qiita.com/advent-calendar/2022/cxx)無事^[[主催のくせに9日近く遅刻したやつ](https://twitter.com/wx257osn2)がいるので無事と呼べるかは怪しい]完走しました！皆様今年もご参加ありがとうございました．
本稿は，C++ AdC 2022 20日目の記事でやろうと思ったが[調査の結果やりたいことができんことがわかり，諦めた](https://twitter.com/wx257osn2/status/1606716867468742656)没ネタについての記事です^[所詮没の記事なので結構雑に書こうと思います]．
というか，有識者各位誰か教えてください！！！！！！！！！！

# DPC++はC++ AMPの夢を見るか

昨年も記事にしましたが[C++ AMPが死にました](https://zenn.dev/wx257osn2/articles/rip_cxx_amp-ksaudhawigubweinfwklaeiuhfawelifh)．
当時の記事にも記載している通り，

- シングルソースで
- 最新のC++の言語機能が使えて
- グラフィックスAPIとのinteropがあり
- デバイス非依存で
- 突然死なない

APIを探し求めている^[具体的な行動は特にしてない．というかこの記事をC++ AdCの記事とすることで締切駆動進捗をやろうとした結果失敗した]わけですが，やはり先の記事でも述べた通りSYCLを本命に据えているわけですね．
というわけで，SYCLの1実装であるIntel oneAPI DPC++が上記のような振る舞いをできるのか調べてみました^[というわけで記事タイトルの「C++ AMP」は「上記のような性質を持ったAPI」のことを指しており，DPC++でC++ AMPを書こう！という意味ではありません．あしからず]．

## やりたいこと

`SYCL <-> バックエンドAPI <-> グラフィックスAPI` のような形でそれぞれのAPIをinteropし，SYCLからグラフィックスAPIのリソースを入出力とした演算をかけることが目標です．
バックエンドAPI(CUDAやOpenCLなど)とグラフィックスAPI(ここではDirectX)のinteropは概ね問題なく動作するはずなので，SYCLとバックエンドAPIのinteropについて調べます．
問題なく動作したらグラフィックスAPIと接続してみます．

### SYCLとバックエンドのinterop

SYCLは🆑の名を冠していますが，その実バックエンドにOpenCLは要求しておらず，実際にCUDAバックエンドなどの実装が存在します．
そのためSYCLではバックエンドとの汎用的なinterop APIが提供されており，これを用いることでSYCLのリソースからバックエンドのリソースを抜き出したり，逆にバックエンドのリソースからSYCLのリソースを生成したりできるわけですね．
DPC++はOpenCLバックエンドを実装していますので， `sycl::make_なんとか<cl::sycl::backend::opencl>(OpenCLのリソース)` みたいなコードを書くといい感じにOpenCLのリソースをSYCLに持ち込めます．
というわけで，以下のようなコードを用意します．

```cpp
#define CL_USE_DEPRECATED_OPENCL_1_2_APIS
#include<CL/sycl.hpp>
#include<CL/cl.h>
#include<stdexcept>
#include<memory>
#include<type_traits>
#include<vector>
#include<iostream>

auto create_cl_context(::cl_device_id device_id){
	cl_int ret;
	std::unique_ptr<std::remove_pointer_t<::cl_context>, decltype(&::clReleaseContext)> context = {::clCreateContext(nullptr, 1, &device_id, nullptr, nullptr, &ret), &::clReleaseContext};
	if(ret != CL_SUCCESS)
		throw std::runtime_error("create_cl_context failed");
	return context;
}

auto create_cl_command_queue(::cl_context context, ::cl_device_id device_id){
	cl_int ret;
	std::unique_ptr<std::remove_pointer_t<::cl_command_queue>, decltype(&::clReleaseCommandQueue)> queue = {::clCreateCommandQueue(context, device_id, 0, &ret), &::clReleaseCommandQueue};
	if(ret != CL_SUCCESS)
		throw std::runtime_error("create_cl_command_queue failed");
	return queue;
}

auto create_cl_buffer(::cl_context context, ::cl_mem_flags flags, std::size_t size){
	cl_int ret;
	std::unique_ptr<std::remove_pointer_t<::cl_mem>, decltype(&::clReleaseMemObject)> buffer = {::clCreateBuffer(context, flags, size, nullptr, &ret), &::clReleaseMemObject};
	if(ret != CL_SUCCESS)
		throw std::runtime_error("create_cl_buffer failed");
	return buffer;
}

void write_buffer(::cl_command_queue queue, ::cl_mem obj, std::size_t size, const void* ptr){
	if(::clEnqueueWriteBuffer(queue, obj, CL_TRUE, 0, size, ptr, 0, nullptr, nullptr) != CL_SUCCESS)
		throw std::runtime_error("write_buffer failed");
}

void read_buffer(::cl_command_queue queue, ::cl_mem obj, std::size_t size, void* ptr){
	if(::clEnqueueReadBuffer(queue, obj, CL_TRUE, 0, size, ptr, 0, nullptr, nullptr) != CL_SUCCESS)
		throw std::runtime_error("read_buffer failed");
}

int main()try{
	const auto device_id = /* ここにOpenCLのデバイスID */;
	auto device = sycl::make_device<cl::sycl::backend::opencl>(device_id);
	auto platform = device.get_platform();
	std::cout << platform.get_info<sycl::info::platform::name>() << std::endl;
	std::cout << device.get_info<sycl::info::device::name>() << std::endl;
	const auto cl_context = create_cl_context(device_id);
	auto context = sycl::make_context<cl::sycl::backend::opencl>(cl_context.get());
	const auto cl_queue = create_cl_command_queue(cl_context.get(), device_id);
	auto queue = sycl::make_queue<cl::sycl::backend::opencl>(cl_queue.get(), context);

	const std::vector<float> a(3, 42.f);
	const std::vector<float> b = {0.1f, 0.2f, 0.3f};
	std::vector<float> c(3);
	auto cl_a = create_cl_buffer(cl_context.get(), CL_MEM_READ_ONLY, sizeof(float) * a.size());
	auto cl_b = create_cl_buffer(cl_context.get(), CL_MEM_READ_ONLY, sizeof(float) * b.size());
	auto cl_c = create_cl_buffer(cl_context.get(), CL_MEM_WRITE_ONLY, sizeof(float) * c.size());
	write_buffer(cl_queue.get(), cl_a.get(), sizeof(float) * a.size(), a.data());
	write_buffer(cl_queue.get(), cl_b.get(), sizeof(float) * b.size(), b.data());
	auto ba = sycl::make_buffer<cl::sycl::backend::opencl, float, 1>(cl_a.get(), context);
	auto bb = sycl::make_buffer<cl::sycl::backend::opencl, float, 1>(cl_b.get(), context);
	auto bc = sycl::make_buffer<cl::sycl::backend::opencl, float, 1>(cl_c.get(), context);
	queue.submit([&](sycl::handler& cgh){
		auto acca = ba.get_access<sycl::access_mode::read>(cgh);
		auto accb = bb.get_access<sycl::access_mode::read>(cgh);
		auto accc = bc.get_access<sycl::access_mode::write>(cgh);
		cgh.parallel_for(sycl::range<1>{a.size()}, [=](auto idx){
			accc[idx] = acca[idx] + accb[idx];
		});
	});
	queue.wait();
	read_buffer(cl_queue.get(), cl_c.get(), sizeof(float) * c.size(), c.data());
	for(auto x : c)
		std::cout << x << std::endl;
}catch(const std::exception& e){
	std::cerr << "Exception: " << e.what() << std::endl;
}
```

で，あとは `device_id` 変数に適当なデバイスIDを突っ込むだけですね．
まずはプログラム全体が正しく動くことを確認するために，CPUのデバイスIDを突っ込んでみましょう．

![実行結果1](/images/does_dpcxx_dream_of_cxx_amp-oiansdlwiubg/01_sycl_on_cpu.png)

問題なさそうです．では次に，NVIDIA GPUのデバイスIDを突っ込んでみましょう．

![実行結果2](/images/does_dpcxx_dream_of_cxx_amp-oiansdlwiubg/02_sycl_on_gpu.png)

死にました^[具体的には `cgh.parallel_for` でカーネル生成しようとしたタイミングでコケます．想定していないOpenCLバックエンドが来たみたいな顔してんな]．

## まとめ

どうやらこの方針では上手く行かなさそうなので，以下のいずれかを試す必要がありそうです．

- ComputeCppを試す
    - アカウント作るのめんどくせぇっつって放置してます
- DPC++のCUDAバックエンドを有効化して試す
    - 現状OpenCLでNVIDIA GPUを扱うのは無理っぽいので，大人しくCUDAサポートに乗っかる
        - デバイスに依存しちゃうけどまぁ致し方なし…interopの部分さえ置き換えてやれば他は使いまわせるし多少はね？
        - ちなみに何故NVIDIA GPUを扱えないのかについては，もしかするとCUDAのOpenCLが未だに1.2なのが悪いかもしれない？(真面目に調査してないのでわからず)
    - ところで[CUDAバックエンドは現状Linuxでしかサポートされていない](https://intel.github.io/llvm-docs/GetStartedGuide.html#cuda-back-end-limitations)…という記述と，[サポートしとるで](https://intel.github.io/llvm-docs/GetStartedGuide.html#build-dpc-toolchain-with-support-for-nvidia-cuda)という記述がある．どっちやねん
        - > Backend is only supported on Linux
        - > Note, the CUDA backend has Windows support; windows subsystem for linux (WSL) is not needed to build and run the CUDA backend.
    - し，[自前のビルドが必要](https://intel.github.io/llvm-docs/GetStartedGuide.html#build-dpc-toolchain-with-support-for-nvidia-cuda) え，Windowsでこれやんのまじ？めんどくさくない？

**OpenCLバックエンドでよしなにやる方法，あるいはなんでCUDAのOpenCLバックエンドだとコケるのかご存じの方がいらっしゃいましたら是非ご教示お願いします**^[これのために記事書いた．たすけて]．

というわけで，次回^[_いつ？_]，DPC++をビルドしてCUDAバックエンドを試す． **俺達の満足は，これからだ！！** ^[という残念な結果に終わったので，これがAdCの記事ってのもなぁ…となり[20日目の記事を書くに至る](https://zenn.dev/wx257osn2/articles/constexpr_variant-jdbnlwiberawejbf)]