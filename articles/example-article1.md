---
title: "コンテナからreact nativeのEXPOを環境構築する方法"
emoji: "⚛️"
type: "tech"
topics: ["ReactNative", "Expo", "環境構築", "トンネル", "WSL"]
published: true
published_at: 2026-02-16 06:00
publication_name: "secondselection"
---

## はじめに

* **背景と課題**

最近、React Native + Expo の環境を、WSL2上のDockerコンテナで構築しました。
Expoは公式ドキュメント通りに進めればローカル環境（PC直接）での構築は非常に簡単です。
しかし、いざ「Dockerコンテナ」の中で動かそうとすると、**「スマホのExpo Goアプリに画面が映らない（接続できない）」**というネットワークの壁にぶつかる方もいるのではないでしょうか？
この記事では、WSL2とDockerを組み合わせた環境に、スマホの実機確認を行うまでの環境構築手順と私自身、苦労したつまずきポイントを解説します。

* **記事を読んでできること**
  * **最短ルートでの環境構築**: 手順通りに進めるだけで、Docker上のExpoとスマホの実機連携まで完了します。
  * **「ハマりどころ」の回避**: コンテナ環境特有の通信エラーを解消や、見落としがちなミス対応の知見が得られます。

* **💡対象読者**

* React Native / Expo 初学者の方：これからアプリ開発を始めたいけれど、環境構築で挫折したくない。
* 「環境を汚したくない」派のエンジニア：Node.jsや依存ライブラリをPC本体（ローカル）に直接入れたくない。
* Docker環境でExpo Goが繋がらず困っている方：コンテナで起動はできたのに、スマホに画面が映らなくて「詰んだ」と感じている。

## 事前準備

* 必要な環境（ツール）
Windows 11 + WSL2: Linux環境がセットアップ済みであること。
Docker Desktop / Docker Engine: WSL2上でDockerが動作する状態であること。
VS Code: コンテナ内のファイルを編集するために推奨します。
スマートフォン: iOSまたはAndroid（Expo Goアプリをインストール済み）。

## 環境構築手順
1,WSL内に任意のディレクトリを1つ作成する（本記事ではmyappとする）
2,1の配下に 下記に記載したDockerfile,docker-compose.yml をコピーし設置してください

### Dockerfile

```Markdown: Dockerfile
# node.js(バージョン24)をインストール
FROM node:24

# コンテナ中に /app というフォルダを自動作成
# それ以降の命令（npm install など）をすべてその中で実行
WORKDIR /app


# ポートの開放（Expoの通信用）
EXPOSE 8081

#dockerを起動待機中
CMD ["bash"]
```

### docker-compose.yml

```Markdown: Dockerfile
# services:はコンテナの定義
services:
  app:
    build: .
    volumes:
      - .:/app
    ports:
      - "8081:8081"
    tty: true
    stdin_open: true
```

対応後以下のディレクトリ構造になっているか確認してください
WSL
├── myapp
│   │─── Dockerfile
│   └── docker-compose.yml

3, myappディレクトリで下記コマンドを実行してください
docker compose up -d
　　これでコンテナを立ち上げる
　
:::message
補足：下記コマンドでコンテナ内にnpxが入ってるか確認できます
```Markdown
docker compose exec app npx -v
//結果 11.6.2
```
:::

4,expoテンプレートプロジェクトをインストールする
4-1 tempというファイルを作成しそこにexpoプロジェクトを作成する（ライブラリはインストールしない）
docker compose exec app npx create-expo-app@latest temp --no-install

4-2tempフォルダの中身をappフォルダに移動する
docker compose exec app sh -c "mv temp/* . && mv temp/.* . 2>/dev/null; rmdir temp"

4-3 appフォルダにてpackage.jsonにそって必要なライブラリをインストールする
docker compose exec app npm install

補足　4-1であえてフォルダを作成してそこに入れたのは　npx create-expo-app というコマンドは、「まっさらな（何もファイルがない）フォルダ」にプロジェクトを作ることを前提としているから

　　
5,以下のコマンドでコンテナ内でサーバーを起動（トンネル化が必要）
docker compose exec app npx expo start --tunnel

初めてサーバーを起動する際は下記コマンドが表示されるのでyesを選択しダウンロードする
globallyとあるがコンテナ内だけのインストールなので心配なし

? The package @expo/ngrok@^4.1.0 is required to use tunnels, would you like to install it globally? 

6,ダウンロードが終わったら、QRが表示され読み取るとスマホからアプリを表示できる


## つまずきポイント

下記に過去につまづいた部分を記載しました

・なぜtempファイルをわざわざ作成する必要があるのか？

・portsは8081だけでいいのか？

・トンネル化とは何か？

## 最後に

今回の記事では、WSL2 + Docker環境でExpoを構築し、トンネル接続を使って実機確認を行う方法を解説しました。

コンテナ環境でのうまく繋がらない現象は@expo/ngrokライブラリを使用することで
簡単に実機確認できるようになります
最後までお読みいただきありがとうございました。










