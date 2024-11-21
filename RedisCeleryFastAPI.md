# Redis、Celery、FastAPIを使用したマイクロサービスアーキテクチャの構築手順

このドキュメントでは、Windows環境でRedisをWSL2上で動かし、FastAPIとCeleryを組み合わせてマイクロサービスアーキテクチャを構築する方法を説明します。各サーバーは独自のタスク処理を行い、サーバーごとに異なるワーカープロセス数を設定します。

---

## 目次

1. **概要**
2. **環境の準備**
   - WSL2のインストールとセットアップ
   - Redisのインストールと起動
   - Pythonと必要なパッケージのインストール
3. **プログラムの構成**
   - FastAPIアプリケーションの作成
   - Celeryの設定とタスクの定義
   - ワーカーの起動方法
   - Periodicタスクの設定
   - 特定のタスクキューの設定
   - FastAPIで特定のタスクキューを使用する方法
4. **動作確認**
5. **まとめ**

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

   再起動後、PowerShellで以下を実行します。

   ```powershell
   wsl --set-default-version 2
   ```

4. **Linuxディストリビューションのインストール**

   Microsoft StoreからUbuntuをインストールします。

5. **Ubuntuの初期設定**

   Ubuntuを起動し、ユーザー名とパスワードを設定します。

### 2.2 Redisのインストールと起動

1. **パッケージリストの更新**

   Ubuntuターミナルで以下を実行します。

   ```bash
   sudo apt update
   ```

2. **Redisのインストール**

   ```bash
   sudo apt install redis-server
   ```

3. **Redisの設定ファイルを編集（オプション）**

   必要に応じて`/etc/redis/redis.conf`を編集します。

4. **Redisの起動**

   ```bash
   sudo service redis-server start
   ```

5. **Redisの動作確認**

   ```bash
   redis-cli ping
   ```

   `PONG`と表示されれば成功です。

### 2.3 Pythonと必要なパッケージのインストール

