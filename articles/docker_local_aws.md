---
title: "初めてのDocker環境構築をして苦労した話"
emoji: "🐳"
type: "tech"
topics: ["docker", "aws", "localstack"]
published: true
published_at: 2026-08-24 06:00
publication_name: "secondselection"
---

## はじめに

### 背景

先日、業務で初めてDockerを用いたアプリ開発の環境構築を担当しました。
事前に類似PJの構成ファイルも入手していたため、「これと同じように作れば大丈夫そうだな」と思っていました。
しかし、いざ手を動かしてみると一筋縄ではいかず、かなり苦戦することになりました。

本記事では、環境構築をするにあたって私が苦労した体験談や調べて学んだことを共有します。

### 対象読者

* 初心者エンジニアの方
* これから業務で環境構築をする予定のある方

※ 本記事ではDockerの基本的な用語や仕組みについての解説は省略いたします。

## 事前準備

### 必要環境とツール

* Docker環境: WSL2上でDockerが動作する状態であること。

### システム構成

本記事で紹介する私が実施した環境構築のシステム構成図を記載します。
LocalStackを活用し、AWSサービスをローカル上で開発やテストを行える構成を目指しました。

![画像](/images/docker_local_aws/system_diagram.drawio.png)

※作業用コンテナ（my_app）は、実際にPythonなどのアプリケーションコードを配置して開発することを想定した環境です。
　本記事ではアプリコードの実装については割愛します。

### ディレクトリ構成

本記事で構築するディレクトリ構成は以下になります。

```text
myapp/
├── src/         # アプリ用ソースコードフォルダ(今回は内容割愛)
├── Dockerfile
└── compose.yaml
```

## 環境構築

それでは環境構築を実施します
今回はWSL上にLocalStackのコンテナ、作業用コンテナの2つを準備します。

### 作業用コンテナのDockerfile作成

```Dockerfile: Dockerfile
FROM public.ecr.aws/ubuntu/ubuntu:jammy

# 作業ディレクトリの指定
WORKDIR /app

# ubuntuの内部を日本時間に変更
RUN ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
    && apt update \
    && apt install software-properties-common -y \
    && apt update \
    && apt upgrade -y \
    && apt install -y --no-install-recommends \
        git \
        python3.14 -y \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

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

# bashで待機状態にする(コマンド操作できるようになる)
CMD ["bash"]
```

#### Dockerfileの補足

上記のDockerfileは単にツールを入れるだけでなく、「Gitの導入」と「権限エラーの回避」を実施する仕様にしています。

* Gitの導入
  * コンテナ内でコード変更やバージョン管理するためにインストールしています。
  * `git log`の文字化けを防ぐために`LESSCHARSET=utf-8`で文字コードを指定しています。
* 権限エラーの回避
  * コンテナ内をrootユーザーのまま操作すると、コンテナ内で作成・変更したファイルがroot所有になり、WSL側から編集できなくなります（権限エラーになる）。
  * 本環境ではコンテナ内でWSL側のユーザーと同じユーザーを作成しrootではなく自分がファイル操作・実行できるよう所有権を変更しています。

#### Dockerfileで苦労した点

苦労した点は、ビルドエラーが発生している箇所の特定でした。
最初は複数の処理を`&&`でつないだ1つの`RUN`命令として書いており、ビルドが途中で失敗した際に「どのコマンドで止まったのか」がひと目で判別できませんでした。
そこで、デバッグ時はコマンドごとに`RUN`を一時的に分解してビルドを実行し、エラーの発生箇所を特定しました。

```dockerfile: Dockerfile

# 変更前:1つのRUNにまとめていると、エラー箇所の特定が難しい
RUN ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
    && apt update \
    && apt install software-properties-common -y \
    && apt update \
    && apt upgrade -y \
    && apt install -y --no-install-recommends \
        git \
        python3.14 -y \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# 変更後:コマンドごとにRUNを分割し、エラー発生箇所を可視化する
RUN ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
RUN apt update
RUN apt install software-properties-common -y
RUN apt update
RUN apt upgrade -y
RUN apt install -y --no-install-recommends 
RUN apt install -y git
RUN apt install -y python3.14 -y
RUN rm -rf /var/lib/apt/lists/*
RUN apt-get clean

```

「最初から分割しておけば？」とも思いましたが、Dockerでは`RUN`の数だけイメージのレイヤーが作成され、最終的なイメージサイズが大きくなってしまうという仕様があります。
そのため、環境構築中は`RUN`を細かく分けてエラー特定やキャッシュを効かせやすくし、構成が確定した最終段階で1つの`RUN`に集約するのがおすすめです。

### compose.yaml作成

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

  # LocalStack用
  localstack:
    image: localstack/localstack:4.14
    ports:
      - "4566:4566" 
    environment:
      # 自分が使用するAWSサービスを指定
      SERVICES: s3,secretsmanager,ec2
      # デバッグログが出力される設定
      DEBUG: 1
      # docker.sockとLocalStackコンテナの中を共有する
      DOCKER_HOST: unix:///var/run/docker.sock
    volumes:
      # LocalStack内でコンテナを起動するためのdocker.sockの共有
      - "/var/run/docker.sock:/var/run/docker.sock"
      # コンテナ停止後もLocalStackのデータをvol_localstackに保持
      - "vol_localstack:/var/lib/localstack"

volumes:
  vol_localstack:
    driver: local
```

#### compose.yamlの補足

Dockerfileで定義した作業コンテナと、LocalStackをまとめて起動・連携できるようにします。

* `depends_on`で起動順序を制御
  * `my_app`はLocalStackを利用する前提で動くため、`depends_on`を指定して必ずLocalStackコンテナが先に起動するよう制御します。これにより、起動直後の接続エラーを防ぎます。
* ビルド引数（`args`）によるユーザー情報の設定
  * WSL側のユーザー情報をコンテナに引き継ぐための設定です。ご自身の環境のユーザー情報に合わせて変更します。
  * ユーザ情報は以下のターミナルコマンドで確認できます。

 ```bash
# USER_NAME確認
whoami
# USER_UIDの確認
id -u
# USER_GIDの確認
id -g
```

* データの永続化
  * LocalStackはデフォルトのままだと、コンテナを停止した時に作成したS3バケットやデータが削除されます。
  * 名前付きボリュームを割り当てることで、コンテナを再起動してもLocalStack内のデータを保持できるようにしています。

#### compose.yamlで苦労した点

参考にした既存プロジェクトの`compose.yaml`をそのまま流用したところ、LocalStackの起動時にエラーが発生しました。
原因を調べてみると、私が環境構築する1ヶ月前にLocalStack側で仕様変更が行われており、従来の設定ファイルの記述では動作しないことが判明しました。

この問題に対しては、以下の2つのアプローチが考えられます。

* 最新仕様に合わせて記述を更新する
  * 公式ドキュメントを参照し、最新バージョンで推奨されている環境変数や設定項目へ書き換えます。
* 動作実績のある旧バージョンに固定する（今回採用）
  * `localstack/localstack:4.1`のように旧バージョンのイメージタグを明示的に指定し、従来通りの設定で動かす方法です。

「既存PJ通りに書けば動く」と過信せず、依存ツールのアップデート情報や公式ドキュメントを確認する大切さを学びました。

## 最後に

初めての環境構築は既存のファイルを真似するだけではうまくいかず苦労しました。
ですが、エラーの原因を分解して調べたり最新の仕様変化に対応したりすることは、エンジニアとして今後も活きる貴重な経験になりました。
本記事が、これから環境構築に挑戦する方の参考になれば幸いです。
最後までお読みいただきありがとうございました！
