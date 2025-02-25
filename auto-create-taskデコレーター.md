以下は、グローバルなイベントループ（MAIN_LOOP）を採用した場合のデコレーターの紹介文書の例です。  
このドキュメントでは、デコレーターの機能、魅力、こだわり、そしてなぜグローバルなイベントループを利用する設計になっているのかを説明しています。

---

```markdown
# auto_create_task デコレーター紹介

## 概要

`auto_create_task` デコレーターは、同期関数と非同期関数の双方を統一的に扱い、用途に合わせた実行方式（ブロッキング実行と非ブロッキング実行）を提供する高機能なデコレーターです。  
特に、pypubsub などのイベント駆動型アプリケーションにおいて、関数のシグネチャやキーワード引数をそのまま伝播しながら、非同期処理をシームレスに統合できる点が魅力です。

## 主な機能

- **同期/非同期関数の統一サポート**  
  - **非同期関数の場合**  
    - `wait=False`（デフォルト）: 既に実行中のイベントループ上でタスクをスケジュールし、タスクオブジェクトを返します。  
      ※このモードでは、実行時に「動作中のイベントループ」が必須です。  
    - `wait=True`: イベントループが動作していない状態であれば `asyncio.run()` を用いてブロッキング実行し、結果を返します。  
      ※すでにループが動作中の場合はエラーとなります。

  - **同期関数の場合**  
    - `wait=False`: `run_in_executor` を利用して、別スレッドで関数を実行し Future／Task を返します。  
    - `wait=True`: イベントループが動作していない状態なら、`asyncio.run()` によりブロッキング実行し結果を返します。

- **キーワード引数とシグネチャの保持**  
  `functools.partial` および `functools.wraps` を利用することで、元の関数のシグネチャやキーワード引数を正確に保持し、pypubsub など外部ライブラリと連携する際にも情報が失われません。

## グローバルイベントループ（MAIN_LOOP）の採用理由

- **非待機モード（wait=False）での実行要件**  
  非待機モードでは、既に実行中のイベントループが存在していることを前提としています。  
  単に「新しいループを作成して set_event_loop()」するだけでは、そのループは自動的に動作状態（`run_forever()` や `run_until_complete()` で回される状態）にならないため、タスクがスケジュールされても実際に実行される保証がありません。

- **グローバルなフォールバックのメリット**  
  グローバル変数（MAIN_LOOP）としてイベントループを管理することで、  
  - 必要なときに一元的にループを起動し、各タスクを確実に実行できる環境を提供  
  - 複数箇所からの呼び出しに対して、同一のイベントループを共有できる  
  というメリットがあります。

> **注意:**  
> 既にイベントループが適切に管理されているアプリケーションでは、グローバルな MAIN_LOOP は必須ではありません。  
> しかし、イベントループの存在が保証できない環境（例: pypubsub の同期コールバックから非同期処理を発火する場合）では、グローバルなフォールバックは便利な設計となります。

## 実装例

以下は、グローバル変数 MAIN_LOOP を用いたデコレーターの実装例です。

```python
import asyncio
import functools

# グローバルな待機用イベントループ（フォールバックとして利用）
MAIN_LOOP = None