1. **Pythonのインストール**

   [Python公式サイト](https://www.python.org/downloads/windows/)からPythonをダウンロードしてインストールします。インストール時に「Add Python to PATH」にチェックを入れます。

2. **仮想環境の作成**

   コマンドプロンプトまたはPowerShellでプロジェクトディレクトリを作成し、仮想環境をセットアップします。

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

### 3.1 FastAPIアプリケーションの作成

1. **`main.py`****の作成**
   ```python
   # main.py
   from fastapi import FastAPI
   from tasks import sample_task, high_priority_task, low_priority_task

   app = FastAPI()

   @app.get("/run-task")
   def run_task():
       result = sample_task.delay()
       return {"task_id": result.id}

   @app.get("/run-high-priority-task")
   def run_high_priority_task():
       result = high_priority_task.apply_async(queue='high_priority')
       return {"task_id": result.id}

   @app.get("/run-low-priority-task")
   def run_low_priority_task():
       result = low_priority_task.apply_async(queue='low_priority')
       return {"task_id": result.id}
   ```

### 3.2 Celeryの設定とタスクの定義

1. **`celery_app.py`****の作成**

   ```python
   # celery_app.py
   from celery import Celery

   celery = Celery(
       'myproject',
       broker='redis://localhost:6379/0',
       backend='redis://localhost:6379/0'
   )
   ```

2. **`tasks.py`****の作成**

   ```python
   # tasks.py
   from celery_app import celery

   @celery.task
   def sample_task():
       # タスクの処理内容を記述
       return "タスクが完了しました。"

   @celery.task(queue='high_priority')
   def high_priority_task():
       return "高優先度タスクが完了しました。"

   @celery.task(queue='low_priority')
   def low_priority_task():
       return "低優先度タスクが完了しました。"
   ```

### 3.3 ワーカーの起動方法

Celeryワーカーを複数起動することで、各プロセスが独立してタスクを処理することができます。これにより、負荷分散が可能になります。

例えば、2つのワーカープロセスを起動するには、以下のように2つのターミナルを開いてそれぞれでコマンドを実行します。

```bash
celery -A tasks worker --loglevel=info
```

複数のワーカーを使用する際、`--concurrency`オプションとの違いについて理解しておくことが重要です。

- **複数ワーカープロセス**: 各プロセスは独立しており、システム全体のリソース（CPUコアなど）を活用して並列にタスクを処理します。この方法は、複数のプロセスを使用することで、メモリの分離やプロセスのクラッシュが他のワーカーに影響を与えないという利点があります。

- **コンカレンシー（--concurrency オプション）**: 単一のワーカープロセス内で複数のスレッドを使用してタスクを並行処理します。これにより、プロセスのオーバーヘッドを減らすことができますが、メモリ共有による問題やスレッド間の競合に注意が必要です。

例えば、ワーカー数を4に設定する場合:

```bash
celery -A tasks worker --loglevel=info --concurrency=4
```

この設定では1つのプロセス内で4つのタスクを並行して処理します。

### 3.4 Periodicタスクの設定

Celeryを使用して定期的なタスクを実行することも可能です。これは、例えば毎日決まった時間に特定の処理を行う場合に便利です。

1. **Periodicタスクの設定ファイルの作成**

   `tasks.py`に定期タスクの設定を追加します。

   ```python
   # tasks.py
   from celery_app import celery
   from celery.schedules import crontab

   @celery.task
   def sample_task():
       # タスクの処理内容を記述
       return "タスクが完了しました。"

   # 定期タスクの設定
   @celery.on_after_configure.connect
   def setup_periodic_tasks(sender, **kwargs):
       # 10秒ごとにsample_taskを実行
       sender.add_periodic_task(10.0, sample_task.s(), name='add every 10 seconds')

       # 毎日午前7時にsample_taskを実行
       sender.add_periodic_task(
           crontab(hour=7, minute=0),
           sample_task.s(),
           name='run every morning at 7'
       )
   ```

   この設定により、10秒ごとに`sample_task`を実行するタスクと、毎朝7時に実行するタスクが追加されます。

2. **定期タスクの確認**

   Celeryビートを使用してスケジュールを管理します。以下のコマンドでビートを起動します。

   ```bash
   celery -A tasks beat --loglevel=info
   ```

   その後、通常のワーカーも起動して定期タスクを処理できるようにします。

   ```bash
   celery -A tasks worker --loglevel=info
   ```

### 3.5 特定のタスクキューの設定

特定のタスクキューを使用して、タスクを分離し、異なるワーカーに特定のタスクのみを処理させることが可能です。これは、タスクの優先度や役割を分けたい場合に便利です。

1. **特定のキューにタスクを紐づける**

   `tasks.py`で、特定のキューにタスクを紐づけます。

   ```python
   # tasks.py
   from celery_app import celery

   @celery.task(queue='high_priority')
   def high_priority_task():
       return "高優先度タスクが完了しました。"

   @celery.task(queue='low_priority')
   def low_priority_task():
       return "低優先度タスクが完了しました。"
   ```

2. **ワーカーを特定のキューに接続する**

   特定のキューを処理するワーカーを起動するには、`-Q`オプションを使用します。

   ```bash
   celery -A tasks worker --loglevel=info -Q high_priority
   ```

   このコマンドは、`high_priority`キューに送られたタスクのみを処理するワーカーを起動します。

   別のワーカーを`low_priority`キュー用に起動するには、以下のようにします。

   ```bash
   celery -A tasks worker --loglevel=info -Q low_priority
   ```

   これにより、タスクの種類ごとに処理するワーカーを分離し、優先度に応じた負荷分散を実現できます。

### 3.6 FastAPIで特定のタスクキューを使用する方法

FastAPIを使用して特定のタスクキューにタスクを送信するには、`apply_async`メソッドを使い、キュー名を指定します。これにより、タスクを適切なキューに送信し、ワーカーが効率的に処理を行えるようにします。

1. **FastAPIエンドポイントの追加**

   `main.py`に高優先度および低優先度タスクを送信するエンドポイントを追加します。
   ```python
   # main.py
   from fastapi import FastAPI
   from tasks import high_priority_task, low_priority_task

   app = FastAPI()

   @app.get("/run-high-priority-task")
   def run_high_priority_task():
       result = high_priority_task.apply_async(queue='high_priority')
       return {"task_id": result.id}

   @app.get("/run-low-priority-task")
   def run_low_priority_task():
       result = low_priority_task.apply_async(queue='low_priority')
       return {"task_id": result.id}
   ```
   これにより、エンドポイントにアクセスすることで、特定のキューにタスクを送信し、適切なワーカーで処理されるようになります。

---

## 4. 動作確認

### 4.1 アプリケーションの起動

1. **Redisサーバーの起動確認**

   UbuntuターミナルでRedisが起動していることを確認します。

   ```bash
   sudo service redis-server status
   ```

2. **FastAPIサーバーの起動**

   WindowsのコマンドプロンプトまたはPowerShellで:

   ```bash
   venv\Scripts\activate
   uvicorn main:app --reload
   ```

3. **Celeryワーカーの起動**

   別のコマンドプロンプトまたはPowerShellで:

   ```bash
   venv\Scripts\activate
   celery -A tasks worker --loglevel=info
   ```

   複数のワーカーを起動する場合は、別のターミナルを使用して同じコマンドを実行します。

4. **Celeryビートの起動（定期タスクの確認）**

   別のターミナルで以下を実行してCeleryビートを起動します。

   ```bash
   celery -A tasks beat --loglevel=info
   ```

### 4.2 タスクの実行と確認

1. **APIエンドポイントにアクセス**

   ブラウザまたはHTTPクライアントで以下のURLにアクセスします。

   - 通常のタスク: `http://localhost:8000/run-task`
   - 高優先度タスク: `http://localhost:8000/run-high-priority-task`
   - 低優先度タスク: `http://localhost:8000/run-low-priority-task`

2. **タスクIDの取得**

   レスポンスとしてタスクIDが返されます。

   ```json
   {"task_id": "タスクID"}
   ```

3. **タスク結果の確認（オプション）**

   タスクの結果を取得するには、`AsyncResult`を使用します。

   ```python
   from celery.result import AsyncResult
   from celery_app import celery

   def get_task_result(task_id):
       result = AsyncResult(task_id, app=celery)
       if result.ready():
           return result.get()
       else:
           return "タスクはまだ完了していません。"
   ```

---

## 5. まとめ

この手順で、Redis、Celery、FastAPIを組み合わせたマイクロサービスアーキテクチャを構築しました。複数のワーカープロセスを立てることで、各サーバーでタスク処理の負荷分散やスケーラビリティを実現できます。また、コンカレンシー設定との違いを理解することで、より効果的にシステムのリソースを活用できます。さらに、Celeryを使用して定期タスクを設定することで、自動的に繰り返し実行するタスクを簡単に管理できます。特定のタスクキューを使用することで、タスクの優先度や役割を分け、リソースの効率的な利用が可能になります。

---

**注意事項**

- **セキュリティ設定**: 本番環境では、RedisやAPIのセキュリティ設定を強化してください。
- **エラーハンドリング**: 例外処理やリトライ機能を実装し、信頼性を高めましょう。
- **設定のカスタマイズ**: 必要に応じて、CeleryやRedisの設定を最適化してください。

---

これで、Windows環境でRedis、Celery、FastAPIを使用したマイクロサービスアーキテクチャの構築が完了しました。

