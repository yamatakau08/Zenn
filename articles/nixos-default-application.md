---
title: "NixOS switch実行後、既定のアプリケーションのウェブの設定が変わってしまう対処法"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "linux", "nixos" ]
published: false
---

## はじめに
NixOS の環境で、URLリンクをマウスでクリックした場合 `Chrome` ブラウザが開くように Budgie コントロールセンターで、既定のアプリケーションのウェブに `Chrome` に設定したのに、
NixOS の設定変更反映、再起動後、URLをクリックすると、毎回別途インストールしていた `Obsidian` が起動する症状に遭遇してしまいました。
煩わしかったので、対処方法をまとめておきます。

## 対処方法調査
[How to set the default browser in NixOS?](https://unix.stackexchange.com/a/752514) を参考にしました。
この解説では、`qutebrowser` での例です。
```
home-manager.users.<YOUR_USER_NAME>.xdg.mimeApps = {
  enable = true;

  defaultApplications = {
    "text/html" = "org.qutebrowser.qutebrowser.desktop";
    "x-scheme-handler/http" = "org.qutebrowser.qutebrowser.desktop";
    "x-scheme-handler/https" = "org.qutebrowser.qutebrowser.desktop";
    "x-scheme-handler/about" = "org.qutebrowser.qutebrowser.desktop";
    "x-scheme-handler/unknown" = "org.qutebrowser.qutebrowser.desktop";
  };
};
```

この設定と同様の設定するにあたり、

`xdg.mimeApps.defaultApplications` の設定に関して、[xdg.mimeApps.defaultApplications](https://nix-community.github.io/home-manager/options.xhtml#opt-xdg.mimeApps.defaultApplications) に記述説明を参照します。

```
{
  "mimetype1" = [ "default1.desktop" "default.desktop" ];
}
```

`mimetype1` は、参考にしたページ中のをそのまま用いました。

```
"text/html"
"x-scheme-handler/http"
"x-scheme-handler/https"
"x-scheme-handler/about"
"x-scheme-handler/unknown"
```

`default1.desktop` は、既定のアプリケーション、
`default.desktop`  は、`default1.desktop` がインストールされていない場合に用いられるアプリケーション
を指定するとの説明です。

自分の環境でのブラウザ `Chrome` に設定したいのですが、`default1.desktop` に、どういった文字列を設定すれば良いのか分かりませんでしたが、
既定のアプリケーションは、 `~/.config/mimeapps.list` で管理されているのを知りました。
既定のアプリケーションを `Chrome` に設定した状態で確認し、`chrome` の場合は、`chromium-browser.desktop` を設定すれば良いことが分かりました。

```
yama@tnt ~/.config> cat ~/.config/mimeapps.list
[Default Applications]
x-scheme-handler/http=obsidian.desktop
x-scheme-handler/https=obsidian.desktop
x-scheme-handler/chrome=firefox.desktop
text/html=obsidian.desktop
application/x-extension-htm=firefox.desktop
application/x-extension-html=firefox.desktop
application/x-extension-shtml=firefox.desktop
application/xhtml+xml=chromium-browser.desktop
application/x-extension-xhtml=firefox.desktop
application/x-extension-xht=firefox.desktop
x-scheme-handler/about=obsidian.desktop
x-scheme-handler/unknown=obsidian.desktop

[Added Associations]
x-scheme-handler/http=firefox.desktop;chromium-browser.desktop;
x-scheme-handler/https=firefox.desktop;chromium-browser.desktop;
x-scheme-handler/chrome=firefox.desktop;
text/html=firefox.desktop;chromium-browser.desktop;
application/x-extension-htm=firefox.desktop;
application/x-extension-html=firefox.desktop;
application/x-extension-shtml=firefox.desktop;
application/xhtml+xml=firefox.desktop;
application/x-extension-xhtml=firefox.desktop;
application/x-extension-xht=firefox.desktop;
```

`default.desktop` には、NixOS をインストールした際に、インストールされるブラウザ `firefox.desktop` を設定します。

# NixOS での既定のアプリケーションの設定

ここから、NixOS での既定のアプリケーションの設定になります。

既定のアプリケーションは、個人設定の範疇にあたるので、Home Manager で管理するのが良いです。

`xdg.mimeApps.defaultApplications` は、[xdg-mime-apps.nix](https://github.com/nix-community/home-manager/blob/master/modules/misc/xdg-mime-apps.nix) で、Home Manager で定義されています。
設定に用いるNixのモジュールのファイル名を、定義ファイル名に倣い `xdg-mime-apps.nix` とし、下記のように記述しました。

```
yama@tnt ~> cat ~/.config/nix/home-manager/xdg-mime-apps.nix
{ pkgs, ... }:

{
  xdg.mimeApps = {
    enable = true;

    defaultApplications = {
      "x-scheme-handler/http"    = ["chromium-browser.desktop" "firefox.desktop"];
      "x-scheme-handler/https"   = ["chromium-browser.desktop" "firefox.desktop"];
      "text/html"                = ["chromium-browser.desktop" "firefox.desktop"];
      "x-scheme-handler/about"   = ["chromium-browser.desktop" "firefox.desktop"];
      "x-scheme-handler/unknown" = ["chromium-browser.desktop" "firefox.desktop"];
    };
  };
}
```

このモジュールを `home.nix` などの `imports` リストに追加します。

```
# home.nix

imports = [
  # ...
  xdg-mime-apps.nix
  # ...
];
```

システムへ反映（switch）を実行する際、すでに ~/.config/mimeapps.list が存在していると、
Home Manager のファイル生成に失敗しエラーとなるので、あらかじめファイルをリネームしておきます。

```
yama@tnt ~/.config> mv mimeapps.list mimeapps.list.org
```

利用環境に合わせて以下のコマンドで、システムに設定を反映します。

- Home Manager単独の場合 `home-manager switch`
- Home Manager を NixOS モジュールとして利用している場合 `sudo nixos-rebuild switch`

```
yama@tnt ~/.c/n/nixos (main) [4]> sudo nixos-rebuild switch --flake ~/.config/nix/nixos#tnt
[sudo] yama のパスワード:
warning: Git tree '/home/yama/.config/nix' is dirty
building the system configuration...
warning: Git tree '/home/yama/.config/nix' is dirty
activating the configuration...
setting up /etc...
reloading user units for yama...
restarting sysinit-reactivation.target
the following new units were started: NetworkManager-dispatcher.service
Done. The new configuration is /nix/store/0z0hwjign6irmnmnivwhznrv1jlqhpii-nixos-system-tnt-26.05.20251228.c0b0e0f
```

ビルドが成功すると、`~/.config/mimeapps.list` が `/nix/store/` 配下に配置され、設定した内容で更新されます。

```shell
yama@tnt ~> ls -l ~/.config/mimeapps.list
lrwxrwxrwx 1 yama users 84  1月 18 21:56 /home/yama/.config/mimeapps.list -> /nix/store/2lizwdpsxb8504ra146m8frb1vv523g6-home-manager-files/.config/mimeapps.list

yama@tnt ~/.config> cat mimeapps.list
[Added Associations]

[Default Applications]
text/html=chromium-browser.desktop;firefox.desktop
x-scheme-handler/about=chromium-browser.desktop;firefox.desktop
x-scheme-handler/http=chromium-browser.desktop;firefox.desktop
x-scheme-handler/https=chromium-browser.desktop;firefox.desktop
x-scheme-handler/unknown=chromium-browser.desktop;firefox.desktop

[Removed Associations]
yama@tnt ~>
```
これで、`home-manager switch` や `sudo nixos-rebuild switch` を実行しても、既定のアプリケーション設定が勝手に元に戻ってしまうことはなくなります。

## まとめ
Nix のシステム反映によるものなのか、Nix 管理外にしていたファイル `~/.config/mimeapps.list` が、書き換えられる原因までの究明には至りませんでしたが、
Nix の管理下にした事で、この煩わしい症状に悩まされることから解放されました。

