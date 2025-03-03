---
title: "alembicをfastAPIで用いる場合にloggingを実装する場合の注意点"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "fastAPI", "alembic", "logging"]
published: true
published_at: 2025-03-04 08:00 # 未来の日時を指定する
---

## 概要

FastAPIとuvicornを用いてAPIサーバーを立ち上げる場合、uvicorn上で設定を適切に行うことで、意図したログ出力を行うことができます。
しかし、マイグレーションツールであるalembicとの組み合わせた場合にハマってしまったのでそのメモです。  
結論から言うと、alembicが自動生成するenv.pyに含まれるログ設定が、uvicornのログ設定と競合してしまうので、何かしらの対策が必要です。  

## alembicとは？

alembicは、PythonのORMライブラリSQLAlchemyと連携して利用されるデータベースマイグレーションツールです。  
データベースのスキーマ（構造）をバージョン管理し、アプリケーションの変更に合わせたテーブル構造の更新を自動化するためのツールです。  
alembicを利用することで、開発環境と本番環境で同一のデータベース構造を保ちつつ、スムーズな移行が可能になります。  

私は以下を参照して、なんとなく理解しました。

https://alembic.sqlalchemy.org/en/latest/  
https://zenn.dev/shimakaze_soft/articles/4c0784d9a87751  
https://qiita.com/ktamido/items/8c3e56de1e889ec8a4cf

## alembicの`env.py`におけるlogging設定

alembicが自動生成する`env.py`には、以下のようにloggingの設定を行うコードが含まれています。

```python:env.py
import os
from logging.config import fileConfig

from sqlalchemy import engine_from_config, pool

from alembic import context

DATABASE_URL = os.getenv("DATABASE_URL", "fail_to_get_database_url")

config = context.config

# ↓ここでログ設定が行われる
if config.config_file_name is not None:
    fileConfig(config.config_file_name)


target_metadata = None

# ...以下略
```

このコードは、alembicが用いるロガーを初期化するためのものです。  
が、この記述により、uvicornで指定したloggingの設定が上書きされる可能性があります。  
この記事を書いている際に公式ドキュメントを見たのですが、ちゃんと言及されていますね。。。知りませんでした。

![](/images/0009_llyssm_logging_on_fastapi_and_alembic/001.png =600x)

## 問題の詳細

FastAPIの`lifespan`イベント内などでalembicを利用してマイグレーションを実行する場合、alembicの`env.py`内のlogging設定が呼び出されます。

```python:main.py
import os
from contextlib import asynccontextmanager

from fastapi import FastAPI

ALEMBIC_CONFIG_PATH = os.getenv("ALEMBIC_CONFIG_PATH")

def apply_migrations():
    alembic_cfg = Config(ALEMBIC_CONFIG_PATH)
    # ↓ここでalembicのenv.pyが呼び出される
    current_rev = command.current(alembic_cfg, verbose=True)
    head_rev = command.heads(alembic_cfg, verbose=True)

    if current_rev != head_rev:
        try:
            command.upgrade(alembic_cfg, "head")
        except Exception as e:
            raise e

@asynccontextmanager
async def lifespan(app: FastAPI):
    # https://fastapi.tiangolo.com/advanced/events/#lifespan
    # yieldより前のコードは、FastAPI が起動するときに実行される
    try:
        # alembicのマイグレーションを実行
        apply_migrations()
    yield
    # yieldより後のコードは、FastAPI が終了するときに実行される


app = FastAPI(lifespan=lifespan)

if __name__ == "__main__":
    import uvicorn

    uvicorn.run(
        "src.main:app",
        host="0.0.0.0",
        port=8000,
        reload=True,
        log_config=config,
    )
```

結果として、uvicornで渡していたログ設定が無効になってしまい、意図しないログ出力やレベルでの出力が発生する場合があります。  

### 具体的な影響例

- **ログフォーマットの不統一:** uvicornで指定したフォーマットが、alembic実行時に標準のフォーマットに戻ってしまう。
- **ログレベルの混乱:** 本来はDEBUGレベルで出力していたログが、alembicの設定によってINFOレベルやWARNINGレベルに変更され、デバッグ情報が見えなくなる。

## 解決策

単純に、alembicの`env.py`内にあるログ設定部分をコメントアウトするだけで良いです。以下のように修正します。

```python
# Interpret the config file for Python logging.
# このコードをコメントアウトすることで、uvicorn.runでの設定が維持される
# if config.config_file_name is not None:
#     fileConfig(config.config_file_name)
```

## 代替案

ほかには、lifespan関数内で、alembicのマイグレーション処理の後にログ設定を再度行う方法も考えられますが、試していません。

```python
import logging.config

@asynccontextmanager
async def lifespan(app: FastAPI):
    # https://fastapi.tiangolo.com/advanced/events/#lifespan
    # yieldより前のコードは、FastAPI が起動するときに実行される
    try:
        # alembicのマイグレーションを実行
        apply_migrations()
        # ログ設定を再度行う
         logging.config.dictConfig(config)
    yield
    # yieldより後のコードは、FastAPI が終了するときに実行される
```
