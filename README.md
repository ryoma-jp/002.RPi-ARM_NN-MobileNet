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
