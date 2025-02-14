---
title: "コンテナ内でuvicorn経由でfastAPIを動かす場合のメモ"
emoji: "🐋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "python", "fastAPI", "uvicorn"]
published: true
published_at: 2025-02-14 12:00 # 未来の日時を指定する
---

## 概要

docker内でuvicorn経由でfastAPIを動かす場合のメモです。  
loggingの設定や、環境に応じた切り替えなどを考慮して、uvicornの起動をmain.pyから行うようにしました。

## 構成

構成は以下の通りです。  

```
app
 ┣ src
 ┃ ┣ main.py
 ┣ Dockerfile
 ┣ docker-compose.yaml
 ┣ log_config.yaml
 ┗ requirements.txt
```

`Dockefile`は以下の通りです。  
`main.py`からuvicornを起動するようにしているので、単純なコマンドになっています。

```Dockerfile
# ベースイメージとしてPython 3.9を使用
FROM python:3.9-slim

# コンテナ内での作業ディレクトリを設定
WORKDIR /app

# 必要なパッケージをインストール
# ここで ping を含む net-tools をインストール
RUN apt-get update && apt-get install -y iputils-ping curl && apt-get clean

# 必要なPythonパッケージをインストール
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# アプリケーションコードをコンテナにコピー
COPY . /app

# コンテナ内でFastAPIアプリを起動
CMD ["python", "src/main.py"]
```

次は`docker-compose.yaml`です。  

```yaml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: my-app
    container_name: my-app
    ports:
      - "8080:8080"
    volumes:
      - .:/app
    command: ["python", "src/main.py"]
```

`main.py`の例です。  
uvicornの起動をmain.py内で行うことで、ログ設定や環境変数に応じた設定を行うことができます。

```python
import yaml
from fastapi import FastAPI

# 環境変数の設定
# dotenvを使ってもいい
# ENV = "development"
# DATABASE_URL = "hogehoge"

# FastAPIのインスタンスを作成
# 必要に応じて、ミドルウェアの設定やルータの追加を行う
app = FastAPI()

# ミドルウェアの追加
# app.add_exception_handler()
# ...

# ルータの追加
# app.include_router()

# ルートパスの例
@app.get("/")
def hello_world():
    return {"message": "Hello World"}

#　ログ設定の読み込み
with open(r"log_config.yaml") as f:
    log_config = yaml.safe_load(f)


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8080,
        reload=True, # ENVによって変更も可能
        log_config=log_config,
    )
```

## 参照したサイト

とても助かりました。ありがとうございます。

https://zenn.dev/techflagcorp/articles/8d6327311e1e9f  
https://zenn.dev/satonopan/articles/c4e6d55a64da0c  
https://nikkie-ftnext.hatenablog.com/entry/uvicorn-practice-default-logging-equivalent-yaml  
