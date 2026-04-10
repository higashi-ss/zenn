---
title: "コンテナからReact NativeのExpoを環境構築する方法"
emoji: "⚛️"
type: "tech"
topics: ["reactnative", "expo", "環境構築", "トンネル接続", "ngrok"]
published: true
published_at: 2026-04-14 06:00
publication_name: "secondselection"
---

## 1. はじめに

### 背景と課題

Expoは公式ドキュメント通りに進めればローカル環境（PC上に直接）での構築は非常に簡単です。
しかし、いざDockerコンテナの中で動かそうとすると、 **「スマホのExpo Goアプリに画面が映らない（接続できない）」** というネットワークの壁にぶつかる方もいるのではないでしょうか？
私は最初うまくいきませんでした…
この記事では、WSL2とDockerを組み合わせた環境に、スマホの実機確認までの環境構築手順と私がつまづいた部分を解説します。

また先にReact NativeとExpoについても簡単に説明いたします。
本記事は環境構築の解説記事ですので初めて触れる方も想定しています。React Native + ExpoはiOSアプリ、Androidアプリを1つのコードで両方のOSに対応できる「クロスプラットフォーム開発」を実現します。本来なら別々に必要な学習コストや開発時間を大幅に短縮できるため、本記事を機にぜひ始めてみてください！

### そもそもReact NativeとExpoって？

React Nativeは、Meta社（旧Facebook）が開発したフレームワークでReactを使ってiOS/Androidアプリを同時に作ることができます。
ExpoはReact Nativeの開発をさらにお手軽にするための開発支援ツール群です。
開発支援ツールは下記等がありますが、本記事では詳細説明を割愛いたします。

* Expo SDK : カメラや位置情報などのモバイル機能の実装を簡素化する専用ライブラリが使用できる
* Expo Go : スマホからQRコードを読み取るだけで実機確認ができる
* EAS : PC上にビルド環境がなくてもクラウド上でビルドができる

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

### システム構成図

本記事で紹介する環境構築が完了後のシステム構成図を記載します。

![画像](/images/expo_in_docker/system_diagram.drawio.png)

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
globallyとあるがコンテナ内だけの影響のためインストールの心配ありません。

```bash
? The package @expo/ngrok@^4.1.0 is required to use tunnels, would you like to install it globally? 
```

### 3-6. QR表示

ダウンロードが終わったら、QRが表示され読み取るとスマホからアプリが起動する。

#### 参考画像

* ターミナル画面
![画像](/images/expo_in_docker/expo_start.png)

* スマホ画面（ファーストビュー）
![画像](/images/expo_in_docker/expo_firstview_mobile.png)

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

2026年3月時点でExpoのportは8081のみです。
Expoの使用ポートを調べると「19000」「19001」という情報もありますが古い情報です。
現在は使用しません。

### そもそもコンテナ上だとQRコードが読み込めない原因は？

スマホからコンテナ内部の住所（IPアドレス）が見えないためです。
通常、スマホからPCのIPアドレスは見えますがコンテナのIPアドレスは、PC内部で独立し隠蔽されておりスマホから確認できません。
そのため、コンテナ上のExpoが発行したQRコードをスマホで読み取っても、アドレスエラーが発生します。

この問題を解決するのが**「トンネル化」**です。`@expo/ngrok`と`--tunnel`オプションを活用することで、外部からアクセス可能な専用経路（トンネル）を生成し、スマホとコンテナを繋ぐことが可能になります。

## 5. 最後に

今回の記事では、Dockerコンテナ上でExpoプロジェクトを立ち上げ、実機確認する方法を解説しました。
コンテナ環境からスマホとうまく繋がらない現象は`--tunnel`オプションと`@expo/ngrok`ライブラリを活用することで解決できます。

最後までお読みいただきありがとうございました。

## 6. 参考

@[card](https://reactnative.dev/)
@[card](https://expo.dev/)
@[card](https://docs.expo.dev/more/expo-cli/)
@[card](https://qiita.com/KyongminS/items/9056a169cdf0b24d599f)
