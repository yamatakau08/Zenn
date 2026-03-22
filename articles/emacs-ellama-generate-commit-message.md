---
title: "Emacs Ellama を用いて Magit で Git コミットメッセージを生成する"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "Emacs", "Ellama", "Magit", "Git" ]
published: false
---

## はじめに

Git のコミットメッセージを考えるのは、意外と悩みがちです。
そこで、コミットメッセージを LLM（大規模言語モデル）に生成させてみることにしました。

Emacs では、Git のフロントエンドとして Magit、LLM のフロントエンドとして Ellama があります。

Magit のコミットメッセージ用バッファ (COMMIT_EDITMSG) で、Ellama の `ellama-generate-commit-message` 関数を呼び出すだけで、
コミットメッセージ案を生成できます。

## 環境
Emacs
- パッケージ
  - [Magit](https://github.com/magit/magit)
  - [Ellama](https://github.com/s-kostyaev/ellama)
- LLM プロバイダー (ollamaなど)

パッケージのインストールと設定は本記事では扱いません。

## 手順
Git 管理下のディレクトリあるいはファイルを開いた状態で、以下の手順を実行します。

1. `C-x g` を入力し `magit-status` を実行します。
2. Magit バッファでコミット対象のファイルに対して `s` を押し、ステージングに登録します。
3. `c c` を押し `magit-commit-create` を実行します
4. `COMMIT_EDITMSG` バッファが開いたら、`M-x ellama-generate-commit-message` を実行します。

実行後、LLM によってコミットメッセージ案が自動生成されます。

## コミットメッセージのテンプレート

生成されるコミットメッセージの形式は、`ellama.el` 内で定義されているユーザー設定可能な変数 `ellama-generate-commit-message-template` で制御されます。

デフォルトのテンプレートは以下の通りです。

```elisp:ellama.el
(defcustom ellama-generate-commit-message-template "<INSTRUCTIONS>
You are a professional software developer.

Write a concise commit message based on a diff in the following format:
<FORMAT>
The first line should contain a short title describing major changes in functionality.
Then, add one empty line. Then, add a detailed description of all changes.
</FORMAT>
<EXAMPLE>
Improve abc

Improved the feature abd by adding a new module xyz.
</EXAMPLE>

**Reply with the commit message only, and without any quotes.**
</INSTRUCTIONS>

<DIFF>
%s
</DIFF>"
  "Prompt template for `ellama-generate-commit-message'."
  :type 'string)
```

## コミットメッセージの形式変更

`ellama-generate-commit-message-template` を変更することで、好みの形式に変更できます。

たとえば、コミットメッセージを 1 行に限定したい場合は、プロンプト内の `<FORMAT> ... </FORMAT>` 節から `Then, add one empty line. Then, add a detailed description of all changes.` という指示を削除することで実現できます。

以下は、`use-package` を用いて、`:custom` 節で `ellama-generate-commit-message-template` を設定する例です。

```elisp
(use-package ellama
  :ensure t

  :custom
  (ellama-generate-commit-message-template "<INSTRUCTIONS>
You are a professional software developer.

Write a concise commit message based on a diff in the following format:
<FORMAT>
The first line should contain a short title describing major changes in functionality.
</FORMAT>
<EXAMPLE>
Improve abc
</EXAMPLE>

**Reply with the commit message only, and without any quotes.**
</INSTRUCTIONS>

<DIFF>
%s
</DIFF>"
   "Prompt template for `ellama-generate-commit-message'.")

  ;;...
)
```

また、日本語のコミットメッセージを生成したい場合は、プロンプト内 `<FORMAT> ... </FORMAT>` の指示に `in Japanese.` を追加し、
`The first line should contain a short title describing major changes in functionality in Japanese.`
とすることで実現できます。

```elisp
(use-package ellama
  :ensure t

  :custom
  (ellama-generate-commit-message-template "<INSTRUCTIONS>
You are a professional software developer.

Write a concise commit message based on a diff in the following format:
<FORMAT>
The first line should contain a short title describing major changes in functionality in Japanese.
</FORMAT>
<EXAMPLE>
Improve abc
</EXAMPLE>

**Reply with the commit message only, and without any quotes.**
</INSTRUCTIONS>

<DIFF>
%s
</DIFF>"
   "Prompt template for `ellama-generate-commit-message'.")

  ;;...
)
```

## ellama-generate-commit-message 関数とコミットメッセージ形式の切り替え

`ellama-generate-commit-message` の関数定義は以下の通りです。

```elisp:ellama.el
(defun ellama-generate-commit-message ()
  "Generate commit message based on diff."
  (interactive)
  (save-window-excursion
    (when-let* ((default-directory
		 (if (string= ".git"
			      (car (reverse
				    (cl-remove
				     ""
				     (file-name-split default-directory)
				     :test #'string=))))
		     (file-name-parent-directory default-directory)
		   default-directory))
		(diff (or (ellama--diff-cached)
			  (ellama--diff))))
      (ellama-stream
       (format ellama-generate-commit-message-template diff)
       :provider ellama-coding-provider))))
```

内部で `(format ellama-generate-commit-message-template diff)` を呼び出し、テンプレート `ellama-generate-commit-message-template` を用いて LLM のプロンプトを構成しています。

なので、用途（英語用、 1 行用、日本語 1行用など）ごとに、テンプレート変数とそれを呼び出す関数を用意すれば、コマンド実行時にコミットメッセージの形式を切り替えられます。


## まとめ
Emacs の Ellama には、LLM を介して Git のコミットメッセージを生成する関数が用意されています。
既存の関数の紹介ですが、Magit と併用することで、簡単にコミットメッセージを生成してくれるのは非常に便利です。

自身も Git でコードを管理していますが、自分しか使わないこともあり、コミットメッセージを考えるのが面倒で内容をおろそかにしていました。
しかし、`ellama-generate-commit-message` を使い始め、diff の内容から適切なコミットメッセージを生成してくれる便利さを実感し、現在は常用するようになりました。

コミットメッセージの内容を考える手間が面倒だと感じている方は、ぜひ試してみてください。
