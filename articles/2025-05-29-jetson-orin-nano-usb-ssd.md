---
title: "Jetson Orin NanoをUSB接続のSSDから起動する"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["jetson" ]
published: true
---

microSDカードで起動を確認したJetson Orin NanoをUSB接続のSSDから起動するまでの手順をまとめます。

![](/images/jetson-orin-nano-usb-ssd.jpg)

この方法の便利なところは、SDK Managerが不要なところです。
ext4フォーマットのパーティションのファイルの読み書きが必要なので、UbuntuなどのLinux環境は必要です。
（最近はWindows11でもext4フォーマットのパーティションのファイルの読み書きが可能なようですが未調査）

## 動作確認環境

* Jetson Orin Nano Super
* [JetPack 6.2](https://developer.nvidia.com/embedded/jetpack-sdk-62)
* microSD 64GB
* USB接続のSSD（[SSD-PSM960U3-MB](https://www.buffalo.jp/product/detail/ssd-psm960u3-mb.html)）


## 手順

### 1. microSDカードで起動

公式のドキュメントに従って、microSDカードで起動できるようになるところまで進めます。

https://www.jetson-ai-lab.com/initial_setup_jon.html

何段階かに分けてアップデートするのがポイントのようです。

UEFI Firmware version 3.0（JetPack 5.xのときのやつ）
→ JetPack 5.1.3を使ってUEFI Firmware version 5.0にアップデート
→ JetPack 5.1.3を使ってUEFI Firmware version 36.3.0にアップデート
→ JetPack 6.2を使ってUEFI Firmware version 36.4.3にアップデート

https://x.com/Tiryoh/status/1917168926548758743

https://x.com/Tiryoh/status/1917169325318045780

### 2. USB接続のSSDにmicroSDカードの内容をコピー

PCにmicroSDカードとUSB接続のSSDを両方とも接続して、microSDカードの内容をUSB接続のSSDにコピーします。

balenaEtcherにはSDカードのclone機能があるので、これを使ってコピーしました。ddコマンド等でもコピーできると思います。

### 3. USB接続のSSDのファイルを編集

`/boot/extlinux/extlinux.conf`を編集して、ストレージを変更します

root=/dev/mmcblk0p1 -> root=/dev/sda1

```diff:/boot/extlinux/extlinux.conf
TIMEOUT 30
DEFAULT primary

MENU TITLE L4T boot options

LABEL primary
      MENU LABEL primary kernel
      LINUX /boot/Image
      INITRD /boot/initrd
-     APPEND ${cbootargs} root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 mminit_loglevel=4 console=ttyTCU0,115200 firmware_class.path=/etc/firmware fbcon=map:0 nospectre_bhb video=efifb:off console=tty0 
+     APPEND ${cbootargs} root=/dev/sda1 rw rootwait rootfstype=ext4 mminit_loglevel=4 console=ttyTCU0,115200 firmware_class.path=/etc/firmware fbcon=map:0 nospectre_bhb video=efifb:off console=tty0 

# When testing a custom kernel, it is recommended that you create a backup of
# the original kernel and add a new entry to this file so that the device can
# fallback to the original kernel. To do this:
#
# 1, Make a backup of the original kernel
#      sudo cp /boot/Image /boot/Image.backup
#
# 2, Copy your custom kernel into /boot/Image
#
# 3, Uncomment below menu setting lines for the original kernel
#
# 4, Reboot

# LABEL backup
#    MENU LABEL backup kernel
#    LINUX /boot/Image.backup
#    INITRD /boot/initrd
#    APPEND ${cbootargs}
```

### 4. 起動

Jetson Orin NanoにUSB接続のSSDを接続して起動します（microSDカードは取り外しておきます）。

## 注意

M.2 SlotでNVMe接続のSSDの場合はおそらく`/boot/extlinux/extlinux.conf`を編集する際のパスが変わると思います（未確認）。

## 参考

https://www.reddit.com/r/JetsonNano/comments/1hth1vo/booting_jetson_orin_nano_super_from_ssd/
