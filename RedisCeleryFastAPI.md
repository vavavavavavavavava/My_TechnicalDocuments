# Redis、Celery、FastAPIを使用したマイクロサービスアーキテクチャの構築手順

このドキュメントでは、Windows環境でRedisをWSL2上で動かし、FastAPIとCeleryを組み合わせたシンプルなマイクロサービスアーキテクチャを構築する方法を説明します。今回のサンプルプログラムでは、Celeryタスクの結果をクライアントが取得できるようにし、FastAPIのエンドポイントは **app.py** に定義、**main.py** から uvicorn を起動する構成となっています。また、タスクは "default" キューで処理するため、Celeryワーカー起動時には **-Q default** オプションが必要です。

---

## 目次

- [Redis、Celery、FastAPIを使用したマイクロサービスアーキテクチャの構築手順](#redisceleryfastapiを使用したマイクロサービスアーキテクチャの構築手順)
  - [目次](#目次)
  - [1. 概要](#1-概要)
  - [2. 環境の準備](#2-環境の準備)
    - [2.1 WSL2のインストールとセットアップ](#21-wsl2のインストールとセットアップ)
    - [2.2 Redisのインストールと起動](#22-redisのインストールと起動)
    - [2.3 Pythonと必要なパッケージのインストール](#23-pythonと必要なパッケージのインストール)
  - [3. プログラムの構成](#3-プログラムの構成)
    - [3.1 FastAPIアプリケーションとエンドポイントの作成](#31-fastapiアプリケーションとエンドポイントの作成)
    - [3.2 Celeryの設定とタスクの定義](#32-celeryの設定とタスクの定義)
    - [3.3 uvicornを使ったFastAPIサーバーの起動](#33-uvicornを使ったfastapiサーバーの起動)
    - [3.4 Celeryワーカーの起動（-Q default オプションの指定）](#34-celeryワーカーの起動-q-default-オプションの指定)
    - [3.5 複数フェーズのタスク処理とワーカーの分割](#35-複数フェーズのタスク処理とワーカーの分割)
  - [4. 動作確認](#4-動作確認)
  - [5. まとめ](#5-まとめ)

---

## 1. 概要

- **FastAPI**: 軽量で高性能なPython製Webフレームワーク。  
- **Celery**: 分散タスクキューを提供する非同期タスクジョブキュー。  
- **Redis**: インメモリデータストア。Celeryのブローカーおよびバックエンドとして使用。  
- **WSL2**: Windows上でLinux環境を実行するためのサブシステム。Redisを実行するために使用。

---

## 2. 環境の準備

### 2.1 WSL2のインストールとセットアップ

1. **WSLの有効化**  
   管理者権限でPowerShellを開き、以下のコマンドを実行します。

   ```powershell
   dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
   dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
   ```

2. **再起動**  
   コンピューターを再起動します。

3. **WSL2をデフォルトに設定**

   ```powershell
   wsl --set-default-version 2
   ```

4. **Linuxディストリビューションのインストール**  
   Microsoft StoreからUbuntuをインストールします。

5. **Ubuntuの初期設定**  
   Ubuntuを起動し、ユーザー名とパスワードを設定します。

### 2.2 Redisのインストールと起動

1. **パッケージリストの更新**

   ```bash
   sudo apt update
   ```

2. **Redisのインストール**

   ```bash
   sudo apt install redis-server
   ```

3. **Redisの起動**

   ```bash
   sudo service redis-server start
   ```

4. **Redisの動作確認**

   ```bash
   redis-cli ping
   ```

   `PONG` と表示されれば成功です。

### 2.3 Pythonと必要なパッケージのインストール

1. **Pythonのインストール**  
   [Python公式サイト](https://www.python.org/downloads/windows/)からPythonをダウンロードしてインストールします。（インストール時に「Add Python to PATH」にチェックを入れてください。）

2. **仮想環境の作成**

   ```bash
   mkdir myproject
   cd myproject
   python -m venv venv
   venv\Scripts\activate
   ```

3. **必要なパッケージのインストール**

   ```bash
   pip install fastapi uvicorn celery redis
   ```

---

## 3. プログラムの構成

今回のサンプルプログラムは、以下の4ファイルで構成されています。

### 3.1 FastAPIアプリケーションとエンドポイントの作成

**app.py**

```python
from fastapi import FastAPI, HTTPException
from tasks import add
from celery.result import AsyncResult
from celery_app import celery_app

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello, World!"}

# /add エンドポイント：クエリパラメータ a, b を受け取り、Celeryタスクを非同期実行
@app.get("/add")
def call_add(a: int = 1, b: int = 2):
    task = add.delay(a, b)
    return {"task_id": task.id, "status": task.status}

# /result/{task_id} エンドポイント：タスクIDを指定してタスクの状態と結果を取得
@app.get("/result/{task_id}")
def get_result(task_id: str):
    task = AsyncResult(task_id, app=celery_app)
    if task.state == 'PENDING':
        return {"status": task.state, "result": None}
    elif task.state == 'FAILURE':
        raise HTTPException(status_code=500, detail=str(task.info))
    else:
        return {"status": task.state, "result": task.result}
```

### 3.2 Celeryの設定とタスクの定義

**celery_app.py**

```python
from celery import Celery

# Redisをブローカーおよびバックエンドとして設定
celery_app = Celery(
    "worker",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/0"
)

# tasks.add タスクを "default" キューで実行するようルーティング設定
celery_app.conf.task_routes = {
    "tasks.add": {"queue": "default"}
}
```

**tasks.py**

```python
from celery_app import celery_app

# サンプルタスク：2つの数値の和を計算
@celery_app.task
def add(x, y):
    return x + y
```

### 3.3 uvicornを使ったFastAPIサーバーの起動

**main.py**

```python
import uvicorn

if __name__ == "__main__":
    uvicorn.run("app:app", host="0.0.0.0", port=8000, reload=True)
```

### 3.4 Celeryワーカーの起動（-Q default オプションの指定）

タスクルーティングで "default" キューが指定されているため、Celeryワーカーは以下のコマンドで **-Q default** オプションを付与して起動します。

```bash
celery -A celery_app worker -Q default --loglevel=info
```

### 3.5 複数フェーズのタスク処理とワーカーの分割

従来のシンプルなタスク実行に加え、処理内容が複数の段階に分かれている場合、Celeryの **chain** 機能を利用して、各フェーズを連続的に実行することが可能です。  
たとえば、あるタスクが2段階の処理を行うとします。

- **フェーズ1**: 高速だがメモリ消費が激しい処理。  
  ※このフェーズはリソース消費が大きいため、ワーカーは **concurrency=1** でシリアルに処理するのが望ましい。

- **フェーズ2**: 並列実行が可能な処理。  
  ※こちらは複数のタスクを同時に処理できるよう、**concurrency=8** などの設定でワーカーを起動します。

この仕組みを実装するため、タスクを以下のように定義します。

```python
# multi_phase_tasks.py

from celery import Celery, chain

# Celeryの設定（ブローカーとバックエンドにはRedisを使用）
celery_app = Celery("multi_phase", broker="redis://localhost:6379/0", backend="redis://localhost:6379/0")

@celery_app.task(queue="phase1")
def phase_one(data):
    """
    フェーズ1：高速だがメモリを多く使用する重い前処理
    """
    # 例：入力データの前処理
    processed_data = {"step1_result": data}
    return processed_data

@celery_app.task(queue="phase2")
def phase_two(result_from_phase_one):
    """
    フェーズ2：並列実行可能な処理
    """
    # 例：前処理の結果をもとに追加計算を実施
    final_output = {"final_result": result_from_phase_one}
    return final_output
```

上記のタスクを **chain** を利用して連結し、FastAPIから1つのタスクIDで管理する例は以下の通りです。

```python
# app_chain.py

from fastapi import FastAPI
from multi_phase_tasks import phase_one, phase_two
from celery import chain

app = FastAPI()

@app.post("/start_multi_phase")
def start_multi_phase(data: dict):
    """
    クライアントからのリクエストを受け、フェーズ1とフェーズ2を連結して実行します。
    単一のtask_idで最終結果が取得可能です。
    """
    task_chain = chain(phase_one.s(data), phase_two.s())
    async_result = task_chain.apply_async()
    return {"task_id": async_result.id}
```

#### ワーカーの起動方法

それぞれのキューに対して、適切な **concurrency** 設定でワーカーを起動します。たとえば:

- **フェーズ1用ワーカー（concurrency=1）**  
  ```bash
  celery -A multi_phase_tasks worker -Q phase1 --concurrency=1 --loglevel=info
  ```

- **フェーズ2用ワーカー（concurrency=8）**  
  ```bash
  celery -A multi_phase_tasks worker -Q phase2 --concurrency=8 --loglevel=info
  ```

このように、**chain** を用いてタスクを連結し、各フェーズごとに最適なワーカー設定を行うことで、全体として単一のタスクIDで結果を管理しながらも、各フェーズの特性に応じたリソース管理が実現できます。

---

## 4. 動作確認

1. **Redisサーバーの起動確認**  
   Ubuntuターミナルで Redis が起動していることを確認します。

   ```bash
   sudo service redis-server status
   ```

2. **FastAPIサーバーの起動**  
   WindowsのコマンドプロンプトまたはPowerShellで以下を実行します。

   ```bash
   venv\Scripts\activate
   python main.py
   ```

3. **Celeryワーカーの起動**  
   - 既存のタスク用ワーカーは以下のコマンドで起動します。

     ```bash
     celery -A celery_app worker -Q default --loglevel=info
     ```

   - 複数フェーズのタスク用ワーカーは、上記の通りそれぞれ別々に起動します。
     - フェーズ1用:  
       ```bash
       celery -A multi_phase_tasks worker -Q phase1 --concurrency=1 --loglevel=info
       ```
     - フェーズ2用:  
       ```bash
       celery -A multi_phase_tasks worker -Q phase2 --concurrency=8 --loglevel=info
       ```

4. **APIエンドポイントのテスト**  
   - `http://localhost:8000/` にアクセスし、「Hello, World!」が表示されることを確認。  
   - `http://localhost:8000/add?a=3&b=5` にアクセスしてタスクIDと初期状態が返されることを確認。  
   - 返されたタスクIDを用いて `http://localhost:8000/result/{task_id}` にアクセスし、タスクが完了していれば計算結果（例：8）が返されます。  
   - さらに、複数フェーズのタスク処理をテストするため、`http://localhost:8000/start_multi_phase` にPOSTリクエストを送り、単一のタスクIDで全フェーズの処理が連結されていることを確認してください。

---

## 5. まとめ

本ドキュメントでは、Redis、Celery、FastAPI を組み合わせたシンプルなマイクロサービスアーキテクチャの構築方法を示しました。  

- **FastAPI** のエンドポイントは **app.py** に定義し、**main.py** から uvicorn を起動しています。  
- **Celery** は Redis をブローカー・バックエンドとして利用し、タスクは "default" キューで実行するようにルーティング設定しています。  
- Celery ワーカーは **-Q default** オプションを指定して起動することで、該当キューのタスクのみを処理します。  
- さらに、Celeryの **chain** 機能を用いることで、複数フェーズのタスク処理を連結し、各フェーズに応じたワーカー（フェーズ1はconcurrency=1、フェーズ2はconcurrency=8）で実行する仕組みを導入できます。これにより、単一のタスクIDで全体の結果を管理しつつ、各フェーズのリソース消費や並列度を最適化することが可能となります。

**注意事項**:  

- 本番環境では、RedisやAPIのセキュリティ設定、エラーハンドリングの強化を検討してください。  
- 必要に応じて、Celery や Redis の設定を最適化し、システムのスケーラビリティや負荷分散を図ってください。