---
title: "niri 環境で Emacs のウィンドウの高さを画面の高さに自動で合わせる設定"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "Emacs", "wayland", "niri" ]
published: false
---

## はじめに
Wayland niri 環境で Emacs を起動すると、ウィンドウの高さが他のアプリと一致しない問題がありました。

これまで、画面の高さを合せるため、niri のショートカット `Super + F` でウィンドウを最大化した後、
マウスドラッグで横幅を調整するという煩雑な操作を行っていました。

この手間を解消するための対処方法を共有しておきます。

## 環境

```console
> niri --version
niri 25.11 (Nixpkgs)
```

```console
(emacs-build-description)

In GNU Emacs 30.2 (build 1, x86_64-pc-linux-gnu, GTK+ Version 3.24.51,
cairo version 1.18.4)
System Description: NixOS 26.05 (Yarara)

Configured using:
 'configure
 --prefix=/nix/store/1vmdsfvnhi5zgsjqn2vngqqkb9zhs8a4-emacs-pgtk-30.2
 --disable-build-details --with-modules --with-pgtk
 --disable-gc-mark-trace --with-compress-install
 --with-toolkit-scroll-bars --with-native-compilation
 --without-imagemagick --with-mailutils --without-small-ja-dic
 --with-tree-sitter --without-xinput2 --without-xwidgets --with-dbus
 --with-selinux'
```

## 対処方法

niri が ウィンドウマネージャーとして Emacs のウィンドサイズ(高さ)を適切に制御できるようにするため、Emacs でのフレームの高さを指定する `height` 変数を設定しないようにします。
ただし、この変更のみではモードライン以下の領域が画面下部からはみ出してしまう症状が発生します。

GitHub の https://github.com/YaLTeR/niri/issues/2632#issuecomment-3586826278 を参照したところ
以下の設定
`(setopt frame-inhibit-implied-resize t)`
を追加することで、モードラインやミニバッファも正しく表示されるようになります。

## まとめ
niri を利用するまで、Emacs 側でフレームの位置やサイズを設定していましたが、niri 環境下ではウィンドウマネージャー側がサイズを制御します。
そのため、Emacs 側ではサイズに関する直接的な設定を行わないのが最適なようです。
