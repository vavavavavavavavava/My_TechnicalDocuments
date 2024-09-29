# SqlAlchemyとAlembicを使ったデータベース操作入門

## はじめに

**SqlAlchemy**と**Alembic**は、Pythonでデータベース操作を行う際に非常に有用なツールです。SqlAlchemyはORM（Object Relational Mapper）として、データベース操作をオブジェクト指向的に扱うことができ、Alembicはデータベースのマイグレーション（スキーマのバージョン管理）を容易にします。本ガイドでは、これらのツールを使用してデータベースを操作し、外部キーで関連付けられたデータの取得方法までを解説します。

---

## 目次

1. [環境設定と前提条件](#環境設定と前提条件)
2. [SqlAlchemyモデルの作成](#SqlAlchemyモデルの作成)
3. [Alembicによるデータベースマイグレーション](#Alembicによるデータベースマイグレーション)
4. [リレーションシップの定義とデータの関連付け](#リレーションシップの定義とデータの関連付け)
5. [データの追加と取得](#データの追加と取得)
6. [外部キーで関連付けられたデータの取得](#外部キーで関連付けられたデータの取得)
7. [まとめ](#まとめ)

---

## 環境設定と前提条件

### 必要なパッケージのインストール

以下のコマンドで必要なパッケージをインストールします。

```bash
pip install sqlalchemy alembic
```

### 仮想環境の作成（推奨）

プロジェクトごとに仮想環境を作成することをお勧めします。

```bash
python -m venv venv
source venv/bin/activate  # Windowsの場合は venv\Scripts\activate
```

---

## SqlAlchemyモデルの作成

### モデルファイルの作成

`models.py`というファイルを作成し、データベースのモデルを定義します。

```python
# models.py
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'  # テーブル名
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)
```

---

## Alembicによるデータベースマイグレーション

### Alembicの初期化

プロジェクトディレクトリで以下のコマンドを実行し、Alembicを初期化します。

```bash
alembic init alembic
```

### Alembicの設定ファイルの編集

#### `alembic.ini`

データベースの接続先を設定します。ここではSQLiteを使用します。

```ini
# alembic.ini
sqlalchemy.url = sqlite:///example.db
```

#### `alembic/env.py`

モデルのメタデータをAlembicに認識させます。

```python
# alembic/env.py
import sys
sys.path.append('.')
from models import Base  # 追加

target_metadata = Base.metadata  # 修正
```

### マイグレーションスクリプトの自動生成

```bash
alembic revision --autogenerate -m "Create users table"
```

### マイグレーションの適用

```bash
alembic upgrade head
```

---

## リレーションシップの定義とデータの関連付け

### モデルに新しいテーブルとリレーションシップを追加

`Address`モデルを追加し、`User`モデルと外部キーで関連付けます。

```python
# models.py
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)
    addresses = relationship("Address", back_populates="user")  # リレーションシップを追加

class Address(Base):
    __tablename__ = 'addresses'
    id = Column(Integer, primary_key=True)
    email_address = Column(String, nullable=False)
    user_id = Column(Integer, ForeignKey('users.id'))  # 外部キーを定義
    user = relationship("User", back_populates="addresses")  # リレーションシップを定義
```

### マイグレーションスクリプトの生成と適用

```bash
alembic revision --autogenerate -m "Add addresses table"
alembic upgrade head
```

---

## データの追加と取得

### データの追加

```python
# add_user_with_address.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from models import User, Address

engine = create_engine('sqlite:///example.db')
Session = sessionmaker(bind=engine)
session = Session()

# ユーザーを作成
new_user = User(name='Hanako', email='hanako@example.com')

# アドレスを作成し、ユーザーに関連付け
address1 = Address(email_address='hanako.home@example.com', user=new_user)
address2 = Address(email_address='hanako.work@example.com', user=new_user)

# データベースに保存
session.add(new_user)
session.add(address1)
session.add(address2)
session.commit()
```

---

## 外部キーで関連付けられたデータの取得

### リレーションシップを使用したデータの取得

```python
# query_user_with_addresses.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from models import User

engine = create_engine('sqlite:///example.db')
Session = sessionmaker(bind=engine)
session = Session()

# ユーザーを取得
user = session.query(User).filter_by(name='Hanako').first()

print(f"User: {user.name}, Email: {user.email}")

# 関連するアドレスを取得
for address in user.addresses:
    print(f"Address: {address.email_address}")
```

**実行結果：**

```
User: Hanako, Email: hanako@example.com
Address: hanako.home@example.com
Address: hanako.work@example.com
```

### 逆方向のリレーションシップを使用したデータの取得

```python
# query_address_with_user.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from models import Address

engine = create_engine('sqlite:///example.db')
Session = sessionmaker(bind=engine)
session = Session()

# アドレスを取得
address = session.query(Address).filter_by(email_address='hanako.home@example.com').first()

# 関連するユーザーを取得
user = address.user

print(f"Address: {address.email_address}")
print(f"Belongs to User: {user.name}, Email: {user.email}")
```

**実行結果：**

```
Address: hanako.home@example.com
Belongs to User: Hanako, Email: hanako@example.com
```

### JOINを使用したクエリの実行

```python
# join_query.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, joinedload
from models import User

engine = create_engine('sqlite:///example.db')
Session = sessionmaker(bind=engine)
session = Session()

# ユーザーとアドレスを結合して取得
users = session.query(User).options(joinedload(User.addresses)).all()

for user in users:
    print(f"User: {user.name}")
    for address in user.addresses:
        print(f" - Address: {address.email_address}")
```

**実行結果：**

```
User: Hanako
 - Address: hanako.home@example.com
 - Address: hanako.work@example.com
```

---

## まとめ

- **SqlAlchemy**を使用することで、データベース操作をオブジェクト指向的に行うことができます。
- **Alembic**はデータベースのスキーマ変更を管理し、マイグレーションを容易にします。
- **リレーションシップの定義**により、外部キーで関連付けられたデータを直感的に操作できます。
- **ORMの利点**として、直接SQLを記述せずにデータベース操作が可能で、コードの可読性と保守性が向上します。
- **データの取得**では、`relationship`を通じて関連データを簡単に取得できます。

---

## 参考情報

- [SqlAlchemy公式ドキュメント](https://www.sqlalchemy.org/)
- [Alembic公式ドキュメント](https://alembic.sqlalchemy.org/)

---

**ご不明な点や追加のご質問がありましたら、お気軽にお知らせください。**