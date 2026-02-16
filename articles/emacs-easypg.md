---
title: "Emacs EasyPG で暗号化ファイルを作成する"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "Emacs", "EasyPG", "GnuPG" ]
published: false
---

## はじめに
Emacs の認証情報管理パッケージである auth-source パッケージを利用するにあたり、
認証情報を安全に保存する `~/.authinfo.gpg` の作成が必要になりました。

通常、GnuPG(GPG) で暗号化されたファイルを扱うには、ターミナルから `gpg` コマンドを操作する必要があります。
しかし、Emacs 標準の EasyPG(EasyPG Assistant) ライブラリを活用すれば、
コマンドを意識することなく、Emacs 上から直接（透過的に）暗号化ファイルの作成・編集・保存が行えます。

## EasyPG とは
EasyPG は Emacs で GNU Privacy Guard を扱うためのユーザーインターフェースです。
標準で付属しており、外部ツールとして GnuPG がインストールされていれば利用できます。

本記事では、EasyPG を使った共通鍵暗号方式によるファイルの暗号・復号化の手順を説明します。

## 環境
本記事の検証環境を以下に示します。
### Emacs の環境
`*scratch*` バッファで評価した結果は以下の通りです。
```elisp
system-type
;; => gnu/linux
emacs-version
;; => "30.2"
```

### EasyPG の設定確認
EasyPG に関連する設定を `*scratch*` バッファで確認します。
#### EasyPG version
```elisp
epg-version-number
;; => "1.0.0"
```

#### EasyPG が利用する GnuPG プログラムの確認
`*scratch*` バッファで `epg-gpg-program` 変数を確認すると、`gpg2` が指定されています(環境によっては `gpg` の場合もあります)。
また `(executable-find epg-gpg-program)` を評価し、Emacs から正しく GnuPG プログラムが検出できるか確認しておきます。
```elisp
epg-gpg-program
;; => "gpg2"

(executable-find epg-gpg-program)
;; => "/etc/profiles/per-user/yama/bin/gpg2"
```

`epg-gpg-program` は、`epg-config.el` 内で定義されており、
`gpg2` が実行ファイルとして検出できれば、`gpg2` に、それ以外の場合は、`gpg` に初期設定されます。
```elsip:epg-config.el
(defcustom epg-gpg-program (if (executable-find "gpg2")
                               "gpg2"
                             "gpg")
```

#### pinentry-mode の設定
`epg-pinentry-mode` は初期設定で `nil` です。

このままだとパスフレーズ入力時に外部ダイアログプログラムが必要になる場合があるため、
Emacs 内で入力を完結させるように、`loopback` を指定します。
以下に `use-package` を用いた設定例を載せておきます。

```elisp
(use-package epg
  :init
  (setopt epg-pinentry-mode 'loopback)
  )
```

### gpg2 コマンドの確認
ターミナルからも `epg-gpg-program` のコマンド `gpg2` のパスが Emacs 側の確認結果と一致していることと、正常に動作することを確認しておきます。
```console
yama@tnt ~> which gpg2
/etc/profiles/per-user/yama/bin/gpg2

yama@tnt ~> gpg2 --version
gpg (GnuPG) 2.4.8
libgcrypt 1.11.2
Copyright (C) 2025 g10 Code GmbH
License GNU GPL-3.0-or-later <https://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Home: /home/yama/.gnupg
サポートしているアルゴリズム:
公開鍵: RSA, ELG, DSA, ECDH, ECDSA, EDDSA
暗号方式: IDEA, 3DES, CAST5, BLOWFISH, AES, AES192, AES256,
      TWOFISH, CAMELLIA128, CAMELLIA192, CAMELLIA256
ハッシュ: SHA1, RIPEMD160, SHA256, SHA384, SHA512, SHA224
圧縮: 無圧縮, ZIP, ZLIB, BZIP2
```

## 暗号化ファイルの作成
まず、Emacs でファイル名の拡張子に `.gpg` がついたファイルを新規作成します。
例として `/tmp/foobar.gpg` を作成します。

1. `C-x C-s` でファイルを保存すると、暗号化設定のための `*Keys*` バッファがポップアップします。
![](/images/EasyPGC-xC-sPrompt.png)

2. 暗号化方式の選択
共通鍵(パスフレーズ)暗号化を行うため、特定の鍵を選択せずに、
`[OK]` フィールドにカーソルを移動し Enter を押します。

3. パスフレーズの入力
ミニバッファで Passphrase 入力を促されるので、設定したいパスフレーズを入力
![](/images/EasyPGEnterPassphrase.png)

4. パスフレーズの再入力
![](/images/EasyPGConfirmPassword.png)

これで暗号化されたファイルが作成されます。

## 暗号化ファイルの確認
作成したファイルが正しく暗号化されているか確認します。

### Emacs での確認
`.gpg` ファイルを Emacs で開こうとすると、ミニバッファでパスフレーズの入力を促されます。
正しいパスフレーズを入力すると、バッファに復号化された内容が表示されます。

### ターミナルでの確認
`cat` コマンドでファイルの中身を直接見ると、暗号化されていることが分かります。
```console
yama@tnt ~> cat -v /tmp/foobar.gpg
M-^L^M^D        ^C
;"sZM-?M-_M-^\#M-sM-RL^AM-xM-@^B6`M-a^PM-/`OM-"^MM-DM-^VM-(M-;^M-PM-<0M-^V<c^ZM-;M-AM-VM-dM-qM-wM-$M-^N2M-fM-aM-^QM-$^^M-r~^[<M-$M-;M-5
M-|M-^GAM-'M-^W0%^CM-PM-<M-YM-(M-[M-*M-2M-xM-{M-kM-e`M-{M-#M-^@[M-xM-d)0+⏎
```

`gpg2` コマンドを用いてターミナル上で復号できるかも確認しておきましょう。
```console
yama@tnt ~> gpg2 --decrypt /tmp/foobar.gpg
gpg: AES256.CFB暗号化済みデータ
gpg: 1 個のパスフレーズで暗号化
This is foobar.gpg⏎
```

### ファイル権限の推奨設定
デフォルトでは group と other に読み取り権限が付与されている場合があります。
```console
yama@tnt ~> ls -l /tmp/foobar.gpg
-rw-r--r-- 1 yama users 93  2月 16 15:44 /tmp/foobar.gpg
```

認証情報などを扱う重要なファイルの場合、所有者のみがアクセスできるよう
`chmod go-r foobar.gpg`
で参照権限を絞っておくと、より安全です。

## まとめ
Emacs EasyPG を活用すれば、GnuPG のコマンド操作を意識することなく、
通常のファイル作成とほぼ同じ感覚で暗号化ファイルを扱うことができます。
`~/.authinfo.gpg` のような認証情報をはじめ、プライバシーに関わる機密ファイルを作成・管理する際に非常に便利です。

