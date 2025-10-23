---
title: "TYPE_CHECKINGとPydanticのmodel_rebuild()の使い方まとめ"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

**Pythonの`TYPE_CHECKING`とPydanticの`model_rebuild()`の役割・使い方**を、実務・zennアウトプット向けに分かりやすく解説します。

---

## 1. `TYPE_CHECKING`とは？

### 役割

- **型ヒント用のimport**だけを、実行時には無視して型チェック時だけ有効にするための仕組みです。
- Pythonの循環import（A→B→Aのようなimport）を回避するためによく使います。

### 使い方

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from src.schemas.user import User  # 型ヒント用。実行時にはimportされない
```

- こうすると、型チェッカー（mypyやPyrightなど）は`User`型を認識しますが、**実行時にはimportされません**。

### どんな時に使う？

- ファイルAとファイルBが**お互いをimportし合う**（循環参照）場合
- 型ヒントだけで十分な場合（実際の値や関数は使わない）

---

## 2. `model_rebuild()`とは？（Pydantic v2系）

### 役割

- **循環参照を含む型ヒント（ForwardRef）**をPydanticが正しく解決できるようにするためのメソッドです。
- Pydantic v2では、循環参照を含むスキーマを使う場合、**明示的に`model_rebuild()`を呼ぶ必要があります**。

### 使い方

```python
from src.schemas.user import User  # 実行時importが必要
UserSettingResponse.model_rebuild()
```

- `UserSettingResponse`の中で`user: \"User\"`のような**文字列型ヒント**（ForwardRef）を使っている場合、`model_rebuild()`を呼ぶことでPydanticが`User`型を解決できるようになります。

### どんな時に使う？

- Pydanticのスキーマで**循環参照**（例：User→UserSetting→Userのような相互参照）がある場合
- 型ヒントに**文字列（\"User\"など）**を使っている場合

---

## 3. よくあるパターン

### 例：UserとUserSettingが相互参照する場合

```python
# user_setting.py
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from .user import User

class UserSettingResponse(BaseModel):
    user: "User"  # 文字列型ヒント
    # ...他のフィールド...

# ファイル末尾で
from .user import User  # 実行時import
UserSettingResponse.model_rebuild()
```

---

## 4. 注意点

- `TYPE_CHECKING`は**型チェック用**、`model_rebuild()`は**実行時の型解決用**です。
- `model_rebuild()`の前に**必ず実体import**（`from ... import User`など）が必要です。
- テストだけで`model_rebuild()`を呼んでも、本番で使うなら**本番コード側で必ず呼ぶ**必要があります。

---

## まとめ

- **循環参照の型ヒントは`TYPE_CHECKING`でimport、型ヒントは文字列で書く**
- **Pydantic v2では`model_rebuild()`を必ず呼ぶ（その前に実体importも忘れずに）**
- **本番コードで`model_rebuild()`を呼ぶことで、テスト・本番どちらでも型解決エラーが出なくなる**

---

zenn記事のアウトラインやサンプルもご希望あればお手伝いします！