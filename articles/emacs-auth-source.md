---
title: "設定ファイルにパスワードを直書きしない！Emacs auth-source の基本的な使い方"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "Emacs", "GnuPG", "EasyPG" ]
published: false
---

## はじめに
Emacs から Gemini を用いるにあたり Emacs の設定ファイル中に API キーを設定する必要が生じました。
しかし、設定ファイルを GitHub で管理している事もあり、API キーを直接書き込むことはセキュリティ上好ましくありません。
そこで、Emacs で認証情報を取り扱える `auth-source` パッケージを利用することにしました。

これまで `auth-source` パッケージを利用した経験がなかったため、
本記事では `auth-source` を用いたパスワード取得方法を中心に、その基本的な使い方を解説します。

## auth-source パッケージとは

`auth-source` は、Emacs に標準で組み込まれている認証情報を一元管理するためのパッケージです。
主に `~/.authinfo` などの設定ファイルを介して、ホスト名やユーザー名に基づいたパスワード等の取得が可能になります。

また、平文の `~/.authinfo` だけでなく GnuPG で暗号化した `~/.authinfo.gpg` からも利用できるため、認証情報を安全に運用することができます。

## 環境
### Emacs
```elisp
(emacs-version)
;; "GNU Emacs 30.2 (build 1, x86_64-pc-linux-gnu, GTK+ Version 3.24.51, cairo version 1.18.4)"
```

### auth-source パッケージ
Emacs に標準で組み込まれています。
詳細については、Emacs Info 内の `Auth-source` セクションに解説があります。

### EasyPG パッケージ
Emacs に標準で組み込まれています。
`auth-source` が利用する暗号化ファイル `~/.authinfo.gpg` を Emacs 上で扱う場合に使用します。
`EasyPG` の設定や使い方については、以下の記事で解説しているので、必要に応じて参照して下さい。
https://zenn.dev/roswell/articles/emacs-easypg

## `auth-source` が使用する認証情報ファイル

`auth-sources` 変数を変更していない場合、以下の順序でファイルが参照されます。
1. `~/.authinfo` (暗号化されていないテキストファイル)
2. `~/.authinfo.gpg` (GnuPG で暗号化されたファイル)
3. `~/.netrc` 

本記事では、セキュリティを考慮し、`GnuPG` で暗号化した `~/.authinfo.gpg` を使用します。

### 認証情報ファイルの書式

ファイル内の記述は `Netrc` file の形式に従います。
```console
machine MYMACHINE login MYLOGINNAME password MYPASSWORD port MYPORT
```

| トークン名                  | 指定内容               | 補足      |
| ---------------------- | ------------------ | ------- |
| machine                | ホスト名, DNS名, IPアドレス | −       |
| login / user / account | ログイン名/ユーザー名/アカウント名 | −       |
| password               | パスワード, API キー      | −       |
| port                   | ポート名あるいはポート番号      | 必要時のみ記述 |

認証情報は、1行に1つの認証情報エントリになります。
`auth-source` は各エントリに `machine` と `login` が定義されていることを前提にしているので、これらは省略せずに記述することを推奨します。

:::message
筆者の経験談
Gemini API を利用した際、プロジェクト名を `login` に、API キーを、`password` に設定し、
`machine` トークンを省略して記述したところ、後述するパスワード検索において、意図しない結果が返ってきてしまい、解決に時間を要しました。
:::
## ~/.authinfo.gpg ファイルの作成

Emacs で、以下のようなエントリを記述した `~/.authinfo.gpg` を作成します。
```console:~/.authinfo.gpg
machine hfoo login lfoo password hlfoopass
machine hbar login lbar password hlbarpass
machine hbaz login lbaz password hlbazpass port 1234
```

ファイル保存後、パーミッションを `600` (所有者のみ読み書き可能) に設定しておくのが望ましいです。
```shell
yama@tnt ~> ls -l .authinfo.gpg
-rw-r--r-- 1 yama users 126  2月 18 17:58 .authinfo.gpg
yama@tnt ~> chmod 600 .authinfo.gpg
yama@tnt ~> ls -l .authinfo.gpg
-rw------- 1 yama users 126  2月 18 17:58 .authinfo.gpg
```

