# auto_create_task デコレーターの紹介

このドキュメントでは、**auto_create_task** デコレーターについて説明します。このデコレーターは、tkinter の `mainloop` と asyncio のイベントループが共存する環境で、非同期タスクの実行を容易にし、UI の応答性を維持しながら「同期的な」結果取得を実現するために設計されています。

---

## 概要

`auto_create_task` は、関数（同期関数および非同期関数）の実行を自動的に非同期タスクとして登録するためのデコレーターです。  
主な特徴は以下の通りです：

- **wait=False モード**  
  - 非同期関数の場合は `loop.create_task()` を用いてタスク化し、Future を返します。  
  - 同期関数の場合は `run_in_executor()` によりバックグラウンドで実行し、その Future を返します。

- **wait=True モード**  
  - 内部でタスクを生成し、そのタスクの結果を `await` する async ラッパーを返します。  
  - 非同期関数および同期関数の両方に対応しており、await 中は他のタスク（例えば UI 更新処理）に制御が戻るため、UI フリーズを防止できます。  
  - このモードでは、呼び出し側は async コンテキスト内で `await` により結果を取得する必要があります。

---

## 実装例

以下は、wait オプションに応じた動作を実現する完成版のデコレーターのコードです。

```python
import asyncio
import functools

def auto_create_task(*, wait=False, executor=None):
    def decorator(func):
        is_coro = asyncio.iscoroutinefunction(func)
        if wait:
            # wait=True の場合、内部でタスクを生成し await して結果を返す async ラッパー
            @functools.wraps(func)
            async def wrapper(*args, **kwargs):
                try:
                    loop = asyncio.get_running_loop()
                except RuntimeError:
                    raise RuntimeError("実行中のイベントループが必要です。")
                
                if is_coro:
                    task = loop.create_task(func(*args, **kwargs))
                    return await task
                else:
                    call_func = functools.partial(func, *args, **kwargs)
                    future = loop.run_in_executor(executor, call_func)
                    return await future
            return wrapper
        else:
            # wait=False の場合、単にタスク（または Future）を返す
            @functools.wraps(func)
            def wrapper(*args, **kwargs):
                try:
                    loop = asyncio.get_running_loop()
                except RuntimeError:
                    raise RuntimeError("実行中のイベントループが必要です。")
                
                if is_coro:
                    return loop.create_task(func(*args, **kwargs))
                else:
                    call_func = functools.partial(func, *args, **kwargs)
                    future = loop.run_in_executor(executor, call_func)
                    return asyncio.ensure_future(future)
            return wrapper
    return decorator
```

---

## 特徴と利点

### UI の応答性の確保

- **非同期タスクの await**  
  wait=True モードでは、内部でタスクを作成し、そのタスクを await することで、結果取得のためにイベントループをブロックせずに待機できます。  
  この await 中は、他のタスク（例えば UI 更新処理）が実行されるため、UI フリーズの発生を抑えることが可能です。

### 同期関数も非同期に実行

- **バックグラウンド処理**  
  同期関数も `run_in_executor()` を使って非同期に実行されるため、重い処理をUIスレッドから分離して実行できます。  
  これにより、UI のスムーズな動作が維持されます。

### 柔軟な呼び出し方法

- **wait=False モード**  
  タスク（Future）を返すため、呼び出し側で後から `await` したり、コールバックを設定したりすることで、非同期処理の結果をハンドリングできます。

- **wait=True モード**  
  呼び出し側が async コンテキスト内で `await` することで、同期的な結果取得が可能です。  
  この場合も、await 中は他のイベントが処理されるため、UI フリーズを回避できます。

---

## 利用例

### 非同期関数の場合 (wait=True)

```python
@auto_create_task(wait=True)
async def async_task(x, y):
    await asyncio.sleep(1)
    return x + y

# 呼び出し側（async コンテキスト内）
async def main():
    result = await async_task(3, 4)
    print("Result:", result)

asyncio.run(main())
```

### 同期関数の場合 (wait=True)

```python
@auto_create_task(wait=True)
def sync_task(x, y):
    import time
    time.sleep(1)
    return x * y

# 呼び出し側（async コンテキスト内）
async def main():
    result = await sync_task(3, 4)
    print("Result:", result)

asyncio.run(main())
```

### wait=False モードの場合

```python
@auto_create_task(wait=False)
async def async_task_no_wait(x, y):
    await asyncio.sleep(1)
    return x - y

# 呼び出し側（async コンテキスト内）
async def main():
    task = async_task_no_wait(10, 4)
    # 他の処理を実行中に、後で結果を await することが可能
    result = await task
    print("Result:", result)

asyncio.run(main())
```

---

## 結論

`auto_create_task` デコレーターは、tkinter の `mainloop` と asyncio のイベントループが共存する環境下で、非同期タスクを簡単に管理し、UI の応答性を確保しながら「同期的」な結果取得を実現するための有用なツールです。  
- **wait=True** モードでは、内部でタスクを生成し await することで、ブロッキングせずに結果を取得可能です。  
- **wait=False** モードでは、Future を返すことで、柔軟な非同期処理のハンドリングが可能です。