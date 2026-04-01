---
title: "NotebookLMの代替に？Open Notebook + Ollama でローカルLLM環境を構築する"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [ "AI", "opennotebook", "Ollama" ]
published: false
---

## はじめに

Google NotebookLM は便利ですが、機密情報や個人情報の取り扱いに注意が必要です。
プライバシーを確保しつつローカル環境で動作する Open Notebook を知り、以前から利用していたローカル LLM `Ollama` を組み合わせて導入してみました。
本記事では、セットアップから動作確認に至る過程で躓いた点も含め、備忘録としてまとめました。

## 環境
機種: MacBook Air M4
メモリ: 32GB
OS: macOS Tahoe バージョン26.3.1

:::details hostinfo
```shell
$ hostinfo
Mach kernel version:
         Darwin Kernel Version 25.3.0: Wed Jan 28 20:54:55 PST 2026; root:xnu-12377.91.3~2/RELEASE_ARM64_T8132
Kernel configured for up to 10 processors.
10 processors are physically available.
10 processors are logically available.
Processor type: arm64e (ARM64E)
Processors active: 0 1 2 3 4 5 6 7 8 9
Primary memory available: 32.00 gigabytes
Default processor set: 788 tasks, 4106 threads, 10 processors
Load average: 1.35, Mach factor: 8.63
```
:::

本記事では Nix を用いた インストール手順を説明しますが、
`Homebrew` など、ご自身の環境に合わせたパッケージマネージャーに適宜読み替えて進めてください。

## Ollama のインストール

