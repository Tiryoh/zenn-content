---
title: "JetsonでROS1/ROS2を使う際のCUDA互換性問題と解決策"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ROS1", "ROS2", "ROS", "CUDA", "Jetson"]
published: false
---

:::message
2025年7月時点での情報です。最新の情報についてはフォーラムや公式ドキュメントを参照してください。  
ミスや誤植があればコメントやSNS等でご指摘いただけると幸いです。
:::

## はじめに

古い世代のJetsonでROS 1/ROS 2を動かそうとした際、CUDAの互換性問題やJetPackの制約に直面しました。特に、新しいROS 2 Jazzyと古いCUDAツールキットの組み合わせで発生する問題の解決策を探る中で、Jetson特有の技術的な制約について理解を深める機会がありました。

この記事では、JetsonでのROS 1/ROS 2環境構築における技術的な課題と解決策について、調査結果と手元のJetsonでの動作確認結果をまとめます。
なお、この記事では実装の詳細には触れません。

NVIDIAのソフトウェア関連は名称が変わることもあるので、基本的なところから整理しています。

## CUDAツールキットとドライバの関係性

### 基本構成

NVIDIAのハードウェアを使うためのソフトウェアは、CUDAツールキット（ライブラリ）とNVIDIAドライバの2つに分かれています。

![](/images/cuda-components.png)

https://docs.nvidia.com/deploy/cuda-compatibility/index.html より引用

#### 重要な特徴

- **CUDAツールキット**：GPUが存在しないハードウェアでもビルド等には使用可能
  - Apple Silicon搭載のMacBook上のDockerなどでも利用できる
- **CUDAドライバ**：基本的にアプリケーションに同梱できない
- **互換性の制約**：実行環境のCUDAドライバが古く、ランタイムが要求するドライバAPIをサポートしていなければ、アプリケーションは動作しない

#### ソフトウェアスタック

CUDAアプリケーションは以下の階層構造で動作しています。

```
アプリケーションコード → CUDAランタイム API → CUDAドライバ API → GPUハードウェア
```

この依存関係のため、新しいCUDAランタイムを静的リンクしても、実行環境のCUDAドライバが古ければランタイムが要求するドライバAPIをサポートせずアプリケーションは動作しません。

## JetPack

### JetPackの構成

NVIDIAのSBCであるJetsonは「JetPack」と呼ばれるソフトウェア群を導入して動作します。

**JetPack = OS + NVIDIAドライバ + CUDAツールキット + その他諸々**

- その他諸々
  - Jetson用のNVIDIA Container Toolkit（旧nvidia-docker2）
  - OpenCV
  - その他NVIDIA製ライブラリ

### Jetson Linux（OS）

基本特徴は以下の通りです。

- **UbuntuベースのカスタムOS**
- Jetson対応のためにNVIDIAがカスタマイズ
  - > NVIDIA® Jetson™ Linux Driver Package is the board support package for Jetson. It includes Linux Kernel, UEFI bootloader, NVIDIA drivers, flashing utilities, sample filesystem based on Ubuntu, and more for the Jetson platform.
    - https://developer.nvidia.com/embedded/jetson-linux
- 昔（JetPack 4まで）は「L4T」と呼ばれていた
  - https://jetsonhacks.com/2022/08/15/jetpack-5-production-release-now-live/

残念ながら使う上で少し不便な制約があります。

- **Ubuntuのリリースから2年遅れて公開**
  - Ubuntu 22.04ベースのJetson Linuxは2024年に公開
  - OSバージョンの問題で、アプリケーション開発の際にはDockerやパッケージマネージャによるバージョン切り替えが多用される
    - NVIDIA公式のサンプルもDockerを前提としたものが多数
- **ホストOSのアップグレードが困難**
  - 通常のPCでは`do-release-upgrade`コマンドなどでUbuntuのバージョンを簡単に上げられるが、Jetsonでは困難
  - ハードウェアが特定のバージョンのOSに紐付いているため、ライブラリバージョンも事実上固定される
  - 仮想環境やコンテナである程度新しいバージョンにも追従可能だが、ホストOS自体の変更は基本的に不可
  - 古いOSではglibcのバージョンが古いため、Dockerコンテナ実行時などでエラーが発生する（ことがある）
    - Dockerオプションで回避可能（ただしセキュリティ観点では推奨されない）

### Jetson用NVIDIA Container Toolkit

#### PC用GPUとの違い

Jetson用のNVIDIA Container Toolkitは、PC用GPUに使うNVIDIA Container Toolkitとは異なる特殊なソフトウェアです。

Dockerコンテナ起動時に**ホストOSからNVIDIAドライバやCUDAツールキットがマウントされます**

- JetPack 4まで：NVIDIAドライバとCUDAツールキットがマウントされる
- JetPack 5から：NVIDIAドライバのみがマウントされる

#### L4T Baseイメージ

