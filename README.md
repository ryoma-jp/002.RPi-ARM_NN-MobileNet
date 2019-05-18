# 002.RPi-ARM_NN-MobileNet: Raspberry Pi上でARM NN上でMobileNetを動かす

## 環境

* RaspberryPi 3 Model B+
* ARM NN: https://github.com/ARM-software/armnn

## 手順

### Raspberry Piの準備

1. Raspbian Stretch with desktopをダウンロード  
ダウンロード後，解凍する  
https://www.raspberrypi.org/downloads/raspbian/
1. SDカードをSD Card FormatterでFAT32でフォーマットする  
https://www.sdcard.org/jp/downloads/formatter/
1. SDカードにRaspbian Stretch with desktopを書き込む  
Win32 Disc Imagerでimgファイルを指定してWriteする  
https://forest.watch.impress.co.jp/docs/review/1067836.html
1. Raspberry Piの初期設定を行う(sudo raspi-config)  
下記設定後，再起動するかを聞かれるので，Yesを選択して再起動する  
	* raspi-configの更新  
	→「8 Update」を選択
	* ロケールの設定  
	→「4 Localisation Options」を選択
		* I1 Change Locale：ja_JP.UTF-8 UTF-8
		* I2 Change Timezone：Asia → Tokyoを選択
		* I3 Change Keyboard Layout：Generic 105-key (Intl) PC → Other → Japanese → Japanese (OADG 109A) → The default for the keyboard layout → No compose keyを選択
		* I4 Change Wi-fi Country：JP Japanを選択
	* SSH serverの有効化  
	→「5 Interfacing Options」→「P2 SSH」→「Yes」を選択し，piユーザのパスワードを変更する
	* ファイルシステムの拡張  
	→「7 Advanced Options」→「A1 Expand Filesystem」を選択
1. パッケージ，OS，ファームウェアの更新

		$ sudo apt-get update
		$ sudo apt-get upgrade
		$ sudo apt-get dist-upgrade
		$ sudo apt-get install rpi-update
		$ sudo rpi-update
		$ sudo reboot

### ARM NNの準備

Arm NN SDKのインストールに必要なパッケージのインストール

	$ sudo apt-get install scons cmake autoconf automake libtool curl make g++ unzip

armnnとComputeLibraryを取得

	$ export BASEDIR=`pwd`
	$ git clone https://github.com/Arm-software/armnn
	$ git clone https://github.com/ARM-software/ComputeLibrary

BOOSTをインストール

	$ wget https://dl.bintray.com/boostorg/release/1.64.0/source/boost_1_64_0.tar.bz2
	$ tar xf boost_1_64_0.tar.bz2
	$ cd ${BASEDIR}/boost_1_64_0/tools/build
	$ ./bootstrap.sh
	$ ./b2 install --prefix=${BASEDIR}/boost.build
	$ export PATH=${BASEDIR}/boost.build/bin:$PATH
	$ cp ${BASEDIR}/boost_1_64_0/tools/build/example/user-config.jam ${BASEDIR}/boost_1_64_0/project-config.jam
	$ echo "using gcc : arm : arm-linux-gnueabihf-g++-6 ;" >> ${BASEDIR}/boost_1_64_0/project-config.jam

ProtoBufをインストール

	$ cd ${BASEDIR}
	$ git clone -b v3.5.0 https://github.com/google/protobuf.git
	$ cd ${BASEDIR}/protobuf
	$ git submodule update --init --recursive
	$ ./autogen.sh
	$ ./configure
	$ make
	$ make check
	$ sudo make install
	$ sudo ldconfig
	$ export LD_LIBRARY_PATH="/usr/local/lib;${LD_LIBRARY_PATH}"

TensorFlowのコードを取得（ビルドは不要）

	$ cd ${BASEDIR}
	$ git clone https://github.com/tensorflow/tensorflow.git

ComputeLibraryをビルド

	$ cd ${BASEDIR}/ComputeLibrary
	$ scons extra_cxx_flags="-fPIC" benchmark_tests=0 validation_tests=0 neon=1

Arm NNをビルド

	$ pushd tensorflow 
	$ ${BASEDIR}/armnn/scripts/generate_tensorflow_protobuf.sh ${BASEDIR}/tensorflow_protobuf /usr/local
	$ popd
	$ cd ${BASEDIR}/armnn
	$ mkdir build 
	$ cd build 
	$ cmake .. -DARMCOMPUTE_ROOT=${BASEDIR}/ComputeLibrary -DARMCOMPUTE_BUILD_DIR=${BASEDIR}/ComputeLibrary/build -DBOOST_ROOT=${BASEDIR}/boost_1_64_0 -DTF_GENERATED_SOURCES=${BASEDIR}/tensorflow_protobuf -DBUILD_TF_PARSER=1 -DPROTOBUF_ROOT=/usr/local -DARMCOMPUTENEON=1
	$ make
	$ ./UnitTests

ML-examplesを動かして動作確認

	$ cd ${BASEDIR}
	$ git clone https://github.com/ARM-software/ML-examples
	$ cd ${BASEDIR}/ML-examples/armnn-mnist

Makefileを下記のように修正

	diff --git a/armnn-mnist/Makefile b/armnn-mnist/Makefile
	index e9ddfbb..9c1b02c 100644
	--- a/armnn-mnist/Makefile
	+++ b/armnn-mnist/Makefile
	@@ -1,7 +1,10 @@
	-ARMNN_LIB = ${HOME}/devenv/build-x86_64/release/armnn
	-ARMNN_INC = ${HOME}/devenv/armnn/include
	+#ARMNN_LIB = ${HOME}/devenv/build-x86_64/release/armnn
	+#ARMNN_INC = ${HOME}/devenv/armnn/include
	+ARMNN_LIB=${BASEDIR}/armnn/build
	+ARMNN_INC=${BASEDIR}/armnn/include

	-all: mnist_caffe mnist_tf
	+
	+all: mnist_tf

	 mnist_caffe: mnist_caffe.cpp mnist_loader.hpp
	        g++ -O3 -std=c++14 -I$(ARMNN_INC) mnist_caffe.cpp -o mnist_caffe -L$(ARMNN_LIB) -larmnn -larmnnCaffeParser
	@@ -12,6 +15,5 @@ mnist_tf: mnist_tf.cpp mnist_loader.hpp
	 clean:
	        -rm -f mnist_tf mnist_caffe

	-test: mnist_caffe mnist_tf
	-       LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:$(ARMNN_LIB) ./mnist_caffe
	-       LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:$(ARMNN_LIB) ./mnist_tf
	+test: mnist_tf
	+       LD_LIBRARY_PATH="${LD_LIBRARY_PATH};$(ARMNN_LIB)" ./mnist_tf

ML-examplesをビルド

	$ ARMNN_LIB=${BASEDIR}/armnn/build
	$ ARMNN_INC=${BASEDIR}/armnn/include
	$ make test


## 参考URL

|title|URL|
|:----|:----|
|Configuring the Arm NN SDK build environment for TensorFlow|https://developer.arm.com/solutions/machine-learning-on-arm/developer-material/how-to-guides/configuring-the-arm-nn-sdk-build-environment-for-tensorflow/single-page
|ML Examples|https://github.com/ARM-software/ML-examples|

