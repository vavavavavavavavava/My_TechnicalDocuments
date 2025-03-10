# PydanticモデルとSQLAlchemyの統合ガイド

このドキュメントでは、MySQLのJSON型列を扱う際に、Python側でPydanticモデルを直接利用できるようにする2種類のSQLAlchemyカスタム型をご紹介します。  
- **PydanticJSON**: 固定された1種類のPydanticモデルに対応する場合に利用します。  
- **PydanticUnionJSON**: JSON列内に複数のデータ構造（複数のPydanticモデルのどれか）が入る可能性がある場合、Unionのような用途で利用します。

---

## 1. PydanticJSON

### 概要
PydanticJSONは、1種類のPydanticモデルと連携するための汎用SQLAlchemy TypeDecoratorです。  
- **目的**:  
  - データベースへ書き込む際、Pydanticモデルのインスタンスを辞書（JSON）に変換する。  
  - 読み出す際、JSONデータを自動的に指定したPydanticモデルのインスタンスに再構築する。

### 用途
- **固定データ構造**: JSON列の構造が常に一定（例：設定情報、ユーザープロファイルなど）で、必ず同じPydanticモデルで表現できる場合に適しています。  
- **型安全性の向上**: Pydanticのバリデーション機能を活用し、データ整合性を担保できます。

### 利点
- 1箇所で変換ロジックを実装するため、重複した型定義の手間を削減。  
- 読み書き時に自動で変換を行うため、開発者はPydanticモデルのインスタンスとしてデータを扱える。

### 利用例

```python
from pydantic import BaseModel
from sqlalchemy.types import TypeDecorator, JSON
from sqlalchemy import Column, Integer, create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

# Pydanticモデルの定義（1種類）
class MyDataModel(BaseModel):
    key1: str
    key2: int

# 汎用のPydantic対応JSON型（1モデル版）
class PydanticJSON(TypeDecorator):
    impl = JSON

    def __init__(self, model: type[BaseModel], *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.model = model

    def process_bind_param(self, value, dialect):
        if value is None:
            return None
        if isinstance(value, self.model):
            return value.dict()
        if isinstance(value, dict):
            return self.model(**value).dict()
        raise ValueError("値はPydanticモデルまたはdictである必要があります")

    def process_result_value(self, value, dialect):
        if value is None:
            return None
        return self.model(**value)

# SQLAlchemyモデルで利用
class MyTable(Base):
    __tablename__ = 'my_table'
    id = Column(Integer, primary_key=True)
    data = Column(PydanticJSON(MyDataModel))

# セッション設定例
engine = create_engine('mysql+pymysql://user:password@localhost/mydatabase')
Session = sessionmaker(bind=engine)
session = Session()

# 新規レコード作成例
new_data = MyDataModel(key1='example', key2=123)
new_record = MyTable(data=new_data)
session.add(new_record)
session.commit()

# 読み出し例：自動でMyDataModelのインスタンスに変換
record = session.query(MyTable).first()
print(record.data.key1)  # => 'example'
```

---

## 2. PydanticUnionJSON

### 概要
PydanticUnionJSONは、複数のPydanticモデルのどれかに適合する可能性があるJSONデータを扱うためのカスタム型です。  
- **目的**:  
  - 複数のデータ構造（例えば、MyDataModelとMyDataModel2など）を1つのJSON列で保持する際、書き込み時はどのモデルにも合致する辞書に変換し、読み出し時は順番に各モデルで検証して最初に適合したモデルのインスタンスを返します。

### 用途
- **柔軟なデータ構造**: 1つのJSON列に、異なる形式のデータが混在する場合に有効です。  
- **多様な入力データの対応**: APIのレスポンスや多様なイベントログなど、データ構造が状況に応じて変化する場合に利用できます。

### 利点
- 複数の型を1つのカスタム型でまとめることで、コードの重複や管理コストを低減。  
- 各モデルに対するバリデーションを順次行い、適合するものを返すため、柔軟かつ堅牢なデータ変換が可能。

