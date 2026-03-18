---
title: "Emacs ellama で Magit の Git commit メッセージを LLMに作成させる"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "Emacs", "ellama"]
published: false
---

## はじめに

Gitのコミットメッセージを考えるのは、意外と頭を悩ませるものです。
そこで、コミットメッセージの案をLLMに作成させてみることにします。
一方で、Emacsでは、GitフロントエンドとしてMagitを利用すると非常に便利です。
今回は、このMagitのコミットメッセージ用バッファで、Emacs の Ellama パッケージの ellama-generate-commit-message 関数を用いて、
Git のコミットメッセージ案を作成します。


## 環境
Emacs
Emacs パッケージ
- Magit
- Ellama

## 手順
Git 管理下のディレクトリにて、`C-g` で Magit status 実行
COMMIT_EDITMSG バッファが作成された、
`M-x ellama-generate-commit-message` を実行


##

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
```