---
title: "NixOS Linux サスペンド状態からの USB/Bluetooth キーボード/マウス操作での復帰設定"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "NixOS", "Linux", "udev", "Bluetooth", "USB" ]
published: false
---

## はじめに
NixOS をインストールしたノートPCで Wayland コンポジター niri を使用しています。

普段は外部モニターを接続し、クラムシェルモードで、USB 接続の有線マウスと Bluetooth キーボードを接続して使用しています。

これまで、 ノートPCがサスペンド状態に入った後、有線マウスやBluetoothキーボードの操作をしてもサスペンド状態から復帰せず、わざわざ、ノートPCの蓋を開けてサスペンドから復帰させる必要ありました。

クラムシェルモードでの使用において非常に不便なので、USB/Bluetooth 接続のマウスやキーボード操作でサスペンド状態から復帰できるよう設定を変更したので、その手順を共有します。

## 環境

```shell
$ hostnamectl
  Static hostname: myhost
        Icon name: computer-laptop
          Chassis: laptop 💻
Chassis Asset Tag: No Asset Tag
       Machine ID: ********************************
          Boot ID: ********************************
 Operating System: NixOS 26.05 (Yarara)
      CPE OS Name: cpe:/o:nixos:nixos:26.05
           Kernel: Linux 6.18.10
     Architecture: x86-64
  Hardware Vendor: ASUSTeK COMPUTER INC.
   Hardware Model: VivoBook 12_ASUS Laptop E203MA_E203MA
 Hardware Version: 1.0
 Firmware Version: E203MA.315
    Firmware Date: Tue 2019-06-18
     Firmware Age: 6y 9month 4d
```

## サスペンド復帰ができない原因

システムがサスペンド状態に入ると、省電力のために USB ホストコントローラもサスペンド状態に移行します。

この状態では、接続されている USBデバイス(マウスやキーボード)からの入力信号をシステムが検知できなくなるため、デバイスを操作してもサスペンド状態から復帰できないという症状が発生します。

これを解決するには、特定の USB デバイスからの入力でシステムを「起こす（Wakeup）」ための明示的な設定が必要です。

## 設定

### 1. USB 接続のデバイス情報を確認

まず `lsusb` コマンドを使用して、システムに接続されている USB バスおよびデバイスの情報を確認します。

```shell
$ lsusb
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 003: ID 13d3:5a06 IMC Networks USB2.0 VGA UVC WebCam
Bus 001 Device 004: ID 8087:0aaa Intel Corp. Bluetooth 9460/9560 Jefferson Peak (JfP)
Bus 001 Device 005: ID 056e:0157 Elecom Co., Ltd ELECOM BlueLED Mouse
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 002 Device 002: ID 04bb:01a0 I-O Data Device, Inc. I-O DATA SSPJ-UTC
```

今回の対象デバイスは以下の2つです。

- USB接続有線マウス `Bus 001 Device 005 Elecom Co., Ltd ELECOM BlueLED Mouse`
- Bluetooth アダプタ `Bus 001 Device 004: ID 8087:0aaa Intel Corp. Bluetooth 9460/9560 Jefferson Peak (JfP)`

USB接続有線マウスの詳細を確認するには、 `lsusb -v -s 001:005` を実行します。

:::details lsusb -v -s 001:005
```shell
$ lsusb -v -s 001:005

Bus 001 Device 005: ID 056e:0157 Elecom Co., Ltd ELECOM BlueLED Mouse
Couldn't open device, some information will be missing
Negotiated speed: Low Speed (1Mbps)
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               1.10
  bDeviceClass            0 [unknown]
  bDeviceSubClass         0 [unknown]
  bDeviceProtocol         0
  bMaxPacketSize0         8
  idVendor           0x056e Elecom Co., Ltd
  idProduct          0x0157 ELECOM BlueLED Mouse
  bcdDevice            1.00
  iManufacturer           0
  iProduct                2 ELECOM BlueLED Mouse
  iSerial                 0
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength       0x0022
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0
    bmAttributes         0xa0
      (Bus Powered)
      Remote Wakeup
    MaxPower              100mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           1
      bInterfaceClass         3 Human Interface Device
      bInterfaceSubClass      1 Boot Interface Subclass
      bInterfaceProtocol      2 Mouse
      iInterface              0
        HID Device Descriptor:
          bLength                 9
          bDescriptorType        33
          bcdHID               1.10
          bCountryCode            0 Not supported
          bNumDescriptors         1
          bDescriptorType        34 (null)
          wDescriptorLength      52
          Report Descriptors:
            ** UNAVAILABLE **
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0004  1x 4 bytes
        bInterval              10
```
:::



Bluetooth デバイスの情報は、`bluetoothctl` コマンドで詳細が確認できます。
接続されているキーボードは、 `WakeAllowed: yes` でウェイクアップが許可されています。

:::details bluetoothctl
```shell
$ bluetoothctl devices
Device CB:17:70:C1:XX:XX HHKB-Hybrid_4
$ bluetoothctl info CB:17:70:C1:XX:XX
Device CB:17:70:C1:80:41 (random)
        Name: HHKB-Hybrid_4
        Alias: HHKB-Hybrid_4
        Appearance: 0x03c1 (961)
        Icon: input-keyboard
        Paired: yes
        Bonded: yes
        Trusted: no
        Blocked: no
        Connected: yes
        WakeAllowed: yes
        LegacyPairing: no
        CablePairing: no
        UUID: Generic Access Profile    (00001800-0000-1000-8000-00805f9b34fb)
        UUID: Generic Attribute Profile (00001801-0000-1000-8000-00805f9b34fb)
        UUID: Device Information        (0000180a-0000-1000-8000-00805f9b34fb)
        UUID: Battery Service           (0000180f-0000-1000-8000-00805f9b34fb)
        UUID: Human Interface Device    (00001812-0000-1000-8000-00805f9b34fb)
        Modalias: usb:v04FEp0022d0001
        Battery Percentage: 0x64 (100)
```
:::

