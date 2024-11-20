# マイクロサービスアーキテクチャにおけるFastAPIを用いたAPIゲートウェイと統合Swagger UIの実装

このドキュメントでは、FastAPIを使用したマイクロサービスアーキテクチャで以下の構成を実現する手順を説明します。

1. **バックエンドサーバー**  
   - 複数のFastAPIアプリケーション（`server1`, `server2`, `server3`）を独立して動作させます。
   - `root_path`を設定して、APIゲートウェイ経由で適切に動作するようにします。

2. **APIゲートウェイ**  
   - バックエンドサーバーへのプロキシを実装します。
   - `/all_docs`エンドポイントで、複数の`openapi.json`を統合し、一つのSwagger UIを提供します。

---

## 構成

### 全体の構成図

```plaintext
Client
  |
API Gateway (FastAPI)
  |
  +-- /server1/docs --> Backend Server1 (FastAPI)
  |
  +-- /server2/docs --> Backend Server2 (FastAPI)
  |
  +-- /server3/docs --> Backend Server3 (FastAPI)
  |
  +-- /all_docs --> Combined Swagger UI
```

---

## 実装

### バックエンドサーバー

各バックエンドサーバーは、`FastAPI`アプリケーションとして動作します。以下に`server1`の例を示します。`server2`と`server3`も同様に実装します。

#### **`server1.py`**

```python
from fastapi import FastAPI

app = FastAPI(root_path="/server1")

@app.get("/items/")
async def read_items():
    return {"server": "server1", "items": ["item1", "item2"]}
```

#### **`server2.py`**

```python
from fastapi import FastAPI

app = FastAPI(root_path="/server2")

@app.get("/users/")
async def read_users():
    return {"server": "server2", "users": ["user1", "user2"]}
```

#### **`server3.py`**

```python
from fastapi import FastAPI

app = FastAPI(root_path="/server3")

@app.get("/orders/")
async def read_orders():
    return {"server": "server3", "orders": ["order1", "order2"]}
```

---

### APIゲートウェイ

APIゲートウェイは以下の機能を実装します。

- **プロキシエンドポイント**  
  `/server1/*`などのリクエストをバックエンドサーバーに転送します。
  
- **統合Swagger UI**  
  `/all_docs`エンドポイントで、すべてのバックエンドサーバーの`openapi.json`を統合したSwagger UIを提供します。

#### **`api_gateway.py`**

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse, RedirectResponse
from fastapi.middleware.cors import CORSMiddleware
import httpx
from starlette.responses import Response
from typing import List

app = FastAPI()

# CORS設定
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# バックエンドサーバー情報
backend_servers = {
    "server1": "http://localhost:8001",
    "server2": "http://localhost:8002",
    "server3": "http://localhost:8003",
}

# プロキシエンドポイント設定
for server_name, server_url in backend_servers.items():
    @app.api_route(f"/{server_name}/{{path:path}}", methods=["GET", "POST", "PUT", "DELETE", "OPTIONS", "HEAD"])
    async def proxy(request: Request, path: str, server_name=server_name, server_url=server_url):
        url = f"{server_url}/{path}"
        headers = dict(request.headers)
        headers["host"] = server_url.replace("http://", "")
        async with httpx.AsyncClient() as client:
            response = await client.request(
                request.method,
                url,
                headers=headers,
                params=request.query_params,
                content=await request.body()
            )
        return Response(
            content=response.content,
            status_code=response.status_code,
            headers=dict(response.headers),
            media_type=response.headers.get("content-type")
        )

# /all_docsエンドポイント
@app.get("/all_docs", include_in_schema=False)
async def get_all_docs():
    return RedirectResponse(url="/docs_all")

# 統合OpenAPIスキーマを生成
from fastapi.openapi.utils import get_openapi

@app.get("/openapi_all.json", include_in_schema=False)
async def get_combined_openapi():
    async with httpx.AsyncClient() as client:
        openapi_schemas = []
        for server_name, server_url in backend_servers.items():
            resp = await client.get(f"{server_url}/openapi.json")
            schema = resp.json()
            new_paths = {}
            for path, path_item in schema["paths"].items():
                new_path = f"/{server_name}{path}"
                new_paths[new_path] = path_item
            schema["paths"] = new_paths
            openapi_schemas.append(schema)
    openapi = get_openapi(
        title="Combined API",
        version="1.0.0",
        routes=app.routes,
    )
    openapi["paths"] = {}
    for schema in openapi_schemas:
        openapi["paths"].update(schema["paths"])
    openapi["components"] = {"schemas": {}}
    for schema in openapi_schemas:
        if "components" in schema and "schemas" in schema["components"]:
            openapi["components"]["schemas"].update(schema["components"]["schemas"])
    return JSONResponse(openapi)

# 統合Swagger UIを提供
from fastapi.openapi.docs import get_swagger_ui_html

@app.get("/docs_all", include_in_schema=False)
async def get_swagger_documentation():
    return get_swagger_ui_html(
        openapi_url="/openapi_all.json",
        title="Combined API Docs"
    )
```

---

## 起動手順

### 1. バックエンドサーバーの起動

以下のコマンドを使用して、各バックエンドサーバーを起動します。

```bash
uvicorn server1:app --port 8001
uvicorn server2:app --port 8002
uvicorn server3:app --port 8003
```

### 2. APIゲートウェイの起動

以下のコマンドでAPIゲートウェイを起動します。

```bash
uvicorn api_gateway:app --port 8000
```

---

## 動作確認

### 1. 個別のSwagger UI

以下のURLにアクセスして、各バックエンドサーバーのSwagger UIを確認します。

- `http://localhost:8000/server1/docs`
- `http://localhost:8000/server2/docs`
- `http://localhost:8000/server3/docs`

### 2. 統合Swagger UI

`http://localhost:8000/all_docs`にアクセスして、統合されたSwagger UIを確認します。

---

## 注意事項

1. **エラーハンドリング**  
   バックエンドサーバーの障害や接続エラーに対する処理を追加する必要があります。

2. **認証とセキュリティ**  
   認証が必要な場合、APIゲートウェイで適切にトークンを設定してください。

3. **スキーマ競合**  
   同じ名前のスキーマが複数存在すると統合時に競合が発生する可能性があります。スキーマ名をユニークにするなどの対策を検討してください。

---

これで、マイクロサービス環境でのFastAPIを用いたAPIゲートウェイと統合Swagger UIの実装が完了しました。必要に応じてカスタマイズしてください！