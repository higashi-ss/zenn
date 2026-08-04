---
title: "dockerでLocalStackを使用したAWSのローカル環境を構築"
emoji: "🐳"
type: "tech"
topics: ["docker", "aws", "localstack", "hooks"]
published: true
published_at: 2026-08-17 06:00
publication_name: "secondselection"
---

## はじめに

### 背景と課題

AWS環境で開発やテストを行う場合、RDSなどの従量課金制サービスを使用する度にコストがかかります。
LocalStackを使えば、実際のクラウド環境を使用することなく、ローカル上で無料かつ安全に開発・テストが可能です。
本記事では、チームメンバー全員が簡単に同じ環境を再現できるよう、Dockerを使った環境構築の手順を解説します。

### 対象読者

* AWSをローカル環境で試したい人
* LocalStackのdocker開発に苦戦している人

※　本記事ではdockerに関する基本的な用語や仕組みについての解説は省略いたします。

### 構築できる設定

* 

## 事前準備

### 必要環境とツール

* Windows 11 + WSL2: Linux環境がセットアップ済みであること。
* Docker環境: WSL2上でDockerが動作する状態であること。
* VS Code: コンテナ内のファイルを編集するために推奨します。

### システム構成

本記事で紹介する環境構築が完了後のシステム構成図を記載します。

![画像](/images/docker_local_aws/system_diagram.drawio.png)

### ディレクトリ構成

本記事のディレクトリ構成は以下になります。

```Markdown
myapp/
├── .devcontainer/
│   └── devcontainer.json
├── src/         # アプリケーションコード(今回は内容割愛)
├── Dockerfile
└── compose.yaml
```

## AWSのローカル開発

### LocalStackとは

LocalStackは、AWSのクラウド環境をローカルで擬似的に再現できるエミュレーションツールです。
S3、RDS、Lambda、SQSといった幅広い主要サービスに対応しています。
本番のAWS環境に依存しないため、以下のような大きなメリットがあります。

* **コスト削減**：ローカルで完結するため、無料テストできる
* **開発の高速化**：実際のAWS環境へデプロイする待ち時間がなく、素早くローカルでテストできる

:::message

なお、LocalStackには無料版と有料版が存在します。
本記事では無料版を使用した環境構築を解説します。無料版と有料版の機能の違いについては、以下をご参照ください。

:::

