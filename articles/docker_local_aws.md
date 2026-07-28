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

AWSの本番クラウド上で開発をする際一部サービスはお金がかかるし何かと不便です
LocalStackを使うとクラウド環境を使用しなくても、ローカル環境上で開発及びテストができます。
今回はこの環境構築を解説します
dockerを使うことで他のユーザーも簡単に環境構築できるので便利です。

### 対象読者

* AWSをローカル環境で試したい人
* LocalStackのdocker開発に苦戦している人

※　本記事ではdockerに関する基本的な用語や仕組みについての解説は省略いたします。

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
├── db/                       # DB初期化
│   └── create_table.sql
├── src/                      # アプリケーションコード
├── Dockerfile                # lambdaサービス構築用のDockerfile
├── compose.yaml              # 先ほどのComposeファイル
└── pyproject.toml            # Pythonパッケージ依存関係・プロジェクト定義
```

## AWSのローカル開発

### LocalStackとは

LocalStackは、AWSのクラウドサービスをローカル環境で擬似的に再現できるツールです。
本番のAWS環境に依存せず、S3やLambdaDynamoDBなどの挙動をローカルで再現できるため、テストの高速化やクラウド利用コストの削減といったメリットがあります。
対応しているサービスも多く、S3、DynamoDB、Lambda、SNS、SQSなど、幅広いAWS機能を模擬的に利用できます。

## 環境構築手順

それでは環境構築を実施します
今回はWSL上にDBのコンテナ、localstackのコンテナ、開発・実行用コンテナの3つを準備します。

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
RUN add-apt-repository ppa:deadsnakes/ppa -y
RUN apt update
RUN apt upgrade -y
RUN apt install python3.14 -y
RUN apt install curl -y
RUN apt install git -y
RUN apt install unzip -y

# AWS CLI v2 のインストール
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip awscliv2.zip \
    && ./aws/install \
    && rm -rf awscliv2.zip aws

RUN rm -rf /var/lib/apt/lists/*
RUN apt-get clean

# public.ecr.aws/ubuntu/ubuntu:jammy だと、git log が文字化けするので less コマンドの charset を指定する
ENV LESSCHARSET=utf-8

RUN curl -sS https://bootstrap.pypa.io/get-pip.py | python3.14

# pip3でpyproject.toml記載のパッケージをインストール
COPY pyproject.toml ./
RUN pip3 install --no-cache-dir --ignore-installed .[dev]

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

アップロード手順
ローカルスタックが最新版はユーザー登録が必要なので、必要ない古いやつをインストールしていること
最後ディレクトリ所有者をルートから自分に変更していること

### pyproject.toml作成

* コード

```toml: pyproject.toml
[project]
name = "ecovpp"
version = "0.1.0"
description = ""
readme = "README.md"
authors = [
    { name = "Your Name", email = "your.email@example.com" },
]
requires-python = ">=3.14"


dependencies = [
    "aws-lambda-powertools==3.29.0",
    "boto3==1.43.14",
    "pymysql==1.2.0",
    "tenacity==9.1.4"
]

[project.optional-dependencies]
dev = [
    "coverage==7.14.1",
    "pytest==9.0.3",
    "pytest-mock==3.14.0",
    "pytest-cov==6.0.0",
    "pytest-freezer==0.4.9",
    "requests-mock==1.10.0",
    "moto==5.2.1",
    "flake8==7.3.0",
    "mypy==2.1.0",
    "bandit==1.9.4",
    "ruff==0.15.0",
    "types-PyMySQL==1.1.0.20260518",
    "types-PyYaml==6.0.12.20260518",
    "types-requests==2.32.4.20250913",
    "python-taint==0.42",
    "mypy-boto3-dynamodb==1.43.0",
    "boto3-stubs[s3,secretsmanager]==1.43.36"
]

[tool.pytest.ini_options]
# pytestでパスを省略した場合に使用されるパス
testpaths = ["src/tests"]
# pytestでimportを探すパス
pythonpath = ["src/app"]

[tool.mypy]
# コンテナ内のPythonの実行パスを指定して、インストール済みライブラリを認識させる
python_executable = "/usr/bin/python3.14"

# チェック対象のパスを設定
mypy_path = "src/app"

