---
title: "PydanticでUpdate用モデルを動的生成する：バリデーション継承＋Optional対応まで"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python, pydantic, fastapi, TypeScript]
published: true
published_at: 2025-04-11 08:00 # 未来の日時を指定する
---

## はじめに

FastAPI + Pydantic でAPIを構築する中で、「部分更新」を実装するためには、すべてのフィールドがOptionalな `UpdateModel` を書く必要があります。しかしBaseModelと同じフィールドとvalidatorを持つ `UpdateModel` を毎回手で書くのは面倒ですし、バリデーションを忘れたり、間違った型を指定してしまうこともあります。

そこで、以下を満たす `UpdateModel` を自動生成する関数を作成しました：

- `Optional` 変換：元のフィールドをすべて `Optional` に変換（部分更新対応）
- バリデーション継承：元モデルの `@field_validator` をそのまま継承
- FastAPIのOpenAPIドキュメントにも正しく出力される
- SQLAlchemyなど属性ベースのオブジェクトから `.model_validate()` できる

---

## 背景

例えば次のような `BaseModel` があったとします：

```python
from pydantic import BaseModel, field_validator

class UserBase(BaseModel):
    name: str
    age: int

    @field_validator("name")
    def name_must_not_be_empty(cls, v):
        if not v.strip():
            raise ValueError("name must not be empty")
        return v
```

このままでは PUT /users/{user_id} のリクエストボディとして受け取ることはできません。
そこで、次のような UpdateModel を作る必要があります：

```python
class UserUpdate(BaseModel):
    name: Optional[str] = None
    age: Optional[int] = None
```

このような UpdateModel を毎回手で書くのは面倒ですし、バリデーションを忘れたり、間違った型を指定してしまうこともあります。
そこで、`UserBase` から自動生成する関数を作成します。
この関数は、元のモデルのフィールドをすべて `Optional` に変換し、バリデーションを継承します。
さらに、FastAPIのOpenAPIドキュメントにも正しく出力されるようにします。

## 実装

以下のような関数を作成します：

```python
from typing import Optional, get_origin, Union
from pydantic import BaseModel, create_model, ConfigDict

def make_update_model(model_cls: type[BaseModel]) -> type[BaseModel]:
    fields = {}
    for field, annotation in model_cls.__annotations__.items():
        # Optionalでない場合はOptionalにする
        if get_origin(annotation) is not Union:
            fields[field] = (Optional[annotation], None)
        else:
            fields[field] = (annotation, None)

    return create_model(
        f"{model_cls.__name__}Update",
        __base__=model_cls,  # 元モデルのバリデーションを継承
        __module__=model_cls.__module__,  # スキーマ生成で必要
        model_config=ConfigDict(from_attributes=True),  # 属性ベースのオブジェクトからmodel_validate可能に
        **fields
    )
```

この関数は、元のモデルのフィールドをすべて `Optional` に変換し、バリデーションを継承します。

## 使い方(API側)

以下のように使います：

```python
class UserBase(BaseModel):
    name: str
    age: int

UserUpdate = make_update_model(UserBase)

# FastAPIでそのまま使える
@app.put("/users/{user_id}")
def update_user(user_id: int, data: UserUpdate):
```

これで、次のようなリクエストボディが受け取れます：

```json
{
    "name": "Updated Name"
}
```

バリデーションも継承されているので、空文字はエラーになります。

```python
def test_update_model_validation():
    with pytest.raises(ValidationError):
        UserUpdate.model_validate({"name": "    "})  # name must not be empty → バリデーションエラー
```

## 使い方(フロントエンド側)

Pythonで UserBase をもとに UserUpdate を生成するのと同様に、TypeScript 側でも元の `User` 型に対して `Partial<User>` を使うことで、すべてのフィールドがOptionalな更新用の型を定義できます。

```typescript
// types.ts
export type User = {
  name: string;
  age: number;
};

// 部分更新用の型
export type UserUpdate = Partial<User>;
```

PUT リクエストでの実装例はこんな感じになります。

```typescript
// api.ts
import axios from "axios";
import type { UserUpdate } from "./types";

export async function updateUser(userId: number, data: UserUpdate) {
  const response = await axios.put(`/users/${userId}`, data);
  return response.data;
}
```

これで、Python側・TypeScript側ともに「部分更新用のOptionalモデル」を共通の設計思想で使うことができます。

## 参考

https://docs.pydantic.dev/latest/concepts/models/#dynamic-model-creation

https://fastapi.tiangolo.com/