ーーーーーーーーーーーーーーーーーーーーーーーー

:::message

### 対象読者

* durable functionsの基本を理解したい方
* LambdaやStep Functionsのワークフローの改善／見直しを考えている方

:::


:::message alert

既存の通常Lambdaから切り替えは不可。
新規作成時のみ設定可能です。
(上記の3の手順を忘れて保存した場合、再作成になるのでご注意ください)

:::


## 4. 【step/wait】サーバーレスで「待つ」を実現する


-----------------------
↓
コード記載部分のコピペ

1. 構成のポイント：なぜスマホと繋がらないのか？
通常、ExpoはPCとスマホが同じWi-Fi（ローカルネットワーク）に繋がっていることを前提としています。 しかし、**「WSL2 + Docker」**の環境では、以下の図のようにネットワークが階層化されています。

物理ネットワーク（あなたのWi-Fi）

WSL2の仮想ネットワーク

Dockerコンテナのネットワーク

スマホから見ると、コンテナの中で動いているExpo CLIは「二重の壁」の向こう側に隠れてしまっているため、単純なIP指定では接続できないのです。

これを一撃で解決するのが、Expo公式が提供している 「Tunnel（トンネル）機能」 です。これを使うと、インターネット経由でセキュアなエンドポイントを作成してくれるため、ネットワーク構成を気にせずスマホと接続できるようになります。

1. 【実践】DockerでExpo環境を構築する
それでは、実際に環境を作っていきましょう。

2.1. プロジェクト構造
適当な作業ディレクトリを作成し、以下の3つのファイルを用意します。

Plaintext
.
├── docker-compose.yml
├── Dockerfile
└── (プロジェクトファイルがここに生成されます)
2.2. Dockerfile の作成
Node.jsをベースに、Expo CLIの動作に必要なパッケージをインストールします。

Dockerfile

# Dockerfile

FROM node:20-slim

WORKDIR /app

# gitやプロセス管理に必要なツールをインストール

RUN apt-get update && apt-get install -y \
    git \
    openssl \
    && apt-get clean

# Expo CLIをグローバルにインストール

RUN npm install -g expo-cli

EXPOSE 8081

CMD ["/bin/bash"]
2.3. docker-compose.yml の作成
毎回長いコマンドを打たなくて済むよう、Composeで定義します。

YAML

# docker-compose.yml

services:
  app:
    build: .
    volumes:
      - .:/app
    ports:
      - "8081:8081"
    tty: true
    stdin_open: true
2.4. コンテナの起動と初期化
ターミナル（WSL2）で以下のコマンドを実行します。

Bash

# コンテナの起動

docker compose up -d --build

# コンテナ内に入る

docker compose exec app bash

# プロジェクトの作成（初回のみ）

# my-app は任意の名前に変えてください

npx create-expo-app my-app

# プロジェクトディレクトリへ移動

cd my-app
3. 重要：スマホ実機で確認するための設定
ここからが本題です。コンテナの中で起動したExpoをスマホで確認します。

3.1. Expo Go アプリの準備
お手持ちのスマートフォン（iPhone / Android）に 「Expo Go」 アプリをインストールしておいてください。

3.2. 魔法のコマンド：--tunnel
コンテナ内でプロジェクトを起動する際、普通に npx expo start と打つのではなく、以下のオプションを付けます。

Bash
npx expo start --tunnel
ここがポイント！

初めて実行する場合、@expo/ngrok のインストールを促されるので y を押して進めます。

実行後、ターミナルに大きな QRコード が表示されます。

このQRコードをスマホのカメラ（またはExpo Goアプリ）でスキャンしてください。

これで、インターネットを介してコンテナとスマホが接続され、アプリの画面がスマホに表示されるはずです！

1. WSL2環境でよくあるトラブルシューティング
4.1. ファイル監視数の上限エラー
ENOSPC: System limit for number of file watchers reached というエラーが出ることがあります。これはWSL2（Linux）側のファイル監視上限が低いために起こります。その場合は、WSL2のターミナルで以下を実行して上限を上げてください。

Bash
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
4.2. ホットリロードが効かない場合
Dockerのボリュームマウント経由だと、ファイルの変更検知がうまくいかないことがあります。その場合は、プロジェクト直下の app.json もしくは packge.json の設定を見直すか、コンテナを再起動してみてください。

まとめ
今回は、WSL2 + Docker環境でExpoを構築し、トンネル接続を使って実機確認する方法を解説しました。

この記事のまとめ

コンテナ環境ではネットワークの壁があるため、--tunnel オプションが必須。

Dockerを使えば、ローカル環境を汚さずに複数のReact Nativeプロジェクトを管理できる。

WSL2側のファイル監視設定（inotify）に注意。

次のアクション 環境が整ったら、まずは App.js のテキストを書き換えて、スマホ側の表示がリアルタイムで変わる感動を味わってみてください！そこから先は、NativeWindでスタイルを当てたり、React Navigationで画面遷移を作ったりと、あなたのアイデアを形にするだけです。

ハッピーコーディング！
