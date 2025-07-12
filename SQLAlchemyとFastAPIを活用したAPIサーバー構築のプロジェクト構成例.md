# SQLAlchemyとFastAPIを活用したAPIサーバー構築のプロジェクト構成例

## 1. はじめに

このレポートは、SQLAlchemyとFastAPIを組み合わせたAPIサーバーの構築におけるプロジェクト構成例と設計思想についてまとめたものです。本例では、共通プロジェクトで管理されたデータモデルおよびDB接続設定を活用しつつ、FastAPIの依存性注入機能（Depends）を利用して効率的なDBセッション管理やパラメータ処理を実現する方法を紹介します。さらに、リポジトリパターンとサービス層の明確な役割分担により、データアクセスとビジネスロジックの分離を図り、保守性と拡張性に優れたシステムアーキテクチャを構築するための具体例を示します。

本レポートは以下の観点から構成されています。

- **プロジェクトフォルダ構成:**  
  src直下に各サブフォルダ（api、core、db、repositories、schemas、services）を配置し、testsは別ディレクトリとして管理する例を示します。

- **FastAPIの依存性注入（Depends）の活用:**  
  DBセッション管理や共通パラメータ処理、リクエストボディの扱いについて具体例を挙げ、その挙動や注意点を解説します。

- **リポジトリとサービスの役割分担:**  
  リポジトリ層でのデータアクセスの抽象化と、サービス層でのビジネスロジック実装の分離により、システムの柔軟性とテスト容易性を高める設計手法を解説します。

- **全体のデータフロー:**  
  クライアントからのリクエストがどのように処理され、各層を経由してレスポンスが生成されるか、具体的な流れを整理します。

---

## 2. プロジェクトフォルダ構成

以下は、SQLAlchemyとFastAPIを利用する際の一例のプロジェクト構成です。ここでは、DBのエンジンは共通プロジェクトで管理され、データモデルも共通プロジェクトからimportする前提としています。

```
project/
├── src/
│   ├── main.py             # FastAPIアプリのエントリーポイント
│   ├── api/                # エンドポイント関連
│   │   ├── __init__.py
│   │   └── routes/
│   │       ├── __init__.py
│   │       └── sample.py   # サンプルのエンドポイント定義
│   ├── core/               # アプリケーション設定、環境変数など
│   │   ├── __init__.py
│   │   └── config.py
│   ├── db/                 # DBセッション管理（共通のエンジン利用）
│   │   ├── __init__.py
│   │   └── session.py
│   ├── repositories/       # データアクセス層（リポジトリパターン）
│   │   ├── __init__.py
│   │   └── user_repository.py
│   ├── schemas/            # Pydanticスキーマ
│   │   ├── __init__.py
│   │   └── user.py
│   └── services/           # ビジネスロジック層
│       ├── __init__.py
│       └── user_service.py
└── tests/                  # テストコード
    ├── __init__.py
    └── test_sample.py
```

### 各ディレクトリの役割

- **main.py:**  
  FastAPIのインスタンス作成、ミドルウェア設定、ルーティングの登録を行うエントリーポイントです。

- **api/routes:**  
  APIの各エンドポイントをリソースごとに分割して定義します。2段構造にすることで、エンドポイントの整理や拡張が容易になります。

- **core:**  
  環境変数や設定情報（例：DATABASE_URLなど）を管理します。

- **db:**  
  共通プロジェクトで定義されたDBエンジンを利用し、リクエストごとにDBセッションを生成・管理する依存関数を提供します。

- **repositories:**  
  SQLAlchemyを用いたCRUD操作など、直接的なデータアクセスのロジックを隠蔽し、外部に対して統一的なインターフェースを提供します。

- **schemas:**  
  Pydanticを利用した入力バリデーションやレスポンスデータのシリアライゼーションに用いるモデルを定義します。

- **services:**  
  リポジトリを利用して取得したデータに対してビジネスロジックや加工処理を実装し、エンドポイント層をシンプルに保ちます。

- **tests:**  
  各層の単体テストや統合テストを記述し、システム全体の品質を担保します。

---

## 3. FastAPIのDependsの活用例

FastAPIの`Depends`は依存性注入の仕組みとして、以下のような多彩な用途で利用されます。

### 3.1. DBセッション管理

リクエストごとにDBセッションを生成し、処理後に自動的にクローズするための依存関数です。

```python
# src/db/session.py
from sqlalchemy.orm import sessionmaker
from common.db.base import engine  # 共通プロジェクトで作成されたエンジン

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

ルートハンドラーでは次のように利用します。

```python
# src/api/routes/sample.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from db.session import get_db
from schemas.user import UserOut
from services.user_service import fetch_users

