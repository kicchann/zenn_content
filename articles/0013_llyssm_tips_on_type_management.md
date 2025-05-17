---
title: "PydanticでUnion型の自動判別とexclude_unset=Trueでハマった"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "pydantic", "fastapi"]
published: true
published_at: 2025-05-18 09:00 # 未来の日時を指定する
---

## はじめに

FastAPIのPydanticベースのAPI開発で、`Union[...]` や `.model_dump(exclude_unset=True)` を使う場面がでてきたのですが、
どハマりしたので、**備忘録として**まとめておきます。  

---

## 問題1: Union型の自動判別は「順番依存」

Pydanticの`Union`型（例: `Union[TextItem, ImageItem]`）は、**データの内容から自動でどちらの型か判別**してくれます。  
しかし、**判別ロジックは「Unionで最初に定義された型」から順にマッチ**を試みるため、意図しない型でインスタンス化されることがあります。  

### 例

```python
from pydantic import BaseModel
from typing import Literal, Union

class TextItem(BaseModel):
    item_type: Literal["text"]
    description: str

class ImageItem(BaseModel):
    item_type: Literal["image"]
    url: Optional[str] = None
    description: str

class ResponseModel(BaseModel):
    item: Union[TextItem, ImageItem]
```

このとき、Pydanticは上から順に型マッチを試みるため、仮に`item_type="image"`のデータを渡しても、`TextItem`の構造にマッチすればそちらの型が選ばれます。

```python
data = {
    "item": {
        "item_type": "image",
        "description": "サンプル画像"
        # url は未指定（None）
    }
}

parsed = ResponseModel(**data)
print(type(parsed.item))  # <class '__main__.TextItem'> 
```

---

## 解決策1: discriminatorを使う

Pydantic v1.8以降では、**discriminator**（判別子）を使って  
「どの型か」を明示的に判別できます。

```python
from pydantic import Field

class ResponseModel(BaseModel):
    item: Union[TextItem, ImageItem] = Field(..., discriminator="item_type")
```

こうすることで、`item_type`の値に応じて正しい型が選ばれます。  
**Union型を使う場合は必ずdiscriminatorを指定しましょう。**

---

## 問題2: exclude_unset=Trueでフィールドが消える

Pydanticの`.model_dump(exclude_unset=True)`は、**「デフォルト値のまま変更されていないフィールド」を辞書化時に除外**します。
Updateメソッドを実装する際には便利なのですが、Union型と組み合わせ使う際には注意が必要です。
つまり、上記の例だと、

```python
text_item = TextItem(description="foo") # item_typeはデフォルト値
text_item.model_dump(exclude_unset=True)  # => {'description': 'foo'}  # item_typeが消える！
```

と、`item_type`がデフォルト値のままなので、discriminatorに使うフィールドも除外されてしまいます。

このとき、**Union/discriminatorで使うフィールド（`item_type`）も除外される**ことがあり、  
DB保存やAPIレスポンスで「型情報が消える」→「復元時に型判別できない」  
という問題が発生します。

---

## 解決策2-1: discriminatorフィールドは必ず明示的にセット

- discriminatorで使うフィールド（例: `item_type`）は**必ず明示的にセット**する
- `exclude_unset=True`を使う場合は、**型判別に必要なフィールドが除外されていないか注意**

## 解決策2-2: __init__で自動補完しておく

- `__init__`メソッドをオーバーライドして、`item_type`を自動でセットすれば、`exclude_unset=True`を回避できます。

```python
class TextItem(BaseModel):
    item_type: Literal["text"]
    description: str

    def __init__(self, **data):
        data["item_type"] = "text"  # item_typeを自動でセット
        super().__init__(**data)
```


---

## まとめ

- Union型＋discriminator指定で「型安全」な自動判別を実現
- `exclude_unset=True`使用時は、**型判別に必要なフィールドが除外されないよう注意**
- `discriminator`に使うフィールドは
  - 明示的にセットする
  - `__init__`メソッドをオーバーライドして自動でセットする

---

## 参考

- [Pydantic: Discriminated Unions](https://docs.pydantic.dev/latest/concepts/unions/#discriminated-unions)
- [Pydantic: exclude_unset](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_dump)

---