### 2. カーネルパラメータの追加

USB オートサスペンドを無効化し、サスペンド中のデバイスからウェイクアップ信号を受け取れるようにします。
`hardware-configuration.nix` に、以下の `boot.kernelParams = ... ` を追加します。

```nix:hardware-configuration.nix
{ config, lib, pkgs, modulesPath, ... }:

{
  ...

  # Disable USB auto-suspend to ensure wakeup signals are sent from USB devices
  boot.kernelParams = lib.mkAfter [ "usbcore.autosuspend=-1" ];

  ...
}
```

設定を追記後、 `sudo nixos-rebuild switch` を実行してシステム設定を反映し、ノートPCを再起動してください。

### 3. 設定反映後の確認

ノートPCを再起動後、設定が正しく反映されているか確認します。

- `autosuspend` が `-1` (無効) になっているか
- 各 USB デバイスの `wakeup` が `enabled` になっているか

```shell
### オートサスペンド設定の確認
$ cat /sys/module/usbcore/parameters/autosuspend
-1

# 各 USB デバイスの wakeup 許可状態を確認
$ grep . /sys/bus/usb/devices/*/power/wakeup
/sys/bus/usb/devices/1-2/power/wakeup:enabled
/sys/bus/usb/devices/1-9/power/wakeup:enabled
/sys/bus/usb/devices/usb1/power/wakeup:enabled
/sys/bus/usb/devices/usb2/power/wakeup:enabled
```

これにより、USB デバイスの操作による復帰が可能になります。
**しかし、この設定だけでは、意図しない外部イベントでもサスペンドが解除されてしまいます。**

### 4. ウェイクアップ対象の限定 (udev ルールの追加)

私の環境では、
USB 外付 SSD `Bus 002 Device 002: ID 04bb:01a0 I-O Data Device, Inc. I-O DATA SSPJ-UTC` を接続しています。
ここに、 NixOS システムをインストールしています。

そのため、USB ホストコントローラによるデバイスへのポーリングに対し、SSD コントローラが応答し、ウェイクアップイベントが誘発され、サスペンドが解除し、外部モニターが点灯するという副作用が発生しました。

そこで、ウェイクアップの要因を、HIDデバイス(マウス・キーボード)と Bluetooth 機器 のみに限定します。

USBのデバイスクラス分類を利用します。
- `bDeviceClass == 00` : HID デバイス
- `bDeviceClass == e0` : Bluetooth コントローラ

NixOS `hardware-configuration.nix` の `services.udev.extraRules` に以下のルールを追加します。

```nix:hardware-configuration.nix
{ config, lib, pkgs, modulesPath, ... }:

{

  ...

  services.udev.extraRules = lib.mkAfter ''
    # USB HID devices (mouse, keyboard, etc.): bDeviceClass==00: device class is defined at the interface level
    ACTION=="add", SUBSYSTEM=="usb", ATTR{bDeviceClass}=="00", ATTR{power/wakeup}="enabled"

    # Bluetooth / Wireless Controller: bDeviceClass==e0
    ACTION=="add", SUBSYSTEM=="usb", ATTR{bDeviceClass}=="e0", ATTR{power/wakeup}="enabled"
  '';
}
```

`sudo nixos-rebuild switch` でシステム設定反映し、ノートPCを再起動。

以上で、意図しないディスクアクセスなどによる復帰を防ぎつつ、USB/Bluetooth 機器の操作のみでスマートにサスペンドから復帰できるようになります。

### 特定のデバイスのみを許可する（より詳細な設定）
特定のデバイス（例：特定のメーカーのマウスのみ）に限定してウェイクアップを許可したい場合は、idVendor や idProduct を指定することで、よりセキュアで厳密な制御が可能です。

先ほどの lsusb で確認した Elecom 製マウス（ID 056e:0157）のみを対象とする場合は、以下のように記述します。

```nix
    SUBSYSTEM=="usb", ATTR{idVendor}=="056e", ATTR{idProduct}=="0157", ATTR{power/wakeup}="enabled"
```

## 注意 外部モニターの省電力設定

OS 側の設定が正しく完了していても、使用している外部モニター自体の省電力設定（ディープスリープモードなど）が有効になっていると、サスペンド復帰時に画面が正常に点灯しないことがあります。
USB/Bluetooth 機器からの操作で本体のサスペンドが解除されても映像信号が復帰しない場合は、モニター側のメニュー設定から電力管理設定を確認し、ディープスリープをオフにしてみてください。

## まとめ

NixOS (Linux) で使用していると、特有の不便さに直面することが多々あります。
今回のサスペンド復帰の問題も、長らく放置していた課題の一つでした。

解決に至る過程で、Linux の電源管理や udev ルールの仕組みなど学ぶことが多く、非常に有意義なカスタマイズとなりました。

同様の不便さを感じている方にとって、本記事の内容が参考になれば幸いです。
