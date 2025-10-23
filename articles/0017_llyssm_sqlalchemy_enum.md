---
title: "SQLAlchemy Enum で文字列値が期待どおりにならずに苦労した話"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "sqlalchemy", "database"]
published: true
---

## 概要

- `Enum(Color)` のように Python `Enum` をそのまま SQLAlchemy の `Enum` 型へ渡すと、既定では **メンバー名 (`EnumMember.name`) が列挙体値として採用**されます。
- 開発中の Web アプリでは `AccessType` を小文字値 (`value`) に統一していましたが、ある列の列挙型が大文字値のまま生成・解釈されていたため、DB が返す小文字値 (`public`) と整合が取れず `LookupError` が発生しました。

## 症状

```text
LookupError: 'public' is not among the defined enum values. Enum name: accesstype. Possible values: PRIVATE, PUBLIC, GROUP, LINK
```

## 原因分析

1. [SQLAlchemyのドキュメント](https://docs.sqlalchemy.org/en/20/core/type_basics.html#sqlalchemy.types.Enum)にもある通り、`Enum(SomePythonEnum)` をそのまま使うと **列挙値はメンバー名**で展開されます。
2. `Color` のような Enum を小文字値で定義したとします。

    ```python
    class Color(str, Enum):
        RED = "red"
        BLUE = "blue"
        GREEN = "green"
    ```

3. ところが `Column(Enum(Color))` と宣言すると、データベース側には `ENUM('RED','BLUE','GREEN')` が作成されます。そのため、クエリでフィルタをかけるときに `RED` が期待される場面で `red` が返ってきて例外が発生します。

    ```python
    from sqlalchemy import Enum

    # この場合、DB側では ENUM('RED','BLUE','GREEN') が作成されます
    color = Column(Enum(Color, default=Color.RED))

    ```

## 解決策

`Enum` 型生成時に `values_callable` を用いて、Python Enum の `.value` を列挙値として利用します。

```python
from sqlalchemy import Enum

# この場合、DB側では ENUM('red','blue','green') が作成されます
color = Column(
    Enum(
        Color,
        values_callable=lambda enum_cls: [member.value for member in enum_cls],
        name="colortype",
    ),
    default=Color.RED.value,
)
```

- これにより `Enum` が `["red", "blue", ...]` を列挙値として保持するようになり、DB との整合が取れるようになります。

## 代替案：Enum の命名スタイルを揃える

- **名前と値の表記を完全に一致させる**（例: `class Color(Enum): red = "red"`）なら、`Enum(Color)` だけで DB との齟齬は生まれません。

    ```python
    # 名前と値の表記を完全に一致させる
    class Color(str, Enum):
        red = "red"
        blue = "blue"
        green = "green"
    ```

- そもそも[Pythonのドキュメント](https://docs.python.org/ja/3.13/library/enum.html)では Enumのメンバー名を大文字で書いているのに、SQLAlchemy のドキュメントでは小文字になっており、これが混乱の原因です。実装がどちらのスタイルに寄っているか確認し、プロジェクト内で統一するのが良いと思います。

## 参考

- SQLAlchemy 公式ドキュメント: [sqlalchemy.types.Enum](https://docs.sqlalchemy.org/en/20/core/type_basics.html#sqlalchemy.types.Enum)
- Python 公式ドキュメント: [enum — 列挙型](https://docs.python.org/ja/3.13/library/enum.html)