---
title: "NixOS Linux で ノートPC の蓋を閉じた際に suspend に入る設定を調べてみた!"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "NixOS", "Linux", "logind" ]
published: true
---

## はじめに
ノートPCに NixOS をインストールしました。
外部モニターを接続し、ノートPCをクラムシェルモード（ドック状態）で使用しています。
特に設定していないのに、ノートPCの蓋を閉じてもサスペンドにならず、そのままクラムシェルモードで利用できています。
その設定がどこで行われているのか疑問に感じたため調べてみました。
本記事では、その調査結果についてまとめています。

## 環境

```shell
$ hostnamectl
  Static hostname: <my-hostname>
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

## ノートPCの蓋を閉じた際のサスペンド動作へ入る仕組み
1. ノートPCの蓋を閉じると、`Lid Closed` という ACPI （Advanced Configuration and Power Interface） イベントが発生します。
2. カーネルは ACPI イベントを受け取り、システム全体のログインや電源管理を司るサービス `systemd-logind` (以下、`logind`)に通知します。
3. 通知を受けた `logind` は、設定ファイル内の `HandleLidSwitch` という項目の値を確認し、サスペンド（スリープ）や画面ロックなどの動作を実行します。

## logind の設定箇所の確認

ということで、`logind` の `HandleLidSwitch` の設定箇所を調べていきます。

### NixOS の設定 configuration.nix の確認

`nix repl` で `lidSwitch` オプションの設定を確認しましたが、オプション名が変更されており、値を参照できませんでした。

```shell
$ nix repl
Nix 2.31.3
Type :? for help.
nix-repl> :lf .
warning: Git tree '/home/my/.config/nix' is dirty
Added 12 variables.
_type, dirtyRev, dirtyShortRev, inputs, lastModified, lastModifiedDate, narHash, nixosConfigurations, outPath, outputs, sourceInfo, submodules

nix-repl> nixosConfigurations.my-hostname.options.services.logind.lidSwitch
{
  __toString = «lambda __toString @ /nix/store/4ggd0kb8as38xa0kr730qpnsa89df0x7-source/lib/modules.nix:1168:20»;
  _type = "option";
  apply = «lambda apply @ /nix/store/4ggd0kb8as38xa0kr730qpnsa89df0x7-source/lib/modules.nix:1977:19»;
  declarationPositions = [ ... ];
  declarations = [ ... ];
  definitions = [ ... ];
  definitionsWithLocations = [ ... ];
  description = "Alias of {option}`services.logind.settings.Login.HandleLidSwitch`.";
  files = [ ... ];
  highestPrio = 9999;
  isDefined = false;
  loc = [ ... ];
  options = [ ... ];
  type = { ... };
trace: Obsolete option `services.logind.lidSwitch' is used. It was renamed to `services.logind.settings.Login.HandleLidSwitch'.
«error: evaluation aborted with the following error message: 'Renaming error: option `services.logind.settings.Login.HandleLidSwitch' does not exist.'»;
  valueMeta = { ... };
  visible = false;
}
```

### logind 設定ファイルの確認

次に、`logind` が実際に読み込んでいる設定ファイルを確認をします。

`systemd-analyze cat-config` を使用すると、`/etc/systemd/logind.conf` が読み込まれていることがわかります。

```shell
$ systemd-analyze cat-config systemd/logind.conf
# /etc/systemd/logind.conf -> /nix/store/g5ymfyip8ma7lgxlxxafdvxsr6x9cy6z-etc-systemd-logind.conf
[Login]
KillUserProcesses=false
```

`HandleLidSwitch` に関する設定はありませんでした。

### logind 実行状態での確認

現在の動作状態を調べるため、`busctl` コマンドを使用して D-Bus (Desktop Bus) 経由で `logind` のプロパティを確認します。


```shell
$ busctl list
NAME                                 PID PROCESS         USER             CONNECTION    UNIT                      SESSION DESCRIPTION
...
:1.2                                 825 systemd-logind  root             :1.2          systemd-logind.service    -       -
...
org.freedesktop.login1               825 systemd-logind  root             :1.2          systemd-logind.service    -       -
...
```

次に、`busctl introspect` コマンドを用いて、オブジェクトの詳細を取得します。

```shell
$ busctl --help
busctl [OPTIONS...] COMMAND ...

Introspect the D-Bus IPC bus.

Commands:
  ...
  introspect SERVICE OBJECT [INTERFACE]
  ...
```

- `SERVICE` に `org.freedesktop.login1`  (NAME列 `:1.2` などの数字だけではなく、サービス名を指定)
- `OBJECT`  に `/org/freedesktop/login1` (`login1` が公開するルートオブジェクトのパス)

を指定すると、そのオブジェクトが持つ「メソッド」「シグナル」「プロパティ」がすべて表示されます。


```shell
$ busctl introspect org.freedesktop.login1 /org/freedesktop/login1
NAME                                TYPE      SIGNATURE           RESULT/VALUE                             FLAGS
 ...
.HandleLidSwitch                    property  s                   "suspend"                                const
.HandleLidSwitchDocked              property  s                   "ignore"                                 const
.HandleLidSwitchExternalPower       property  s                   ""                                       const
 ...
```

ようやく `logind` の `HandleLidSwitch` に関する現在の設定値が確認できました!

LidSwitch のイベントに対するデフォルトの動作が、archlinux.jp ドキュメント [ACPI イベント](https://wiki.archlinux.jp/index.php/%E9%9B%BB%E6%BA%90%E7%AE%A1%E7%90%86#ACPI_%E3%82%A4%E3%83%99%E3%83%B3%E3%83%88) にまとめられていました。

なお、プロパティの FLAGS にある `const` は、その値がサービス起動時に決定され、動作中には変更されない定数です。

## 蓋の開閉状態のリアルタイム確認
以下のコマンドを実行すると、ノートPCの蓋の開閉に応じた状態変化をリアルタイムで確認できます。
```shell
$ watch -n 1 "busctl get-property org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager LidClosed"
```

## まとめ

以上の調査から、NixOS 固有の設定ではなく `logind` のデフォルト値によってこの挙動が制御されているとわかりました。

ノートPCの蓋を閉じた際にサスペンドする設定がどこで定義されているのか、そして現在の状態を D-Bus 経由で確認する方法を整理できました。
デフォルトで「外部モニター接続時（Docked）はサスペンドを無視する」という設定になっているおかげで、特別な設定なしにクラムシェルモードが利用できていた、という仕組みが明確になりました。