---
title: "GnuPG ターミナルで完結するファイル暗号化・復号化手順"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "Linux", "Mac", "GnuPG" ]
published: true
---

## はじめに
Emacs の auth-source パッケージで、認証情報を管理する `~/.auth-info.gpg` ファイルを利用する必要が出てきました。
これに伴い、ファイルを安全に扱うための暗号化が必要になったので、Emacs 連携の前に GnuPG (GNU Privacy Guard) 単独でファイルの暗号化、復号化を確認してみることにしました。

本記事は、ファイルの暗号化と復号化の手順、およびその際につまずいた点をまとめたものになります。

## 環境
環境は Linux (NixOS) です。
```shellscript
yama@tnt ~> uname -a
Linux tnt 6.12.68 #1-NixOS SMP PREEMPT_DYNAMIC Fri Jan 30 09:28:49 UTC 2026 x86_64 GNU/Linux
```

## GnuPG (GNU Privacy Guard) のインストール
[GnuPG](https://www.gnupg.org/) あるいはお使いのパッケージマネージャーでインストールしてください。
多くの環境では Version 2 系が標準となっています。
`gpg` または `gpg2` コマンドが利用できれば準備完了です。

以下は NixOS で GnuPG をインストールした場合の実行例です。
筆者の環境では、`gpg2` と `gpg` が同じ実体を指していることが確認できました。

```shellscript
yama@tnt ~> gpg --version
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

yama@tnt ~> which gpg
/etc/profiles/per-user/yama/bin/gpg
yama@tnt ~> which gpg2
/etc/profiles/per-user/yama/bin/gpg2

yama@tnt ~> readlink $(which gpg)
/nix/store/2rkpckzry5slhzc92f26i436z0lkrkks-home-manager-path/bin/gpg
yama@tnt ~> readlink $(which gpg2)
/nix/store/2rkpckzry5slhzc92f26i436z0lkrkks-home-manager-path/bin/gpg2
```

## 事前知識

### 暗号方式について
GnuPG には大きく分けて2つの暗号化方式があります。
- 対称暗号 (共通鍵暗号) パスフレーズ(symmetric passphrase)を用いる方法
- 非対称暗号 (公開鍵暗号) 公開鍵/秘密鍵ペアを用いる方法

本記事では、`--symmetric` オプションを用いた対称暗号方式に限定して解説します。

### --pinentrymode オプション
ファイルの暗号化および復号化の際にはパスフレーズの入力が必要になります。

GnuPG のデフォルト設定 `--pinentry-mode ask` では、OSごとの `pinentry` ダイアログプログラムが起動します。しかし、これには別途ツールのインストールや設定が必要になり、環境構築が煩雑になりがちです。

そこで本記事では、パスフレーズの入力を実行中のターミナル上で行う `--pinentry-mode loopback` を指定します。

#### `man gpg` によるオプション解説
`man gpg` における `--pinentry-mode` の説明は以下の通りです。
```shellscript
> man gpg
  ...
--pinentry-mode mode
	Set the pinentry mode to mode.  Allowed values for mode are:
    default
		Use the default of the agent, which is ask.
		ask    Force the use of the Pinentry.
		cancel Emulate use of Pinentry's cancel button.
        error  Return a Pinentry error (``No Pinentry'').
        loopback
			Redirect Pinentry queries to the caller.
			Note that in contrast to Pinentry the user is not prompted again if he enters a bad password.
```

## ファイルの暗号化手順
実際に対称暗号方式でファイルを暗号化してみます。
以下のコマンドを実行してください。
`-c` は `--symmetric` の短縮オプションです。

```shellscript
gpg --pinentry-mode loopback --symmetric file
# または
gpg --pinentry-mode loopback -c file
```

実行すると、パスフレーズの入力が求められます。
完了すると元のファイルと同じディレクトリに、
`.gpg` の拡張子がついた暗号化ファイル `file.gpg` が生成されます。

:::message alert
**注意** `--pinentry-mode loopback` オプションの指定を忘れないようにして下さい。
:::

### 実行例
`/tmp/foobar` というファイルを作成し、暗号化する流れは以下になります。
下記のように、`cat -v /tmp/foobar.gpg` ファイルの中身が暗号化されているのが分かります。
```shellscript
yama@tnt /tmp> pwd
/tmp
yama@tnt /tmp> echo "This is foobar" > /tmp/foobar
yama@tnt /tmp> cat foobar
This is foobar
yama@tnt /tmp> gpg --pinentry-mode loopback -c /tmp/foobar
パスフレーズを入力:
yama@tnt /tmp> ls /tmp/foobar*
/tmp/foobar  /tmp/foobar.gpg
yama@tnt /tmp> cat -v /tmp/foobar.gpg
M-^L^M^D        ^C
M-^LM-^NM-&M-^S{1M-AM-(M-rM-RD^A'wM-QM-/!YM-|M-FM-xWP8i^VM-^GM-^@_M-)^XM-QM-zM-o^\M-v^V1vM-EeM- M-^VM-Ii-^ZM-^QM-<M-^[M-hLM-^CM-^FM-LlM->M-cM-KM-b
M-^^GM-j19M-E^_M-sM-^AM-f~M-,;M-^AFM-^KbM-}
```

#### 注意点 `--pinentry-mode loopback` オプションを忘れた場合
このオプション指定を忘れると、ターミナル上でパスフレーズを受け取ることができず、以下のエラーで失敗します。注意して下さい。
```shellscript
yama@tnt /tmp> echo "This is foobar" > /tmp/foobar
yama@tnt /tmp> gpg -c /tmp/foobar
gpg: エージェントに問題: Pinentryがありません
gpg: パスフレーズの作成エラー: 操作がキャンセルされました
gpg: '/tmp/foobar'の共通鍵暗号に失敗しました: 操作がキャンセルされました
```

## ファイルの復号化手順
次に、暗号化したファイルを復号して内容を表示します。

```shellscript
gpg --pinentry-mode loopback -d file.gpg
# または
gpg --pinentry-mode loopback --decrypt file.gpg
```

暗号化時と同様に
`--pinentry-mode loopback` オプションの指定を忘れないようにしてください。

### 実行例
```shellscript
yama@tnt /tmp> gpg --pinentry-mode loopback -d /tmp/foobar.gpg
gpg: AES256.CFB暗号化済みデータ
gpg: 1 個のパスフレーズで暗号化
This is foobar
yama@tnt /tmp> gpg --pinentry-mode loopback --decrypt /tmp/foobar.gpg
gpg: AES256.CFB暗号化済みデータ
gpg: 1 個のパスフレーズで暗号化
This is foobar
```

### 元ファイルの削除
デフォルトでは元の平文ファイルは削除されません。
暗号化と復号化を確認したら、暗号化前の元ファイル(平文ファイル)は削除しておきましょう！

## まとめ
当初 `Pinentryがありません` というエラーメッセージの原因が分からず、
`Pinentry` プログラムを別途インストールしたり、`gpg-agent` の設定したりと試行錯誤しました。

https://qiita.com/ychubachi/items/d20552de5a61683e6d93
上記記事を参考に Emacs EasyPG の `epg-pin-entrymode` 変数の定義を確認したところ、GnuPG の `pinentry-mode` オプションを知りました。
```elisp:epg-config.el
;; In the doc string below, we say "symbol `error'" to avoid producing
;; a hyperlink for `error' the function.
(defcustom epg-pinentry-mode nil
  "The pinentry mode.

GnuPG 2.1 or later has an option to control the behavior of
Pinentry invocation.  The value should be the symbol `error',
`ask', `cancel', or `loopback'.  See the GnuPG manual for the
meanings.

A particularly useful mode is `loopback', which redirects all
Pinentry queries to the caller, so Emacs can query passphrase
through the minibuffer, instead of external Pinentry program."
  :type '(choice (const nil)
		 (const ask)
		 (const cancel)
		 (const error)
		 (const loopback))
  :version "27.1")
```

`loopback` モードを利用することで、外部プログラムを介さずに呼び出し元(今回の検証ではターミナル、Emacs上ではミニバッファ)からパスフレーズを入力できるようになります。

同じようなエラーで困っている方は、まずはこの `loopback` オプションを試してみることをおすすめします。
