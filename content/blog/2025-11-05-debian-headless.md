---
title: "Debian Headless Install"
date: 2025-11-05T00:00:00+09:00
archives:
    - 2025-11
categories: ["blog"]
tags: ["note", "linux"]
draft: false
---
# はじめに
Debianのheadlessインストールを行います. ネットワーク接続以外はキーボード, マウスやディスプレイは接続されていないＰＣにインストールします.

## 手順

- インストーラ作成
  1. Debianの[`netinst.iso`](https://www.debian.org/CD/netinst/)をダウンロード
  2. isoを解凍して, インストーラが自動で進むように設定を書き換え
  3. インストーラ起動後にSSHサーバを起動するように設定
  4. 

# インストーラ作成

1. [`netinst.iso`](https://www.debian.org/CD/netinst/)をダウンロード
```bash
$ wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-x.x.x-amd64-netinst.iso
```

2. isoを解凍して, インストーラが自動で進むように設定を書き換え
```bash
$ sudo apt -y install xorriso -extract / isofiles
$ xorriso -osirrox on -indev debian-x.x.x-amd64-netinst.iso
```

Grubのtimeoutを1秒に変更する.
```txt:iso/isolinux/isolinux.cfg
# D-I config version 2.0
# search path for the c32 support libraries (libcom32, libutil etc.)
path
prompt 0
# continue to next in 1 second
timeout 1
include menu.cfg
default vesamenu.c32
```

```txt:iso/isolinux/gtk.cfg
default installgui
label installgui
    menu label ^Graphical install
    menu default
    kernel /install.amd/vmlinuz
    append vga=788 auto=true priority=critical file=/cdrom/preseed.cfg initrd=/install.amd/gtk/initrd.gz --- quiet 
# auto=true set locale, keyboard in preseed
# priority=critical suppress low priority questions
# file=/cdrom/preseed.cfg locate preseed.cfg
```

preseed.cfgファイルを作成して, 静的IPを設定する.
```bash
$ wget https://www.debian.org/releases/trixie/example-preseed.txt
$ sudo mv ./example-preseed.txt ./iso/preseed.cfg
```

```txt:iso/preseed.cfg
# IPv4 example
d-i netcfg/get_ipaddress string 192.168.128.147
d-i netcfg/get_netmask string 255.255.255.0
d-i netcfg/get_gateway string 192.168.128.1
d-i netcfg/get_nameservers string 192.168.128.1
d-i netcfg/confirm_static boolean true

### Network console
# Use the following settings if you wish to make use of the network-console
# component for remote installation over SSH. This only makes sense if you
# intend to perform the remainder of the installation manually.
d-i anna/choose_modules string network-console
#d-i network-console/authorized_keys_url string http://10.0.0.1/openssh-key
d-i network-console/password password r00tme
d-i network-console/password-again password r00tme
```

ファイルのMD5を再計算する.
```bash
$ pushd iso
$ chmod 666 md5sum.txt
$ find -follow -type f -exec md5sum {} \; > md5sum.txt
$ chmod 444 md5sum.txt
$ popd
```

ISOファイルを作成する.
```
$ sudo apt install isolinux
$ xorriso -as mkisofs -o debian-13.1.0-amd64-netinst-headless.iso \
        -isohybrid-mbr /usr/lib/ISOLINUX/isohdpfx.bin \
        -c isolinux/boot.cat -b isolinux/isolinux.bin -no-emul-boot \
        -boot-load-size 4 -boot-info-table iso
```

ISOファイルをUSBに書き込みます. `lsblk`でブロックデバイスを確認しておきます.
```bash
$ sudo dd if=debian-13.1.0-amd64-netinst-headless.iso of=/dev/sdb
```