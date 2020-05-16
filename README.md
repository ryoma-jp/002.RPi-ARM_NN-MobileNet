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

2020.5.12 ARMNNのビルドでエラーが出るようになっているので，
https://developer.arm.com/solutions/machine-learning-on-arm/developer-material/how-to-guides/cross-compiling-arm-nn-for-the-raspberry-pi-and-tensorflow
に沿って，インストール  

2020.5.13 変わらずビルドエラー出た  
https://github.com/ARM-software/armnn/issues/355  
の対応が必要で  

cmakeの引数に

	-DCMAKE_CXX_FLAGS=-Wno-error=deprecated-copy

を追加する

	cmake .. -DCMAKE_LINKER=/usr/bin/arm-linux-gnueabihf-ld \
	-DCMAKE_C_COMPILER=/usr/bin/arm-linux-gnueabihf-gcc \
	-DCMAKE_CXX_COMPILER=/usr/bin/arm-linux-gnueabihf-g++ \
	-DCMAKE_C_COMPILER_FLAGS=-fPIC \
	-DCMAKE_CXX_FLAGS=-mfpu=neon \
	-DCMAKE_CXX_FLAGS=-Wno-error=deprecated-copy \
	-DARMCOMPUTE_ROOT=$BASEDIR/ComputeLibrary \
	-DARMCOMPUTE_BUILD_DIR=$BASEDIR/ComputeLibrary/build \
	-DBOOST_ROOT=$BASEDIR/boost \
	-DTF_GENERATED_SOURCES=$BASEDIR/tensorflow-protobuf \
	-DBUILD_TF_PARSER=1 \
	-DPROTOBUF_ROOT=$BASEDIR/protobuf-arm \
	-DARMCOMPUTENEON=1 \
	-DARMNNREF=1
	
これでmakeするとエラーは解消．


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
	+ARMNN_LIB=/usr/local/lib
	+ARMNN_INC=/usr/local/include

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

	$ make test

実行結果

	(armnn) pi@raspberrypi:~/Projects/armnn/work/ML-examples/armnn-mnist $ make test
	LD_LIBRARY_PATH=/home/pi/Projects/armnn/work/armnn-dist/armnn/lib:/home/pi/Projects/armnn/work/armnn-dist/armnn/lib ./mnist_tf
		Predicted: 7
		Actual: 7

## 参考URL

|title|URL|
|:----|:----|
|Configuring the Arm NN SDK build environment for TensorFlow|https://developer.arm.com/solutions/machine-learning-on-arm/developer-material/how-to-guides/configuring-the-arm-nn-sdk-build-environment-for-tensorflow/single-page
|ML Examples|https://github.com/ARM-software/ML-examples|