ローカル LLM の [Ollama](https://github.com/ollama/ollama) をインストールし、バックグラウンドでサービスが起動している状態にします。

以下は Nix の Home Manager を用いたモジュール設定になります

```nix:ollama.nix
{ config, pkgs, ... }:

{
  home.packages = with pkgs; [
    ollama
  ];
  services.ollama.enable = true;
}
```

設定後、`sudo darwin-rebuild switch` あるいは `home-manager switch` を実行してシステムに反映させ、正常に起動することを確認してください。

## Docker インストール

Open Notebook の [公式ドキュメント Prerequisites](https://github.com/lfnovo/open-notebook?tab=readme-ov-file#prerequisites) では `Docker Desktop` のインストールが推奨されています。

しかし、本環境では、Nix Darwin および  Home Manager の darwinModule を利用しているため `Docker Desktop` はインストールできないので `Docker CLI` をインストールします。

ただし、macOS では Linux カーネル機能を直接利用できないため、 `Colima` を導入し、 `Docker CLI` を利用可能にします。
（Linux カーネルが直接利用できる NixOS 等の環境では、この手順は不要です。）

なお、 `Colima` 自体も  Nix でインストール可能です。

Nix で `Docker` 、`Colima` をインストールするモジュール設定は以下の通りです。

```nix:docker.nix
{ config, pkgs, ... }:

{
  home.packages = with pkgs; [
    docker
    colima
  ];
}
```
設定後、`sudo darwin-rebuild switch` あるいは `home-manager switch` を実行してシステムに反映させてください。

### `Colima` と `Docker` の動作確認

インストールが完了したら、以下の手順で動作を確認します。
`Hello from Docker!` と表示されれば正常です。

:::details `Colima` と `Docker` の動作確認
```shell
$ cd /tmp

/tmp$ colima start
INFO[0001] starting colima
INFO[0001] runtime: docker
INFO[0001] creating and starting ...                     context=vm
INFO[0002] downloading disk image ...                    context=vm
INFO[0113] provisioning ...                              context=docker
INFO[0115] starting ...                                  context=docker
INFO[0115] done

/tmp$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
58dee6a49ef1: Pull complete
c3bdf82c34d1: Download complete
Digest: sha256:452a468a4bf985040037cb6d5392410206e47db9bf5b7278d281f94d1c2d0931
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm64v8)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```
:::

## Open Notebook の Docker 設定

[Step 1: Get docker-compose.yml](https://github.com/lfnovo/open-notebook?tab=readme-ov-file#step-1-get-docker-composeyml) に従って実行します。

`open-notebook` ディレクトリの作成と `docker-compose.yml` のサンプルファイルを取得します。

```shell
$ cd
~$ mkdir open-notebook
~$ cd open-notebook/
~/open-notebook$ curl -o docker-compose.yml https://raw.githubusercontent.com/lfnovo/open-notebook/main/docker-compose.yml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1065  100  1065    0     0   2675      0 --:--:-- --:--:-- --:--:--  2682
~/open-notebook$ ls
docker-compose.yml
```

1. 暗号化キーの設定

`docker-compose.yml` 中の
- `OPEN_NOTEBOOK_ENCRIPTION_KEY` の設定
  - `openssl rand -hex 32` で生成した文字列に変更

2. Ollama 接続設定の追加
- `- OLLAMA_BASE_URL=http://host.docker.internal:11434` の追加

```diff
~/open-notebook (master)$ git diff docker-compose.yml
diff --git a/docker-compose.yml b/docker-compose.yml
index 47c59ff..727eb48 100644
--- a/docker-compose.yml
+++ b/docker-compose.yml
@@ -20,7 +20,7 @@ services:
     environment:
       # REQUIRED: Change this to your own secret string
       # This encrypts your API keys in the database
-      - OPEN_NOTEBOOK_ENCRYPTION_KEY=change-me-to-a-secret-string
+      - OPEN_NOTEBOOK_ENCRYPTION_KEY=xxxx309f71ff2b1bfec4b439164c1c0bba2ce749dc57b6ae6da4d2821b24dba2d

       # Database connection (default values - no need to change)
       - SURREAL_URL=ws://surrealdb:8000/rpc
@@ -28,6 +28,7 @@ services:
       - SURREAL_PASSWORD=root
       - SURREAL_NAMESPACE=open_notebook
       - SURREAL_DATABASE=open_notebook
+      - OLLAMA_BASE_URL=http://host.docker.internal:11434
     volumes:
       - ./notebook_data:/app/data
     depends_on:
```

## Open Notebook Docker サービス起動

:::message
Mac の場合 `colima start` の実行を忘れないようにして下さい！
:::

```shell
~/open-notebook$ colima start
INFO[0000] starting colima
INFO[0000] runtime: docker
INFO[0001] starting ...                                  context=vm
INFO[0008] provisioning ...                              context=docker
INFO[0009] starting ...                                  context=docker
INFO[0009] done

~/open-notebook$ docker compose up -d
[+] up 5/5
 ✔ Image lfnovo/open_notebook:v1-latest    Pulled                                                                                               2.3s
 ✔ Image surrealdb/surrealdb:v2            Pulled                                                                                               3.3s
 ✔ Network open-notebook_default           Created                                                                                              0.0s
 ✔ Container open-notebook-surrealdb-1     Created                                                                                              0.0s
 ✔ Container open-notebook-open_notebook-1 Created                                                                                              0.0s
```

## Open Notebook 起動

ブラウザで `http://localhost:8502` を開く
以下の画面が表示されます。

左ペインの `言語` ボタンで表示言語を設定できます。


![](/images/opennotebookinitial.png)

私の環境では、画面が真っ白になり何も表示されない不具合が発生しました。
PC を再起動後に再度実行したところ、無事に Open Notebook の画面が表示されました。

## Open Notebook 設定

### AI プロバイダー設定

Open Notebook で使用する以下の Ollama のモデル

チャットモデル: `gemma3:1b`
Embeddingモデル: `nomic-embed-text:latest`
トランスフォーメーションモデル: `gemma3:1b`

を

`ollama pull gemma3:1b`
`ollama pull nomic-embed-text:latest`

を実行し、事前にインストールしておきます。

:::details ollama の gemma3:1b, nomic-embed-text:latest モデルインストール
```shell
yama@roswell ~/open-notebook (master)> ollama list
NAME                       ID              SIZE      MODIFIED
gpt-oss:20b                17052f91a42e    13 GB     5 months ago
gemma3:latest              a2af6cc3eb7f    3.3 GB    7 months ago

~/open-notebook$ ollama pull gemma3:1b
pulling manifest
pulling 7cd4618c1faf: 100% ▕████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▏ 815 MB
pulling e0a42594d802: 100% ▕████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▏  358 B
pulling dd084c7d92a3: 100% ▕████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▏ 8.4 KB
pulling 3116c5225075: 100% ▕████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▏   77 B
pulling 120007c81bf8: 100% ▕████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▏  492 B
verifying sha256 digest
writing manifest
success

~/open-notebook $ ollama pull nomic-embed-text:latest
pulling manifest
pulling 970aa74c0a90: 100% ▕████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▏ 274 MB
pulling c71d239df917: 100% ▕████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▏  11 KB
pulling ce4a164fc046: 100% ▕████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▏   17 B
pulling 31df23ea7daa: 100% ▕████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▏  420 B
verifying sha256 digest
writing manifest
success

~/open-notebook $ ollama list
NAME                       ID              SIZE      MODIFIED
nomic-embed-text:latest    0a109f422b47    274 MB    8 seconds ago
gemma3:1b                  8648f39daa8f    815 MB    2 minutes ago
gpt-oss:20b                17052f91a42e    13 GB     5 months ago
gemma3:latest              a2af6cc3eb7f    3.3 GB    7 months ago
```
:::

後で気づいたのですが、[Model Provider Support](https://www.open-notebook.ai/features/model-providers.html) のページに推奨のモデルがあります。

### Open Notebook の AIプロバイダー設定
1. 左ペインの `モデル` を選択
2. Ollama の `+ 設定を追加` を押下
- 設定名は任意です。 ここでは、`Ollama Local` としました。
- APIキー 不要です。
- ベースURL は `docker-compose.yml` で指定した `http://host.docker.internal:11434` を指定します

3. `設定を追加` を押します。

![](/images/opennotebookollamasetup.png)

### 接続チェック

コンセントマークを押すと、指定した AI プロバイダーとの接続チェックが行われます。
無事接続が行われると、緑のチェックが入ります。

![](/images/opennotebookollamaconnectioncheck.png)

### AI プロバイダーの `Language` , `Embedding` モデルの設定

#### Language モデル設定

`gemma3:1b` を設定

![](/images/opennotebookollamaLanguageModel.png)

#### Embedding モデル設定
`nomic-embed-text:latest` を設定
![](/images/opennotebookollamaEmbeddingModel.png)


### デフォルトモデル割り当て の設定
チャットモデル、Embeddingモデル、トランスフォーメーションモデルを選択

![](/images/opennotebookDefaultModelsetting.png)


以上で、Open Notebook の設定は終了です。

## Open Notebook 使い方

### ソースの追加

一例として、自身で作成したマークダウンファイル `数学.md` を追加してみました。

左ペインの `新規` をクリックし、新規ソースダイアログを開き、ファイルをアップロードし、完了ボタンを押します。

:::message
PDF のソースを指定する場合には、注意してください！
アプリケーションから生成した PDF の場合、以降で説明する `Embedding済み` の状態が `はい` になりますが、
OCR PDF の場合、 `Processing...` のままになります。
:::

![](/images/opennotebookaddsource.png)


アップロードしたソースは、`notebook_data/upload` の下に配置されます。

```shell
~/open-notebook (master)$ ls notebook_data/uploads/
数学.md
```
:::message
チャットの応答が「ローディング状態」で進まない場合の対処法
gemma3:latest（約3.3GB）のような比較的大きなモデルを使用すると、推論に時間がかかり（例：約33秒）、画面が固まったようになることがあります。
このような場合、ブラウザの再読み込みが有効です。リロードすることで、バックグラウンドで生成されていた応答が正しく表示されます。
根本的な解決策として `gemma3:1b` などの軽量モデルを選択すれば、よりスムーズで軽快なレスポンスが得られます。
:::

### ソースの Embedded 確認

`Embedded済み` が `はい` になったことを確認。
ソースを追加してから、`はい` になるまで、少し時間がかかります。

![](/images/opennotebooksourceembedded.png)

### ソースを元にチャット

ソースをクリックし、チャットを開始。

`逆関数とは` とチャットで質問しましたが、英語で応答されました。

:::message
日本語で応答させるには、
`日本語で、逆関数とは` と、`日本語で` を追加します。
:::

![](/images/opennotebookchat.png)

### ノートブックの作成

左ペインから `ノートブック` を選択し、`+ 新規ノートブック` をクリックし、ノートブック名を入力します。

![](/images/opennotebookcreatenote.png)

`ソースを追加` リストボタンをクリックし、新規のソースか既存ソースを選択することで、ノートブックを作成できます。

![](/images/opennotebookaddsourcefromnotebook.png)

## うまく動作しない場合

設定後に期待通りに動作しない、あるいはチャットの応答が極端に遅い場合は、以下のコマンドでコンテナのログを確認して対処してださい。

```shell
~/open-notebook (master)$ docker compose logs open_notebook
```
ログ中の時刻表示は `UTC` なので、日本時間では `UTC+9` する必要があります。

## まとめ

Open Notebook を利用することで、Google NotebookLM と類似の環境をローカル環境で構築できました。
特に、Ollama と組み合わせることで、機密情報や個人情報を含むデータを扱う場合でも、外部サービスにデータを送信することなく、安心して LLM を利用できる点は大きなメリットです。
一方で、Google NotebookLM と比較すると、機能面や操作性においてはまだ発展途上な部分もあり、特にモデルの選択やレスポンス速度については環境に依存する点に注意が必要です。
Open Notebook を導入を検討している方の参考になれば幸いです。