NVIDIAが公式で公開している[L4T Baseイメージ](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/l4t-base)は、JetsonのGPUにアクセスできるようDockerfile内で予め設定済みです。

- JetPackと同じバージョンのUbuntuをベースに作成
- 主にEGL（Embedded-System Graphics Library）関連が設定済み

https://gitlab.com/nvidia/container-images/l4t-base

### JetPackの書き込み方法

JetPackの書き込み方法は以下の2つがあります。

1. [**SDK Manager**](https://developer.nvidia.com/sdk-manager)経由
   - Ubuntu用アプリケーション
     - Windows版SDK Managerも存在するが、JetPackの書き込みには未対応（2025年7月時点）
   - PCとJetsonをUSB接続して書き込み
2. **microSDカードへのFlash**
   - microSDカード対応モデル（Jetson Orin Nanoなどエントリーモデル）のみ
   - イメージファイルをmicroSDカードに書き込み

同じモデルのJetsonであれば、microSDカードやSSDをクローンして動作させることも可能です。

## Compute CapabilityとCUDA

### Compute Capability

NVIDIAのGPUやJetsonには「Compute Capability」という概念があります。
Pascal、Turing、Ampereなどのコードネームで呼ばれているアーキテクチャごとに異なります。

![](/images/jetson-codename.png)

https://www.macnica.co.jp/business/semiconductor/articles/nvidia/133162/ より引用

#### 各ハードウェアのCompute Capability

- **7.2**：Jetson AGX Xavier、Jetson Xavier NX
- **8.7**：Jetson AGX Orin、Jetson Orin NX、Jetson Orin Nano

詳細はNVIDIAの公式ページにまとまっています。

https://developer.nvidia.com/cuda-gpus

#### 世代による制約

- 新しい世代のハードウェアになるほどCompute Capabilityの数値が大きい
- Compute Capabilityが上がると、必要なCUDAツールキットのバージョンも上がる

### ビルド時の制約

新しいハードウェア対応バイナリをビルドするには、対応バージョンのCUDAツールキットとGCCが必要です。

例えば、Jetson Orin NXにはJetPack 5以降に同梱されているCUDAツールキットが必要になります。

https://developer.nvidia.com/embedded/jetpack-archive

#### 重要な制約

- CUDAツールキットリリース時点で未発表のハードウェア向けバイナリは**原則ビルドできない**
- 新しいCUDAツールキットでは、古いハードウェアへのサポートが終了することもある
- CUDAツールキットに対応するGCCのバージョンは限定的
  - CUDA 10.0：GCC 7
  - CUDA 10.2：GCC 8
  - CUDA 11.4：GCC 11
  - CUDA 12.6：GCC 13

https://gist.github.com/ax3l/9489132

## CUDA用のバイナリの作成

### コンパイルプロセス

`nvcc`コマンドでコンパイルします。実行時に以下のオプションを指定します。

- `arch`：PTXのターゲット
  - **PTX**：NVIDIAのGPU向け仮想アセンブリ言語（**P**arallel **T**hread e**X**ecution）
- `code`：SASSのターゲット
  - **SASS**：GPUアーキテクチャに最適化されたバイナリコード（**S**treaming **Ass**embler）

CUDAコードのコンパイルは以下の流れで行われます。

```
CUDAコード → PTX（中間表現） → SASS（バイナリコード）
```


https://zenn.dev/eduidl/articles/cuda-comparability

PTXは仮想アセンブリ言語とのことで、このアセンブリを直接書くことも想定されているようです。

https://docs.nvidia.com/cuda/archive/11.4.4/ptx-writers-guide-to-interoperability/index.html


### バイナリ互換性の問題

#### アーキテクチャ固有の最適化

バイナリはアーキテクチャごとに最適化されており、**互換性はありません**。

##### 具体例

CUDA 10.0でCompute Capability 7.2向けにビルドしたバイナリの場合

- **CUDA 10.0のJetson AGX Xavier**：動作する
- **CUDA 11.4のJetson Orin NX**：動作しない

#### API互換性の問題

CUDAのAPIはバージョンごとに変更が加えられるため、さらに複雑な問題が発生します。

##### 具体例

CUDA 10.0でCompute Capability 7.2向けにビルドしたバイナリは、Compute Capability 7.2のJetson AGX XavierでもCUDA 10.2環境では`insufficient driver`エラーが発生して動作しません。

### PTXの前方互換性

#### 中間表現の利点

**中間表現であるPTXには前方互換性があります**。

- 基本的に新しい世代の環境では、古い世代をターゲットとしてビルドされたPTXを実行可能
- JIT（Just-In-Time）コンパイルが発生するため、パフォーマンスに影響が出る可能性はある

##### 具体例

CUDA 10.0でCompute Capability 7.2向けにコンパイルしたPTXは以下すべてで動作します。

- CUDA 10.0のJetson AGX Xavier
- CUDA 10.2のJetson AGX Xavier
- CUDA 11.4のJetson Orin NX

#### 制約事項

**世代固有の機能**を使う場合は他の世代では動作しません。

## 開発環境構築における課題例と解決策

### 主な課題

#### OSバージョンの制約

古いUbuntuでROSをコンパイルする選択肢もありますが、パッケージ入れ替えのたびに長時間のビルドが必要になる点でメンテナンスコストが大きくなりがちです。
[dusty-nv/jetson-containers](https://github.com/dusty-nv/jetson-containers)ではそのような選択がされることもあります。

https://github.com/dusty-nv/jetson-containers

#### GCCのバージョン制約

例えばCUDA 10.0を使う場合は、GCC 7までしかサポートしていません。
ROS 2 Jazzyで使う場合はUbuntu 24.04でGCC 7を使うことになります。

新しいlibstdc++に依存するROS 2 Jazzyのライブラリをリンクすると、ABI（Application Binary Interface）の非互換性で下記のようなリンクエラーが発生します。

```
/usr/bin/ld: /opt/ros/jazzy/lib/librclcpp.so: undefined reference to `std::condition_variable::wait(std::unique_lock<std::mutex>&)@GLIBCXX_3.4.30'
/usr/bin/ld: /opt/ros/jazzy/lib/librcl_logging_spdlog.so: undefined reference to `std::filesystem::__cxx11::path::_M_split_cmpts()@GLIBCXX_3.4.26'
collect2: error: ld returned 1 exit status
```

#### L4T Baseイメージの制約

Ubuntu 24.04を使ったL4T Baseイメージは存在しません。また、Ubuntu 22.04を使ったL4T BaseイメージはJetPack 6相当のもののみとなり、古いハードウェアではCUDAツールキットのバージョンが合いません。

https://catalog.ngc.nvidia.com/orgs/nvidia/containers/l4t-base

### 解決策

#### 共有ライブラリとして分離

ROSに依存しない、CUDA APIを利用する処理のみを独立させた共有ライブラリを作成します。
その共有ライブラリは実行環境の差異を吸収させるため、CUDAランタイムに対して動的リンクを使用します。
この共有ライブラリはROSに依存しないため、ROS 1/ROS 2の両方で使用できます。

#### カスタムL4T Baseイメージ

Ubuntu 22.04/24.04に対してL4T Baseイメージと同様にEGL関連の設定をします。
[@atinfinity](https://github.com/atinfinity)さんの [l4t-ros2-docker](https://github.com/atinfinity/l4t-ros2-docker) や [l4t-base-docker](https://github.com/atinfinity/l4t-base-docker) が参考になります。

https://github.com/atinfinity/l4t-base-docker/blob/main/ubuntu2404/Dockerfile


https://x.com/dandelion1124/status/1841401640068518322


## まとめ

### 調査結果

JetsonでのROS1/ROS2環境構築では、以下の技術的な制約と解決策を理解しておくと役立ちます。
これらの理解により、古い世代のJetsonでもROS 2を効率的に運用できるようになります。特に、CUDA処理を独立したライブラリとして分離するアプローチは、他の組み込みシステムでも応用できそうです。

#### 主要な制約

1. **CUDAツールキットとドライバの依存関係**
   - アプリケーションは実行環境のCUDAドライバに依存する
   - 新しいCUDAランタイムを静的リンクしても古いドライバでは動作しない

2. **JetPackの制約**
   - ハードウェアとOSバージョンが紐付けられている
   - Ubuntu 24.04ベースのL4T Baseイメージが存在しない

3. **Compute Capabilityによる互換性問題**
   - バイナリはアーキテクチャ固有で互換性がない
   - PTXは前方互換性があるが、世代固有機能は使用不可

4. **GCCバージョンの制約**
   - CUDAツールキットごとにサポートするGCCバージョンが限定的
   - ROS 2 Jazzyと古いCUDAの組み合わせでABI非互換が発生

#### 実践的な解決策

1. **共有ライブラリの分離**
   - CUDA処理をROSに依存しない共有ライブラリとして分離
   - 動的リンクにより実行環境の差異を吸収

2. **カスタムL4T Baseイメージの活用**
   - Ubuntu 24.04にEGL関連設定を追加
   - 既存のプロジェクトを参考に環境構築


## 参考資料

https://docs.nvidia.com/deploy/cuda-compatibility/index.html

https://www.jetson-ai-lab.com/initial_setup_jon.html

https://docs.nvidia.com/cuda/cuda-for-tegra-appnote/

https://acoustype.com/2025/01/29/cuda%e3%82%92%e5%9b%9e%e9%81%bf%e3%81%97%e3%81%a6ptx%e3%83%97%e3%83%ad%e3%82%b0%e3%83%a9%e3%83%9f%e3%83%b3%e3%82%b0%e3%82%92%e8%a1%8c%e3%81%86%e3%81%a8%e3%81%af%ef%bc%9f/

https://gist.github.com/Tiryoh/2b5dc1f86ab6bc3e6922e97a4c1ac2f2
