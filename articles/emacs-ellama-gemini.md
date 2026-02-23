---
title: "Emacs の Ellama から Gemini 無料枠を利用する"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "Emacs", "Ellama", "Gemini" ]
published: true
---

## はじめに
Emacs からローカル LLM パッケージである `ellama` を利用しようとしましたが、
非力な PC では動作が非常に重く、実用的ではありません。
そのため、`ellama` の利用は控え、ブラウザ版 Gemini を利用していました。

今回 `ellama` 経由でも Gemini 無料枠を利用できることを知りました。
しかし [ellama 公式リポジトリ](https://github.com/s-kostyaev/ellama) の README では設定方法が分かりづらく、導入までに時間を要しました。

本記事では、同様に検討している方に向けて、設定手順を備忘録としてまとめす。

## 前提条件
- Emacs で `ellama` を導入済み、あるいは利用を検討していること
- Gemini API キーを取得済みであること (取得手順については後述します)

## 環境
### Emacs
現在使用している Emacs のバージョンは以下の通りです。
```elisp
emacs-version
;; => "30.2"
```

### ellama
使用している `ellama` パッケージの情報は以下の通りです。
```elisp:ellama.el
;;; ellama.el --- Tool for interacting with LLMs -*- lexical-binding: t -*-

...

;; Package-Requires: ((emacs "28.1") (llm "0.24.0") (plz "0.8") (transient "0.7") (compat "29.1"))
;; Package-Version: 20250918.1606
;; Package-Revision: 2857f85b8f10
```

## Gemini 無料枠利用のための事前準備

Gemini API を利用するために、Google AI StudioでAPIキーを取得します。

### Gemini APIキーの作成
1. [Google AI Studio](https://aistudio.google.com/app/api-keys) にアクセスします。
2. 画面左ペインの `API キー` を選択します。
3. 画面右上の `API キーを作成` ボタンをクリックします。

「新しいキーを作成する」ダイアログが表示されたら、以下の操作を行います。
- 「キー名の設定」に `Gemini API Key` など任意の名前を入力します。
- 「インポートしたプロジェクトを選択」のリストから `+ プロジェクトを作成` を選択します。
	- 「新しいプロジェクト作成」ダイアログで、「プロジェクトに名前をつける」のフィールドに `Gemini Project` など任意の名前を入力します。
	- 「プロジェクトを作成」ボタンをクリックする

### Gemini APIキーの確認
`API キー` 画面の `キー` をクリックすると、Gemini API Key が表示されます。
![](/images/gemini-ai-studio-api-key.png)

## 無料枠で使用可能な Gemini モデル名
[Gemini Developer APIの料金](https://ai.google.dev/gemini-api/docs/pricing?hl=ja) を参照し、「無料枠」の列が 無料 と記載されているモデルを選択します。
後述の設定ファイルに指定するモデル名は、各モデル名の下に記載されている文字列を使用して下さい。

以下に `Gemini 2.5 Flash` の該当箇所を抜粋しています。
モデル名は、青枠で示されている文字列になります。
![](/images/gemini-api-gemini-2.5-flash-price.png)

:::message
「無料」と記載があっても、無料枠で使用できないものや、
[Gemini APIのすべてのモデル](https://ai.google.dev/gemini-api/docs/models?hl=ja&_gl=1*183dft7*_up*MQ..*_ga*NDE3OTE4NjkwLjE3NzE2OTM2MjE.*_ga_P1DBVKWT6V*czE3NzE2OTM2MjEkbzEkZzAkdDE3NzE2OTQ2MDEkajYwJGwwJGgxNDE1NjU5NDcw) の「以前のモデル」セクションで非推奨になっているものがあります。導入前に最新のステータスを確認してください。
:::

### 無料枠で動作するモデル
:::message
2026年2月22日時点で、Gemini 無料枠で利用可能なモデルは以下の通りです。
:::

- `gemini-3-flash-preview`
	- スピードを重視して構築された Google の最もインテリジェントなモデル。最先端のインテリジェンスと優れた検索およびグラウンディング機能を組み合わせたものです。
- `gemini-2.5-flash`
	- 100 万トークンのコンテキスト ウィンドウをサポートし、思考予算を備えた初のハイブリッド推論モデル。
- `gemini-2.5-flash-lite`
	- 最も小型で費用対効果の高いモデル。大規模な使用向けに構築されています

### 無料枠で動作しないモデル
- `gemini-2.5-pro` **無料枠で使用不可**
	- コーディングや複雑な推論タスクに優れた、最先端の多目的モデル。

## 各種設定

### ~/.authinfo.gpg に Gemini API キーのエントリ追加

以降の設定では、Emacs の `auth-source` を用いて、Gemini API キーをパスワードとして取得します。
そのため、`~/.authinfo.gpg` に以下のエントリを追加してください。

`auth-source` を用いたパスワード管理については、以下の記事を参考にしてください。
https://zenn.dev/roswell/articles/emacs-auth-source

```console:~/.authinfo.gpg
machine gemini login gemini-api password MYAPIKEY
```

### llm パッケージのインストールとその設定
`ellama` のバックエンドとして動作する `llm` パッケージをインストールします。
Gemini を利用するためには、`(require 'llm-gemini)` で `make-llm-gemini` 関数を利用可能にする必要があります。
```elisp:llm.el
(use-package llm
  :ensure t

  :init
  (require 'llm-gemini))
```

### ellama パッケージ設定に Gemini の設定を追加
- `ellama-providers` 変数に Gemini の各モデルを追加します。
- API キーはセキュリティ上の理由から直接記述せず、 `auth-source-pick-first-password` 関数を用いて取得します。


```diff elisp:ellam設定ファイル
(use-package ellama
  :ensure t

  :init
  ;; setup key bindings
  ;;(setopt ellama-keymap-prefix "C-c e")
  ;; language you want ellama to translate to
  (setopt ellama-language "Japanese")
  (require 'llm-ollama)
  ;; normal conversation
  (setopt ellama-provider
	  (make-llm-ollama
	   :chat-model "gemma3:latest" ; gpt-oss-20b is too slow
	   :default-chat-non-standard-params '(("num_ctx" . 8192))))

+  (setopt ellama-providers
+	  '(("gemini-3-flash-preview" . (make-llm-gemini
+					 :key (auth-source-pick-first-password :host "gemini" :user "gemini-api")
+					 :chat-model "gemini-3-flash-preview"))
+	    ;; gemini-2.5-pro is not available free tier
+	    ("gemini-2.5-flash" . (make-llm-gemini
+				   :key (auth-source-pick-first-password :host "gemini" :user "gemini-api")
+				   :chat-model "gemini-2.5-flash"))
+	    ("gemini-2.5-flash-lite" . (make-llm-gemini
+				   :key (auth-source-pick-first-password :host "gemini" :user "gemini-api")
+				   :chat-model "gemini-2.5-flash-lite"))
+	    ))
)
```

## 使い方
設定完了後、以下の手順で Gemini モデルに切り替えてチャットを開始します。

1. モデルの選択

`M-x ellama-provider-select` を実行します。
ミニバッファに設定済みのプロバイダー一覧が表示されるので、使用したいモデルを選択してください。

![](/images/ellama-provider-select.png)

2. チャットの開始

`M-x ellama-chat` を実行します。
ミニバッファに対話内容を入力し、 Enter を押します。
![](/images/ellama-chat-start.png)

以下のように `ellama` を通して、Gemini 無料版での対話が開始されます。
![](/images/ellama-chat-gemini.png)

### エラー対応

エラーメッセージ中に `status:429` が表示される場合は、
`model:` の箇所を確認し、指定しているモデル名が正しいか確認してください。

モデル名が正しい場合は、以下の可能性があります。
- 非推奨のモデルを指定している
- 無料枠では使用できないモデルを指定している

### 非推奨モデル `gemini-2.0-flash` を指定した場合
```elisp
Error running timer: (error "Error calling the LLM: Problem calling GCloud Vertex AI: status: 429 message: You exceeded your current quota, please check your plan and billing details. For more information on this error, head to: https://ai.google.dev/gemini-api/docs/rate-limits. To monitor your current usage, head to: https://ai.dev/rate-limit.
* Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_requests, limit: 0, model: gemini-2.0-flash
* Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_requests, limit: 0, model: gemini-2.0-flash
* Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_input_token_count, limit: 0, model: gemini-2.0-flash
Please retry in 45.387738042s.")
```

### 有料枠モデル gemini-2.5-pro 指定した場合

```elisp
Error running timer: (error "Error calling the LLM: Problem calling GCloud Vertex AI: status: 429 message: You exceeded your current quota, please check your plan and billing details. For more information on this error, head to: https://ai.google.dev/gemini-api/docs/rate-limits. To monitor your current usage, head to: https://ai.dev/rate-limit.
* Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_requests, limit: 0, model: gemini-2.5-pro
* Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_requests, limit: 0, model: gemini-2.5-pro
* Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_input_token_count, limit: 0, model: gemini-2.5-pro
* Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_input_token_count, limit: 0, model: gemini-2.5-pro
Please retry in 45.335249014s.")
```

## まとめ
非力なPC環境ではローカル LLM を直接動作させるのは困難ですが、`ellama` を介して Gemini の無料枠を活用することで、使い慣れた Emacs 上で快適な AI 支援環境を構築できます。

ブラウザ版 Gemini と使い分ける手間が省け、エディタ内で完結して対話できる点は、開発や執筆の作業効率を大きく向上させます。

また、`ellama` は Gemini 以外にも多くの LLM プロバイダに対応しています。今回の設定を応用すれば、他の LLM も利用可能です。

Emacs での AI 活用を検討している方にとって、本記事の内容が少しでも参考になれば幸いです。