router = APIRouter()

@router.get("/users", response_model=list[UserOut])
def read_users(db: Session = Depends(get_db)):
    users = fetch_users(db)
    return users
```

### 3.2. クエリパラメータの共通処理

複数のエンドポイントで共通のクエリパラメータを使う場合に依存関数として定義します。

```python
from fastapi import Query

def common_query_params(q: str = Query(..., description="検索キーワード"), page: int = Query(1, ge=1, description="ページ番号")):
    return {"q": q, "page": page}

@router.get("/search")
def search(params: dict = Depends(common_query_params)):
    # paramsは {"q": ..., "page": ...} の形式
    return params
```

### 3.3. リクエストボディとしての利用

通常は依存関数のパラメータはクエリとして扱われますが、明示的に`Body()`を用いることでリクエストボディとして扱えます。

```python
from fastapi import Body

def common_body_params(item: dict = Body(...)):
    # 前処理などがあれば実施
    return item

@router.post("/items")
def create_item(item_data: dict = Depends(common_body_params)):
    return {"item": item_data}
```

---

## 4. リポジトリとサービスの役割分担

システムの保守性と拡張性を高めるために、データアクセス層とビジネスロジック層を明確に分離します。

### 4.1. リポジトリ層

- **主な責務:**  
  SQLAlchemyを用いたデータベースの直接操作（CRUD処理など）を抽象化し、外部からは統一されたインターフェースを提供します。

- **実装例:**

  ```python
  # src/repositories/user_repository.py
  from sqlalchemy.orm import Session
  from common.models.user import User  # 共通プロジェクトからのインポート

  def get_all_users(db: Session):
      return db.query(User).all()

  def get_user_by_id(db: Session, user_id: int):
      return db.query(User).filter(User.id == user_id).first()

  def create_user(db: Session, user_data: dict):
      user = User(**user_data)
      db.add(user)
      db.commit()
      db.refresh(user)
      return user
  ```

### 4.2. サービス層

- **主な責務:**  
  リポジトリ層を活用しつつ、ユーザー登録時の重複チェックやデータ加工など、アプリケーション固有のビジネスロジックを実装します。

- **実装例:**

  ```python
  # src/services/user_service.py
  from sqlalchemy.orm import Session
  from repositories.user_repository import get_all_users, get_user_by_id, create_user

  def fetch_users(db: Session):
      # ここで必要なデータ加工やフィルタリングを実施可能
      return get_all_users(db)

  def register_new_user(db: Session, user_data: dict):
      # 重複チェックの例
      if get_user_by_id(db, user_data.get("id")):
          raise ValueError("User already exists")
      return create_user(db, user_data)
  ```

---

## 5. 全体のデータフロー

1. **リクエスト受信:**  
   クライアントからFastAPIのエンドポイントにリクエストが送信されます。

2. **依存性の解決:**  
   FastAPIは`Depends`を通じて、DBセッションや共通パラメータ、認証情報などを自動的に注入します。

3. **エンドポイント処理:**  
   各エンドポイントは、依存関数で解決された値（例：DBセッション）を用いてサービス層の関数を呼び出し、ビジネスロジックを実行します。

4. **ビジネスロジック実行:**  
   サービス層はリポジトリ層の関数を呼び出し、データベース操作を行い、必要な加工や検証を実施します。

5. **レスポンス生成:**  
   取得・加工されたデータをPydanticスキーマで整形し、クライアントへレスポンスを返します。

6. **リソース解放:**  
   リクエスト処理完了後、`get_db`などの依存関数により自動的にDBセッションがクローズされ、リソースが解放されます。

---

## 6. まとめ

本レポートでは、SQLAlchemyとFastAPIを活用したAPIサーバー構築時のプロジェクト構成例を以下の観点から整理しました。

- **フォルダ構成:**  
  src直下にapi、core、db、repositories、schemas、servicesの各サブフォルダを配置し、テストコードはtestsディレクトリで管理する設計を採用。

- **依存性注入の活用:**  
  FastAPIのDependsを利用し、DBセッション管理、共通パラメータ抽出、リクエストボディの明示的定義など、多様なケースに対応可能な仕組みを実装。

- **リポジトリとサービスの分離:**  
  データアクセス（リポジトリ）とビジネスロジック（サービス）を明確に分割することで、各層の責務を明確にし、システムの保守性とテスト容易性を向上。

- **全体のデータフロー:**  
  リクエストからレスポンス生成までの流れを整理し、各層がどのように連携して処理を行うかを明示。

この構成例と設計思想に基づくアプローチにより、柔軟かつ拡張性の高いAPIサーバーの実装が可能となり、今後の機能拡張や保守作業も容易になります。