# __init__.py がないフォルダ（srcの階層など）もmypyに認識させる
explicit_package_bases = true
```

* コード解説

開発用と本番用でインストールするパッケージを変えていること
この記事に関する不要なコードが多い

### compose.yaml作成

* コード

```yaml: compose.yaml
services:
  lambda:
    build:
      context: .
      dockerfile: ./Dockerfile
      args:
        USER_NAME: ${USER_NAME}
        USER_UID: ${USER_UID}
        USER_GID: ${USER_GID}
    volumes:
      - .:/app    
    # コンテナ上でコマンド操作ができる
    tty: true
    # コンテナ上でキーボード操作ができる
    stdin_open: true
    depends_on:
      - ecomeganedb5_7
      - localstack

  # エコめがねDB(MySQL5.7)用
  # エコめがねDBのinitDBはMySQLのバージョンにかかわらず共通
  ecomeganedb5_7:
    image: public.ecr.aws/docker/library/mysql:5.7
    environment:
      # 初期化の時にrootパスが必要
      MYSQL_ROOT_PASSWORD: "1q2w3e4r"
      TZ: "Asia/Tokyo"
    volumes:
      - vol_ecomeganedb5_7:/var/lib/mysql
      - ./db/cnf/my5_7.cnf:/etc/mysql/conf.d/my.cnf
      - ./db/ecomeganedb5_7:/docker-entrypoint-initdb.d
    ports:
      - 3306:3306

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
  vol_ecomeganedb5_7:
    driver: local
  vol_localstack:
    driver: local
```

* コード解説
ボリュームでデータの永続化を行っている
depends_onでコンテナを起動する順番を安全に処理するよう指示している



### devcontainer作成

* コード

```json: devcontainer.json
{
    "name": "ECOVPP-AI=VPP",
    "dockerComposeFile": [
        "../compose.yaml"
    ],
    "service": "lambda",
    "workspaceFolder": "/app",
    "features": {
        "ghcr.io/devcontainers/features/docker-outside-of-docker:1.10.0": {
        "version": "latest",
        "enableNonRootDocker": "true"
        }
    },
    "customizations": {
        "vscode": {
            "extensions": [
                "ms-python.python",
                "ms-python.flake8",
                "ms-python.mypy-type-checker",
                "charliermarsh.ruff",
                "streetsidesoftware.code-spell-checker",
                "njpwerner.autodocstring",
                "usernamehw.errorlens",
                "eamodio.gitlens"
            ],
            "settings": {
                "[python]": {
                    "editor.formatOnSave": true,
                    "editor.defaultFormatter": "charliermarsh.ruff"
                },
                "editor.codeActionsOnSave": {
                    "source.fixAll": "explicit",
                    "source.organizeImports": "explicit"
                },
                "ruff.lint.enable": false,
                "cSpell.userWords": [
                    "autouse",
                    "boto",
                    "caplog",
                    "charliermarsh",
                    "chikuden",
                    "chikudendb",
                    "cursorclass",
                    "eamodio",
                    "ecocute",
                    "ecomegane",
                    "ecomeganedb",
                    "ecovpp",
                    "ecovppdb",
                    "errorlens",
                    "executemany",
                    "hepco",
                    "infile",
                    "jiseki",
                    "jusin",
                    "kakuho",
                    "kepco",
                    "keyaku",
                    "kyodo",
                    "lesscharset",
                    "localstack",
                    "mainvpp",
                    "moto",
                    "mypy",
                    "njpwerner",
                    "pymysql",
                    "pyproject",
                    "pytest",
                    "pythonpath",
                    "saigai",
                    "snks",
                    "sumame",
                    "teden",
                    "tepco",
                    "testpaths",
                    "usernamehw",
                    "vpp",
                    "vppdb",
                    "yoryo",
                    "yubin",
                    "zipcodemaster"
                ],
                "python.analysis.extraPaths": [
                    "./src/app"
                ]
            }
        }
    },
    "postCreateCommand": "/bin/sh .devcontainer/postCreateCommand.sh"
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
LocalStackを使うことでAWS使用時に必要な認識設定や権限を気にせずかつコストゼロでローカル開発を実施することができます。

最後までお読みいただきありがとうございました。

## 参考

いらなそう