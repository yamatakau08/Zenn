---
title: "niri環境でxremapの自動起動でapplication判別が効かない問題の対処"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "linux", "nixos", "niri", "xremap" ]
published: true
---

## はじめに
NixOS にも興味が湧き、手元にあったノートPC に NixOS をインストールしてみました。

エディタは `Emacs` を常用しているため、

- 半角/全角 → `Esc`
- Caps lock → `Ctrl`

他のアプリケーションでも `Emacs` ライクなキーバインドを使用したいので、
アプリケーションごとに柔軟な設定が可能な [xremap](https://github.com/xremap/xremap) を導入しました。

キーリマップは、作業効率に直結する極めて重要な要素です。
自分の理想どおりに設定が決まると、作業がより一層捗りますよね。

## 背景
NixOS インストール当初は、デスクトップ環境として `Budgie (X11)` を使用していました。
各種アプリケーションの導入に加え、苦労の末、`xremap` の設定やそのサービスの自動起動設定も完了。
アプリケーション判別機能も無事に動作し、理想の環境へと近づきつつありました。

そんな折、スクロール型タイルコンポジター [niri](https://github.com/YaLTeR/niri?tab=readme-ov-file#) の操作体験が素晴らしいという情報を目にし、
`Budgie` から `niri` へ環境を移行することにしました。

`niri` でも引き続き `xremap` を使用するため `Budgie` での設定を流用すれば問題ないと考えていました。
しかし、いざ設定してみると **「サービスの自動起動はできるものの、アプリケーション判別が機能しない」** という問題に直面しました。

Web検索やLLMを活用しても解決に至らず、非常に苦労しましたが、
試行錯誤の結果、簡潔で確実と思われる対処法を見つけることができました。
暫定的な対応かもしれませんが、同様の悩みを持つ方の参考になればと思い、その方法を共有します。

## xremap とは

[xremap](https://github.com/xremap/xremap) は、Rust製の強力なキーリマップツールです。

最大の魅力は [application判別機能](https://github.com/xremap/xremap?tab=readme-ov-file#application) です。
これにより、アプリケーションごとに異なるキーマップを割り当てることが可能になります。

導入時の注意点
[Installation](https://github.com/xremap/xremap?tab=readme-ov-file#installation) の項にある通り、
使用するデスクトップ環境（DE）やウィンドウマネージャー（WM）によってインストールすべきバイナリが異なる点に注意してください。
今回は `niri` 環境で使用するため、`niri` 版をインストールする必要があります。

設定方法
具体的なキーリマップの設定については、公式の [Configuration](https://github.com/xremap/xremap?tab=readme-ov-file#configuration) を参照してください。
また、リポジトリ内の [example](https://github.com/xremap/xremap/tree/master/example) に、
`Emacs` キーバインドの設定例も用意されており、導入の参考になります。

## 環境

NixOS 26.05 (Yarara)

```
yama@tnt ~> /nix/store/xnfcaqkr7jg4q98zbsyv6r2igp3vba4x-xremap-0.14.8/bin/xremap --version
xremap 0.14.8
yama@tnt ~> niri --version
niri 25.11 (Nixpkgs)
```

## xremap サービス起動設定
今回、[xremap-nix-flake](https://github.com/xremap/nix-flake) を利用し、`niri` 版のバイナリをインストールしました。
あわせて、 `xremap` を `systemd` 経由で実行するための定義モジュールを作成しています。

`systemd` サービスには、
  - システム(System)
  - ユーザー(User)

の2種類のモードがあります。

[xremap nix flake のテスト結果](https://github.com/xremap/nix-flake?tab=readme-ov-file#what-this-is)に基づき、
ユーザーモードで起動する設定を用いました。

下記が、`flake.nix` と `xremap-niri.nix` のモジュールになります。

```shell
yama@tnt ~> cat ~/.config/nix/nixos/flake.nix
{
  description = "NixOS configuration flake";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";

    ...

    # xremap flake
    xremap-flake.url = "github:xremap/nix-flake";
  };

  outputs = inputs@{ self, nixpkgs, ... }:
    let
      ...

      mkNixosSystem = hostname: host@{ username, platform, ... }:
        nixpkgs.lib.nixosSystem {
          system = platform;
          specialArgs = {
            inherit inputs username hostname;
          };
          modules = [
	    ...
	    ./xremap-niri.nix
	    ...
          ];
        };
    in
    {
      nixosConfigurations = builtins.mapAttrs mkNixosSystem hosts;
    };
}

yama@tnt ~> cat ~/.config/nix/nixos/xremap-niri.nix
{ pkgs, inputs, username, ... }:

{
  imports = [
    inputs.xremap-flake.nixosModules.default # xremap flake の nixOSModules を利用
  ];

  services.xremap = {
    enable = true; # サービス起動
    withNiri = true; # xremap niri 版
    userName = username;
    serviceMode = "user";

    yamlConfig = builtins.readFile ./xremap-config.yml; # 本ファイルから xremap config.yml のパス指定
  };
}
```

`xremap-niri.nix` の定義モジュールファイルの作成ができたら、
`sudo nixos-rebuild switch --flake <ディレクトリのパス>#<ホスト名>` を実行して、
システムに反映させます。

実際出力された `xremap` の `systemd` ユーザーサービス定義ファイルは、以下の通りになります。

```
yama@tnt ~> cat /etc/systemd/user/xremap.service
[Unit]
Description=xremap user service

[Service]
Environment="LOCALE_ARCHIVE=/nix/store/flgky76263ffcjajps2awh8bx5q0f9pn-glibc-locales-2.40-66/lib/locale/locale-archive"
Environment="PATH=/nix/store/xnfcaqkr7jg4q98zbsyv6r2igp3vba4x-xremap-0.14.8/bin:/nix/store/imad8dvhp77h0pjbckp6wvmnyhp8dpgg-coreutils-9.8/bin:/nix/store/av4xw9f56xlx5pgv862wabfif6m1yc0a-findutils-4.10.0/bin:/nix/store/x3zjxxz8m4ki88axp0gn8q8m6bldybba-gnugrep-3.12/bin:/nix/store/drc7kang929jaza6cy9zdx10s4gw1z5p-gnused-4.9/bin:/nix/store/zf8qy81dsw1vqwgh9p9n2h40s1k0g2l1-systemd-258.2/bin:/nix/store/xnfcaqkr7jg4q98zbsyv6r2igp3vba4x-xremap-0.14.8/sbin:/nix/store/imad8dvhp77h0pjbckp6wvmnyhp8dpgg-coreutils-9.8/sbin:/nix/store/av4xw9f56xlx5pgv862wabfif6m1yc0a-findutils-4.10.0/sbin:/nix/store/x3zjxxz8m4ki88axp0gn8q8m6bldybba-gnugrep-3.12/sbin:/nix/store/drc7kang929jaza6cy9zdx10s4gw1z5p-gnused-4.9/sbin:/nix/store/zf8qy81dsw1vqwgh9p9n2h40s1k0g2l1-systemd-258.2/sbin"
Environment="TZDIR=/nix/store/xaa75rd44q62nc9mrbvym9d1m6gy0fj8-tzdata-2025b/share/zoneinfo"
ExecStart=/nix/store/xnfcaqkr7jg4q98zbsyv6r2igp3vba4x-xremap-0.14.8/bin/xremap /nix/store/8vcbi0ix1djavcp9mchdkb8dqcfc48pw-xremap-config.yml
KeyringMode=private
LockPersonality=true
ProtectSystem=true
RestrictAddressFamilies=AF_UNIX
RestrictRealtime=true
SystemCallArchitectures=native
SystemCallFilter=~@clock
SystemCallFilter=~@debug
SystemCallFilter=~@module
SystemCallFilter=~@reboot
SystemCallFilter=~@swap
SystemCallFilter=~@cpu-emulation
SystemCallFilter=~@obsolete
UMask=077

[Install]
WantedBy=graphical-session.target
```

## 発生した問題

NixOS を再起動し、動作確認すると、
ログイン後に `xremap` のキーリマップ自体は機能しているものの、 **「application 判別が動作していない」** ことが判明しました。

`niri` のセッションに入った直後、ターミナルから `xremap` のサービス状態を確認してみると、以下のようなログが出力されていました。

```shell
Welcome to fish, the friendly interactive shell
Type help for instructions on how to use fish
yama@tnt ~> systemctl --user status xremap
● xremap.service - xremap user service
     Loaded: loaded (/etc/systemd/user/xremap.service; enabled; preset: ignored)
     Active: active (running) since Sat 2026-01-10 18:38:13 JST; 43s ago
 Invocation: 68810156fa254059a4d1b5c29d40fbb4
   Main PID: 1368 (xremap)
      Tasks: 1 (limit: 4414)
     Memory: 3.6M (peak: 3.9M)
        CPU: 57ms
     CGroup: /user.slice/user-1000.slice/user@1000.service/app.slice/xremap.service
             └─1368 /nix/store/xnfcaqkr7jg4q98zbsyv6r2igp3vba4x-xremap-0.14.8/bin/xremap /nix/store/8vcbi0ix1djavcp9mchdkb8dqcfc48pw-xremap-config.yml

 1月 10 18:38:13 tnt xremap[1368]: /dev/input/event6 : Power Button
 1月 10 18:38:13 tnt xremap[1368]: /dev/input/event7 : ELAN1201:00 04F3:3054 Mouse
 1月 10 18:38:13 tnt xremap[1368]: /dev/input/event8 : ELAN1201:00 04F3:3054 Touchpad
 1月 10 18:38:13 tnt xremap[1368]: /dev/input/event9 : Asus WMI hotkeys
 1月 10 18:38:13 tnt xremap[1368]: ------------------------------------------------------------------------------
 1月 10 18:38:13 tnt xremap[1368]: Selected keyboards automatically since --device options weren't specified:
 1月 10 18:38:13 tnt xremap[1368]: ------------------------------------------------------------------------------
 1月 10 18:38:13 tnt xremap[1368]: /dev/input/event0 : AT Translated Set 2 keyboard
 1月 10 18:38:13 tnt xremap[1368]: ------------------------------------------------------------------------------
 1月 10 18:38:51 tnt xremap[1368]: application-client: Niri (supported: false)
```

ログの最後に `application-client: Niri (supported: false)` と表示されいます。

この状態では、アプリケーション毎の個別キーマップが適用されません。

このエラー表示は、
[current_application](https://github.com/xremap/xremap/blob/bb8d3460d1578548978a7d3ae39ea462b04e5b9e/src/client/mod.rs#L54), [current_window](https://github.com/xremap/xremap/blob/bb8d3460d1578548978a7d3ae39ea462b04e5b9e/src/client/mod.rs#L41) から呼び出されている
[check_supported](https://github.com/xremap/xremap/blob/bb8d3460d1578548978a7d3ae39ea462b04e5b9e/src/client/mod.rs#L36) 関数中で出力しています。

current application や current window の情報取得に失敗しているようです。

## 手動での再起動による回復

上記の状態から `xremap` ユーザーサービスを **手動で再起動** してみます。

`systemctl --user restart xremap`

`application-client: Niri (supported: true)`
となり、 **application判別が正常動作する** 事が確認できました。

```shell
yama@tnt ~> systemctl --user restart xremap
yama@tnt ~> systemctl --user status xremap
 ...
 1月 10 18:39:16 tnt xremap[1887]: application-client: Niri (supported: true)
 1月 10 18:39:16 tnt xremap[1887]: application: org.wezfurlong.wezterm
```

## 対処方法

`niri` セッションにログインするたびに、`xremap` のユーザーサービスを手動で再起動するのは手間がかかるため、
自動再起動したいところです。


`niri` の設定ファイル `~/.config/niri/config.kdl` に

```kdl
spawn-at-startup "systemctl" "--user" "restart" "xremap.service"
```

を追加し、

`niri` セッション開始時の startup 処理に `xremap` のユーザーサービス再起動を組み込みます。
これで、ログイン後も **xremapのアプリケーション判別も正常に動作** するようになります。

## まとめ

この問題は、`xremap.service` の起動順序に起因している推測し、 `systemd` 側で調整を試みました。
具体的には、`niri.service` の起動後に `xremap.service` を起動してみたり、色々試しましたが、解決には至りませんでした。

`systemd-analyze plot --user > > user-service-plot.svg` で確認したところ、グラフ内には、`niri.service` の起動が、
`systemd` に把握されていないようです。

現状では `niri` のスタートアップ処理で、`xremap` のユーザーサービス再起動する方法が、
最も簡単かつ確実な対処方法と思います。

もし、より適切な対処方法があれば共有いただければ幸いです。

![](/images/systemd-analyze-user-plot.png)
*systemd-analyze plot --user > user-service-plot.svg*