def auto_create_task(*, wait=False, executor=None):
    """
    同期関数・非同期関数の両方に対応するデコレーター

    [非同期関数の場合]
      - wait=False: 既存の実行中イベントループ上でタスク化して返す
      - wait=True : イベントループが動作していなければ asyncio.run() で同期実行し結果を返す

    [同期関数の場合]
      - wait=False: run_in_executor() により非同期実行（Future/Task を返す）
      - wait=True : イベントループが動作していなければ asyncio.run() でブロッキング実行し結果を返す
    """
    def decorator(func):
        is_coro = asyncio.iscoroutinefunction(func)
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            if is_coro:
                # 非同期関数の場合
                coro = func(*args, **kwargs)
                if wait:
                    try:
                        # 既に実行中のイベントループがある場合はブロッキング実行できない
                        loop = asyncio.get_running_loop()
                        if loop.is_running():
                            raise RuntimeError("wait=True は実行中のイベントループ内では利用できません。直接 await してください。")
                    except RuntimeError:
                        return asyncio.run(coro)
                else:
                    try:
                        loop = asyncio.get_running_loop()
                    except RuntimeError:
                        # MAIN_LOOP をフォールバックとして利用
                        global MAIN_LOOP
                        if MAIN_LOOP is None:
                            MAIN_LOOP = asyncio.new_event_loop()
                            asyncio.set_event_loop(MAIN_LOOP)
                        loop = MAIN_LOOP
                    return loop.create_task(coro)
            else:
                # 同期関数の場合
                call_func = functools.partial(func, *args, **kwargs)
                try:
                    loop = asyncio.get_running_loop()
                except RuntimeError:
                    global MAIN_LOOP
                    if MAIN_LOOP is None:
                        MAIN_LOOP = asyncio.new_event_loop()
                        asyncio.set_event_loop(MAIN_LOOP)
                    loop = MAIN_LOOP
                future = loop.run_in_executor(executor, call_func)
                if wait:
                    try:
                        loop = asyncio.get_running_loop()
                        if loop.is_running():
                            raise RuntimeError("wait=True は実行中のイベントループ内では利用できません。直接 await してください。")
                    except RuntimeError:
                        async def _await_future():
                            return await future
                        return asyncio.run(_await_future())
                else:
                    return asyncio.ensure_future(future)
        return wrapper
    return decorator
```

## 利用例（pypubsub との連携）

以下は、pypubsub のリスナーとして、このデコレーターを利用した例です。

```python
from pubsub import pub

# 同期リスナー：wait=True によりブロッキング実行し、結果を返す
@auto_create_task(wait=True)
def sync_listener(data):
    print("sync_listener started with:", data)
    import time
    time.sleep(1)
    print("sync_listener finished with:", data)
    return f"sync result for {data}"

# 非同期リスナー：wait=False によりタスクオブジェクトを返す（後で実行される）
@auto_create_task()
async def async_listener(data):
    print("async_listener started with:", data)
    await asyncio.sleep(1)
    print("async_listener finished with:", data)
    return f"async result for {data}"

def main():
    # pypubsub のリスナーに登録
    pub.subscribe(sync_listener, 'sync_event')
    pub.subscribe(async_listener, 'async_event')
    
    # 同期リスナーはブロッキング実行され、結果が即座に返る
    result_sync = pub.sendMessage('sync_event', data="Test Sync")
    print("sync_event result:", result_sync)
    
    # 非同期リスナーはタスクオブジェクトを返し、後でイベントループで実行される
    task_async = pub.sendMessage('async_event', data="Test Async")
    print("async_event returned task:", task_async)
    
    # 既存のイベントループでタスクの実行を促す（例: asyncio.run() 内で処理）
    async def runner():
        await asyncio.sleep(2)
    asyncio.run(runner())

if __name__ == '__main__':
    main()
```

## まとめ

この `auto_create_task` デコレーターは以下の点で魅力的です。

- **多用途性:**  
  同期関数と非同期関数の双方を一つのデコレーターで扱え、実行方式を `wait` パラメーターで柔軟に選択できます。

- **シグネチャ保持:**  
  `functools.wraps` と `functools.partial` により、元の関数のシグネチャやキーワード引数を損なわずに処理できます。

- **グローバルイベントループの合理的な採用:**  
  非待機モード（wait=False）では、既に動作中のイベントループが必要です。  
  単に新しいループを作成して `set_event_loop()` するだけでは、そのループは自動的に実行状態にならないため、タスクが実行されないリスクがあります。  
  そのため、必要に応じたフォールバックとしてグローバルな MAIN_LOOP を利用し、タスクの確実な実行を保証しています。

このデコレーターは、非同期処理と同期処理をシームレスに統合したいシナリオ、特にイベント駆動型のアプリケーション（例: pypubsub のリスナー）に最適な設計となっています。
```

---

このような設計思想により、`auto_create_task` デコレーターは実行環境に合わせた柔軟な動作を提供し、グローバルイベントループを適切に管理することで、タスクの確実な実行を実現しています。
