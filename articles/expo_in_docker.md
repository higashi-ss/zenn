---
title: "コンテナからreact nativeのEXPOを環境構築する方法"
emoji: "⚛️"
type: "tech"
topics: ["reactnative", "expo", "環境構築", "トンネル接続", "ngrok"]
published: true
published_at: 2026-02-16 06:00
publication_name: "secondselection"
---

## 1. はじめに

### 背景と課題

Expoは公式ドキュメント通りに進めればローカル環境（PC上に直接）での構築は非常に簡単です。
しかし、いざDockerコンテナの中で動かそうとすると、 **「スマホのExpo Goアプリに画面が映らない（接続できない）」** というネットワークの壁にぶつかる方もいるのではないでしょうか？
私は最初うまくいきませんでした…
この記事では、WSL2とDockerを組み合わせた環境に、スマホの実機確認までの環境構築手順と私がつまづいた部分を解説します。

### 記事を読んでできること

* **最短ルートでの環境構築**: 手順通りに進めるだけで、Docker上のExpoとスマホの実機連携まで完了します。
* **「ハマりどころ」の回避**: 私が実際つまづいた点を3点紹介しておりますので今後のエラー回避の参考になります。

### 対象読者

* React Native + Expo初学者の方
* Node.jsや依存ライブラリをPC本体（ローカル）に直接入れたくない方。
* Docker環境でExpoはインストールできたのに、スマホからビルド（Expo Go）画面が映らなくて困っている方。

## 2. 事前準備

### 必要環境とツール

* Windows 11 + WSL2: Linux環境がセットアップ済みであること。
* Docker環境: WSL2上でDockerが動作する状態であること。
* VS Code: コンテナ内のファイルを編集するために推奨します。
* スマートフォン: iOSまたはAndroid（Expo Goアプリをインストール済み）。

## 3. 環境構築手順

### 3-1. WSL内に任意のディレクトリを作成

本記事では`myapp`とする。

### 3-2. `Dockerfile`,`docker-compose.yml`をコピー

作成した`myapp`ディレクトリ配下へ下記に記載した`Dockerfile`,`docker-compose.yml`をコピーしてください。
※今回使用する`Dockerfile`,`docker-compose.yml`は最小構成内容となります。

Dockerfile

```dockerfile: Dockerfile
# node.js(バージョン24)をインストール
FROM node:24

# コンテナ中に /app というディレクトリを自動作成
# それ以降の命令（npm install など）はすべてその中で実行
WORKDIR /app


# ポートの開放（Expoの通信用）
EXPOSE 8081

# dockerを起動待機中にする
CMD ["bash"]
```

docker-compose.yml

```yaml: docker-compose.yml
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

対応後以下のディレクトリ構造になっているか確認してください。

```Markdown
WSL
├── myapp
│   │─── Dockerfile
│   └── docker-compose.yml
```

### 3-3. コンテナ立ち上げ

`myapp`ディレクトリで下記コマンドを実行してください。コンテナが立ち上がります。

```bash
docker compose up -d
```

### 3-4. expoテンプレートプロジェクトをインストール

* 3-4-1. `temp`というディレクトリを作成しそこにexpoプロジェクトを作成する。
ライブラリはまだインストールしません。

```bash
docker compose exec app npx create-expo-app@latest temp --no-install
```

* 3-4-2. `temp`ディレクトリの中身を`app`ディレクトリに移動する。

```bash
docker compose exec app sh -c "mv temp/* . && mv temp/.* . 2>/dev/null; rmdir temp"
```

* 3-4-3. `app`ディレクトリにて必要なライブラリをインストールする。

```bash
docker compose exec app npm install
```

### 3-5. Expoサーバー起動

下記コマンドを実行しExpoサーバーが起動する。

```bash
docker compose exec app npx expo start --tunnel
```

初めてサーバーを起動する際は下記コマンドが表示されるのでyesを選択しダウンロードする。
globallyとあるがコンテナ内だけの影響のためインストールの心配なし。

```bash
? The package @expo/ngrok@^4.1.0 is required to use tunnels, would you like to install it globally? 
```

### 3-6. QR表示

ダウンロードが終わったら、QRが表示され読み取るとスマホからアプリが起動する。

参考画像
![画像](/images/expo_in_docker/expo_start.png)

## 4. つまずきポイント

過去に私自身がつまづいた部分を記載します。

### なぜtempディレクトリをわざわざ作成する必要があるのか？

`npx create-expo-app` というコマンドは、**空のディレクトリでしか実行できないから**です。
公式のセットアップ手順では元々空のディレクトリにexpoプロジェクトをインストールしていますが、コンテナを作成する場合、`Dockerfile`,`docker-compose.yml`があります。
そのため、`app`ディレクトリで`npx create-expo-app`を実行すると下記のようなエラーが発生します。

```bash
The directory app contains files that could conflict. Please try using a new directory name, or remove the files listed above
```

エラー回避のため、3-4-1、3-4-2ではわざわざ新規ディレクトリを作成しexpoプロジェクトをインストールするという、まどろっこしいやり方をしています。

### portsは8081だけでいいのか？

2026年3月16日時点でExpoのportは8081のみです。
Expoの使用ポートを調べたりAIに聞いたりすると「19000」「19001」という情報がありますが古い情報です。
現在は使用しません。

### トンネル化とは何か？

コンテナ内のExpoとスマホをネットワークで繋ぐための仕組みです。
通常、スマホからPCのIPアドレスは見えますがコンテナのIPアドレスはスマホから確認できません。
そのためコンテナ上で起動したExpoのQRコードを読み込んでもアドレスエラーが発生します。

Expoでは、`@expo/ngrok`というライブラリと`--tunnel`オプションを活用し専用のトンネルを作ってアクセスが可能になります。

## 5. 最後に

今回の記事では、Dockerコンテナ上でExpoプロジェクトを立ち上げ、実機確認する方法を解説しました。
コンテナ環境からスマホとうまく繋がらない現象は`--tunnel`オプションと`@expo/ngrok`ライブラリを活用することで解決できます。

最後までお読みいただきありがとうございました。
