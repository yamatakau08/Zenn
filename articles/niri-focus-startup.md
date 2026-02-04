---
title: "niri 起動時に外部モニターに自動でフォーカスを当てる設定"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "linux", "niri" ]
published: false
---

## はじめに
Laptop に、A scrollable-tiling Wayland compositor [niri](https://github.com/YaLTeR/niri?tab=readme-ov-file) をインストールして約一ヶ月が経ちました。

Laptop の画面 11.6 inch は小さいため、自宅では 39.7 inch の外部モニターをメインに使用し、
全てのウィンドウを外部モニター側で開くようにしています。

そこで困っていたのは、
**niri 起動直後に必ず LapTop の画面がフォーカスが当たってしまう** ことです。
毎回、Super + Shift + ← あるいは → で外部モニターにフォーカスを移動させたり、
マウスを外部モニター側に動かしたりする操作を負担に感じていました。

長らく放置していましが、重い腰を上げて対処方法を探してみたところ、ありました!

## 対処方法

`~/.config/niri/config.kdl` ファイルに、
下記のように、外部モニター (e.g. "HDMI-A-1") 用の output セクションを追加し
[focus-at-startup](https://yalter.github.io/niri/Configuration%3A-Outputs.html#focus-at-startup) を設定する事で対処できます。

```
# ~/.config/niri/config.kdl
// Focus HDMI-A-1 by default.
output "HDMI-A-1" {
    focus-at-startup
}
```

`output` の後ろ指定する文字列は、`connector name` を指定します。

`connector name` は、`niri msg outputs` コマンドを実行し、
出力結果の `Output` 行の右端にある `()` 内の文字列で確認できます。

ちなみに、`eDP-1` は laptop の内蔵モニターになります。

```shell
~> niri msg outputs
Output "Chimei Innolux Corporation 0x1130 Unknown" (eDP-1)
  Current mode: 1366x768 @ 60.012 Hz (preferred)
  Variable refresh rate: not supported
  Physical size: 260x140 mm
  Logical position: 0, 0
  Logical size: 1366x768
  Scale: 1
  Transform: normal
  Available modes:
    1366x768@60.012 (current, preferred)

Output "LG Electronics LG ULTRAFINE 507NTUWNK502" (HDMI-A-1)
  Current mode: 3840x2160 @ 30.000 Hz
  Variable refresh rate: not supported
  Physical size: 930x390 mm
  Logical position: 1366, 0
  Logical size: 3840x2160
  Scale: 1
  Transform: normal
  Available modes:
    3840x2160@30.000 (current)
    ...
    640x480@59.940
```

## 最後に
niri 起動のたびに行なっていた、「フォーカスを外部モニターに移動する操作」が省け、
作業環境がより快適になりました。
他にも [niri configuration](https://yalter.github.io/niri/Configuration%3A-Layout.html) には便利そうな設定が多そうなので、
さらに使いやすくカスタマイズしていこうと思います。