@[card](https://docs.localstack.cloud/aws/licensing/)

## 環境構築手順

それでは環境構築を実施します
今回はWSL上にlocalstackのコンテナ、開発・作業用コンテナの2つを準備します。

### Dockerfile作成

* コード

```dockerfile: Dockerfile
FROM public.ecr.aws/ubuntu/ubuntu:jammy

# 作業ディレクトリの指定
WORKDIR /app

# ubuntuの内部を日本時間に変更
RUN ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

RUN apt update
RUN apt install software-properties-common -y
# 必要なソフトをインストール　gitをインストールすることでコンテナ内でgitコマンド使用できます
RUN apt update && apt install -y curl git unzip

# AWS CLI v2 のインストール
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip awscliv2.zip \
    && ./aws/install \
    && rm -rf awscliv2.zip aws

RUN rm -rf /var/lib/apt/lists/*
RUN apt-get clean

# public.ecr.aws/ubuntu/ubuntu:jammy だと、git log が文字化けするので less コマンドの charset を指定する
ENV LESSCHARSET=utf-8

# 環境変数
ARG USER_NAME
ARG USER_UID
ARG USER_GID

RUN if [ "$USER_UID" -ne 0 ]; then \
        mkdir -p /etc/sudoers.d \
        && groupadd --gid $USER_GID $USER_NAME \
        && useradd -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USER_NAME \
        # パスワードなしで管理者コマンドを実行できる
        && echo "$USER_NAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USER_NAME \
        && chmod 0440 /etc/sudoers.d/$USER_NAME \
        # ディレクトリ所有者変更
        && chown -R $USER_NAME:$USER_NAME /home/$USER_NAME; \
    fi

# ユーザーの切り替えを実施
USER ${USER_NAME:-root}

#　バッシュで待機　コマンド操作できる
CMD ["bash"]
```

* コード解説

上記のDockerfileは単にツールを入れるだけでなく、「コンテナ内での操作性」と「権限エラーの回避」を考慮して作成しています。

* コンテナ内での操作性
  * コンテナ内からLocalStackにアクセスするためのawscliや、開発に必要なgitを導入しています
* 権限エラーの回避
  * コンテナ内をrootユーザーのまま操作すると、コンテナ内で作成・変更したファイルがroot所有になり、WSL側から編集できなくなります（権限エラーになる）。
  * 本環境ではコンテナ内でWSL側のユーザーと同じユーザーを作成しrootではなく自分がファイル操作・実行できるよう所有権を変更しています。

:::message

LocalStackの2026年4月以降のバージョンは、ユーザー登録が必須になりました。
今回はユーザー登録不要でシンプルに動作する過去バージョンを指定しています。

:::

### compose.yaml作成

* コード

```yaml: compose.yaml
services:
  my_app:
    build:
      context: .
      dockerfile: ./Dockerfile
      args:
        USER_NAME: username
        USER_UID: 1000
        USER_GID: 1000
    volumes:
      - .:/app    
    # コンテナ上でコマンド操作ができる
    tty: true
    # コンテナ上でキーボード操作ができる
    stdin_open: true
    depends_on:
      - localstack

  # S3 ローカル用
  # localstack
  localstack:
    image: localstack/localstack:4.14
    ports:
      - "4566:4566" 
    environment:
      SERVICES: s3,secretsmanager
      # デバッグログが出力される設定
      DEBUG: 1
      # docker.sockとLocalStackコンテナの中を共有する
      DOCKER_HOST: unix:///var/run/docker.sock
    volumes:
      # LocalStack内でコンテナを起動するためのdocker.sockの共有
      - "/var/run/docker.sock:/var/run/docker.sock"
      # コンテナ停止後もLocalStack内の最新データを"./localstack/save_data"に保持
      - "vol_localstack:/var/lib/localstack"

volumes:
  vol_localstack:
    driver: local
```

* コード解説

Dockerfileで定義した作業コンテナと、LocalStackをまとめて起動・連携できるようにします。

* depends_onで起動順序を制御
  * my_app側でdepends_onを使用しています。これにより、必ず設定したコンテナ(今回ならLocalStack) が先に起動してからアプリコンテナが起動するよう制御しています。
* データの永続化
  * LocalStackはデフォルトのままだと、コンテナを停止した時に作成したS3バケットやデータが削除されます。
  * 名前付きボリュームを割り当てることで、コンテナを再起動してもLocalStack内のデータを保持できるようにしています。

### devcontainer作成

* コード

```json: devcontainer.json
{
    "name": "my-project",
    "dockerComposeFile": [
        "../compose.yaml"
    ],
    "service": "my_app",
    "workspaceFolder": "/app",
    "features": {
        "ghcr.io/devcontainers/features/docker-outside-of-docker:1.10.0": {
        "version": "latest",
        "enableNonRootDocker": "true"
        }
    },
}

```

* コード解説
ここでLambaコンテナを起動している
この記事でいらない部分が多い

### コンテナ起動

コンテナはdevcontainerを使って開きます。
Visual Studio Codeの左下「><」アイコンをクリックして、「コンテナを再度開く」を選択してください。
コンテナが自動で起動します。

### 動作確認

コンテナ起動後、下記コマンドを入力。
サービス情報が返ってきたらlocalstackをうまくいっておりローカル環境が構築できている

```bash
curl http://localstack:4566/_localstack/health
```

成功例

![画像](/images/docker_local_aws/check_localstack.png)

## 最後に

今回の記事では、Dockerコンテナ上でLocalStackとDBを立ち上げAWSのローカル開発環境を作成しました
LocalStackを使うことでAWS使用時に必要な認識設定や権限を気にせずかつコストゼロでローカル開発を実施できます。

最後までお読みいただきありがとうございました。

## 参考

いらなそう