## auth-source 挙動確認時の注意点
`auth-source` は `~/.authinfo.gpg` を読み込むとその内容をキャッシュします。
そのため、`*scratch*` バッファなどで動作確認を行っている際に `~/.authinfo.gpg` を更新しても、そのままでは変更が反映されません。

設定ファイルの変更を反映させるには、
`M-x auth-source-forget-all-cached` 
を実行して、キャッシュされた古い認証情報をクリアする必要があります。

## パスワード取得

`~/.authinfo.gpg` からパスワードを取得するには、
`auth-source-pick-first-password` 関数を使用します。

引数には、以下のキーと値を用い、`:host` と `:user` の両方を明示的に指定してください。
- `:host "MYMACHINE"`
- `:user "MYLOGINNAME"` 

### 使用上の注意点
1. 引数のキー名称について
キー名称は、必ず `:host` 及び `:user` を使用してください。
設定ファイルトークン名に合わせて `:machine` や `:login`, `:account` と指定しても検索キーとして認識されません。

2. 引数の省略とワイルドカード挙動
`:host` `:user` の両方を明示的に指定して下さい。
指定を省略した場合、そのフィールドはワイルドカード扱いとなり、すべてのエントリに適合してしまいます。

いずれも、意図しないエントリのパスワードが取得される原因となります。

### パスワード取得例

`*scratch*` バッファで動作確認した結果です。
```elisp
;; 正しい指定方法：host と user が一致するエントリを返す
(auth-source-pick-first-password :host "hfoo" :user "lfoo")
;; => "hlfoopass"

(auth-source-pick-first-password :host "hbar" :user "lbar")
;; => "hlbarpass"

(auth-source-pick-first-password :host "hbaz" :user "lbaz")
;; => "hlbazpass"

;; 条件に一致するエントリがない場合
(auth-source-pick-first-password :host "hbar" :user "lbaz")
;; => nil

;; 引数をすべて省略した場合：先頭のエントリが返る
(auth-source-pick-first-password)
;; => "hlfoopass"
```

#### キー名称を誤った場合の挙動
以下のように無効なキー名称を指定すると、検索条件による絞り込みが正しく行われません。
- 誤って `:machine` を用いた場合

```elisp
(auth-source-pick-first-password :machine "hbar" :login "lbaz")
;; => "hlfoopass"
```
`:machine` は有効なキーではないため、引数なしの状態と同じく「ファイル内の先頭エントリ」が取得されます。

- 誤って `:login` を用いた場合
```elisp
(auth-source-pick-first-password :host "hbar" :login "lbaz")
;; => "hlbarpass"
```
`:login` が無視され、`:host "hbar"` の条件のみで検索されます。その結果、ユーザー名が一致しないにもかかわらず、ホスト名が一致した最初のエントリが取得されてしまいます。

いずれのケースも、本来 `nil` が返ることを期待する場面ですが、誤ったパスワードが取得されてしまうため注意が必要です。

## ポート番号の取得
パスワード以外のフィールド（`port` など）を取得したい場合は、`auth-source-search` 関数を利用します。
```elisp
(plist-get (car (auth-source-search :max 1 :host "hbaz" :user "lbaz")) :port)
;; => "1234"
```
`auth-source-search` は条件にマッチしたエントリのリストを返すため、`car` で最初のエントリを取り出し、`plist-get` で目的のキー（`:port`）を指定して値を取得します。

## まとめ
Emacs で `auth-source` を利用することで、設定ファイル内に認証情報を平文で記述する必要がなくなります。
暗号化したファイルから安全に値を取り出す仕組みを整えておけば、
設定ファイルを GitHub などで公開する際も安心感が増し、セキュリティ面でもとても有効な対策になります。

以上、`auth-source` の基本的な使い方の説明でした。

Emacs Info を参照することで、さらに効果的な活用方法が解説されています。ぜひ自分にあった設定を探してみてください。
