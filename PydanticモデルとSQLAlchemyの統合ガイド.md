# PydanticモデルとSQLAlchemyの統合ガイド

このドキュメントでは、MySQLのJSON型列を扱う際に、Python側でPydanticモデルを直接利用できるようにする3種類のSQLAlchemyカスタム型をご紹介します。  
- **PydanticJSON**: 固定された1種類のPydanticモデルに対応する場合に利用します。  
- **PydanticUnionJSON**: JSON列内に複数のデータ構造（複数のPydanticモデルのどれか）が入る可能性がある場合、Unionのような用途で利用します。
- **PydanticDictJSON**: Dict型の型情報を利用して、辞書型のキーと値の型を検証する場合に利用します。

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

## 3. PydanticDictJSON

### 概要
PydanticDictJSONは、辞書型（Dict）のキーと値の型情報を活用して、JSON列に保存される辞書データを型安全に扱うためのカスタム型です。
- **目的**:
  - 辞書型のキーと値の型を明示的に宣言・検証する
  - 複雑な辞書型（ネスト構造、リストを含む辞書など）も型安全に扱う

### 用途
- **キー・値の型がパターン化された辞書**: 例えば、SensorTypeId(str) -> ValueTypeId(int) のような明確な型パターンがある辞書データ
- **センサーデータ**: 文字列キーと数値データなど、型が決まっている場合
- **設定データ**: 異なる型の値を持つが、構造が明確な辞書型データ

### 利点
- 辞書のキーと値の型を明示的に宣言することで、型安全性を高める
- Pydanticの型検証機能を活用して、データの整合性を自動的に担保
- ネストした辞書構造やリストを含む複雑な型にも対応可能

### 利用例

```python
from typing import Dict, Type, Any, List, Union, get_origin, get_args, Optional
from pydantic import BaseModel, create_model, validator
from sqlalchemy.types import TypeDecorator, JSON
from sqlalchemy import Column, Integer, String, create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class PydanticDictJSON(TypeDecorator):
    """
    Dict型の型情報を直接受け取るSQLAlchemy用のTypeDecorator
    """
    impl = JSON

    def __init__(self, dict_type: Type, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        # 型情報をチェック
        origin = get_origin(dict_type)
        if origin is not dict:
            raise ValueError(f"Dict型を指定してください。与えられた型: {dict_type}")
        
        # 型引数を抽出
        type_args = get_args(dict_type)
        if len(type_args) != 2:
            raise ValueError("Dictにはキーと値の型が必要です")
            
        self.key_type, self.value_type = type_args
        
        # 検証用のPydanticモデルを作成
        self._create_validator_model()

    def _create_validator_model(self):
        """検証用のPydanticモデルを動的に作成"""
        key_name = self._get_type_name(self.key_type)
        value_name = self._get_type_name(self.value_type)
        model_name = f'Dict{key_name}To{value_name}Model'
        
        # 検証用のモデルを作成
        self.validator_model = create_model(
            model_name,
            data=(Dict[self.key_type, self.value_type], ...),
            __validators__={
                'validate_data': validator('data', pre=True)(self._validate_dict)
            }
        )

    def _get_type_name(self, typ):
        """型からモデル名を生成する（ネストした型にも対応）"""
        # 型名取得のロジック（省略）...
        return str(typ)  # 簡易版

    @staticmethod
    def _validate_dict(cls, v):
        """辞書の検証関数"""
        if not isinstance(v, dict):
            raise ValueError("辞書型である必要があります")
        return v

    def process_bind_param(self, value, dialect):
        """SQLAlchemyからDBに書き込む際の変換処理"""
        if value is None:
            return None
            
        # 辞書の場合は検証してJSON化
        if isinstance(value, dict):
            try:
                # Pydanticモデルを使用して検証
                validated = self.validator_model(data=value)
                return validated.data
            except Exception as e:
                raise ValueError(f"辞書の検証に失敗しました: {str(e)}")
            
        raise ValueError("値は辞書である必要があります")

    def process_result_value(self, value, dialect):
        """DBから読み込んだ値をPythonオブジェクトに変換する処理"""
        if value is None:
            return None
        return value  # 単純に辞書として返す

# 使用例

# 1. 型定義
SensorDataType = Dict[str, int]
NestedDataType = Dict[str, Dict[str, float]]

# 2. モデル定義
class SensorData(Base):
    __tablename__ = 'sensor_data'
    
    id = Column(Integer, primary_key=True)
    # 型を指定したカラム
    data = Column(PydanticDictJSON(SensorDataType))
    
class WeatherData(Base):
    __tablename__ = 'weather_data'
    
    id = Column(Integer, primary_key=True)
    # ネストした辞書型を指定
    city_temps = Column(PydanticDictJSON(NestedDataType))

# 使用例
def example_usage():
    # センサーデータの作成例
    sensor = SensorData(data={
        "temperature": 24,
        "humidity": 65,
        "pressure": 1013
    })
    
    # ネストした辞書の例
    weather = WeatherData(city_temps={
        "tokyo": {"day": 28.5, "night": 21.3},
        "osaka": {"day": 30.1, "night": 22.8}
    })
    
    return sensor, weather

# セッション設定例
engine = create_engine('mysql+pymysql://user:password@localhost/mydatabase')
Session = sessionmaker(bind=engine)
session = Session()
```

---

## 4. 各カスタム型の比較と使い分け

| 項目 | PydanticJSON | PydanticUnionJSON | PydanticDictJSON |
| --- | --- | --- | --- |
| **対象データ** | 固定の1種類のデータ構造 | 複数の可能性があるデータ構造（Union） | 辞書型データ（キー・値の型が明確） |
| **書き込み時の変換** | 指定モデルのインスタンスを辞書に変換 | 各モデルで検証し、該当するモデルの辞書に変換 | 辞書のキーと値の型を検証して保存 |
| **読み出し時の処理** | 常に指定した1モデルで変換 | 複数モデルを順に検証して最初に合致したモデルに変換 | 辞書データとして取得 |
| **利用シーン** | 固定構造のデータ（設定、プロファイル等） | APIレスポンス、複数形式が混在するデータ | キー値の型が明確な辞書（センサーデータ、マッピング等） |
| **主な利点** | シンプルな構造化データの型安全性 | 柔軟なデータ構造の統一的な処理 | 特に辞書型データの型検証が容易 |
| **受け取るパラメータ** | 単一のPydanticモデルクラス | 複数のPydanticモデルクラスのタプル | Dict[キー型, 値型] の型情報 |

### まとめ
- **PydanticJSON**は、固定構造のデータを1つのPydanticモデルに基づいて扱う場合に適しています。  
- **PydanticUnionJSON**は、異なる複数のデータ構造を1つの列で扱う場合に便利です。  
- **PydanticDictJSON**は、辞書型データのキーと値の型を明示的に検証したい場合、特に特定のキー型と値型のパターンがある場合に最適です。

これらのカスタム型を状況に応じて適切に選択することで、SQLAlchemyとPydanticの利点を組み合わせ、MySQLのJSON列をより効果的に活用できます。特に型安全性と開発効率の向上に貢献します。