### 注意点
- **順序依存**: 読み出し時は指定したモデルの順番に従って検証するため、順序が変換結果に影響する可能性があります。  
- **曖昧なケースへの対処**: 複数のモデルが部分的に共通する場合、意図しない型が選ばれる可能性があるため、入力データの仕様に応じた調整が必要です。

### 利用例

```python
from pydantic import BaseModel
from sqlalchemy.types import TypeDecorator, JSON
from sqlalchemy import Column, Integer, create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from typing import Tuple, Any

Base = declarative_base()

# 複数のPydanticモデルの定義
class MyDataModel(BaseModel):
    key1: str
    key2: int

class MyDataModel2(BaseModel):
    key3: str
    key4: float

# 汎用のUnion対応Pydantic JSON型
class PydanticUnionJSON(TypeDecorator):
    impl = JSON

    def __init__(self, models: Tuple[type[BaseModel], ...], *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.models = models

    def process_bind_param(self, value: Any, dialect):
        if value is None:
            return None
        for model in self.models:
            if isinstance(value, model):
                return value.dict()
        if isinstance(value, dict):
            for model in self.models:
                try:
                    validated = model(**value)
                    return validated.dict()
                except Exception:
                    continue
        raise ValueError("値は対象のPydanticモデルのインスタンスか、適合するdictでなければなりません。")

    def process_result_value(self, value: Any, dialect):
        if value is None:
            return None
        for model in self.models:
            try:
                return model(**value)
            except Exception:
                continue
        raise ValueError("取得した値がどのPydanticモデルにも適合しません。")

# SQLAlchemyモデルでUnion型を利用
class MyTable(Base):
    __tablename__ = 'my_table'
    id = Column(Integer, primary_key=True)
    # MyDataModelまたはMyDataModel2のいずれかに対応
    data = Column(PydanticUnionJSON((MyDataModel, MyDataModel2)))

# セッション設定例
engine = create_engine('mysql+pymysql://user:password@localhost/mydatabase')
Session = sessionmaker(bind=engine)
session = Session()

# MyDataModelのインスタンスでレコードを追加
record1 = MyTable(data=MyDataModel(key1='example', key2=123))
session.add(record1)
session.commit()

# MyDataModel2のインスタンスでレコードを追加
record2 = MyTable(data=MyDataModel2(key3='another', key4=3.14))
session.add(record2)
session.commit()

# 読み出し例
r1 = session.query(MyTable).filter_by(id=record1.id).first()
print(r1.data)  # => MyDataModel(key1='example', key2=123)

r2 = session.query(MyTable).filter_by(id=record2.id).first()
print(r2.data)  # => MyDataModel2(key3='another', key4=3.14)
```

---

## 3. 両者の用途・使い分け

| 項目 | PydanticJSON | PydanticUnionJSON |
| --- | --- | --- |
| **対象データ** | 固定の1種類のデータ構造 | 複数の可能性があるデータ構造（Union） |
| **書き込み時の変換** | 指定モデルのインスタンスを辞書に変換 | 各モデルで検証し、該当するモデルの辞書に変換 |
| **読み出し時の処理** | 常に指定した1モデルで変換 | 複数モデルを順に検証して最初に合致したモデルのインスタンスを返す |
| **利用シーン** | 一定の仕様に沿った設定情報、プロファイル情報等 | APIのレスポンス、イベントログ、複数の形式が混在するデータ |

### まとめ
- **PydanticJSON**は、JSON列のデータ構造が常に一定の場合にシンプルかつ型安全に扱うための手法です。  
- **PydanticUnionJSON**は、複数のデータ構造を1つの列で扱う場合に、柔軟かつ統一的にデータ変換ロジックを実装できる方法です。  
どちらも、SQLAlchemyとPydanticの機能を組み合わせることで、データベースとPythonコード間の変換を自動化し、開発効率とデータの整合性を向上させます。

このように用途に応じて適切なカスタム型を選択することで、MySQLのJSON列をより効果的に活用できるようになります。