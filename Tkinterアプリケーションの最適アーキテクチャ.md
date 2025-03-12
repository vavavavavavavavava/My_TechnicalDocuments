# Tkinterアプリケーションの最適アーキテクチャ説明ドキュメント

本ドキュメントでは、Tkinterアプリケーションの拡張性・保守性・可読性を高めるために設計されたアーキテクチャを、フォルダ構成および各層の土台部分のプログラムコードとともに詳しく解説します。各ファイルは、今後の機能拡張やチーム開発を見据え、以下の５つの基本方針に基づいて設計されています。

1. **プレゼンテーショナルコンポーネントとコンテナコンポーネントの明確な分離**  
   → UIの見た目に関する部分と、ビジネスロジックやステート更新処理を分離

2. **型安全なステート管理**  
   → Pydanticモデルを用いた`Store`によるシングルトンステート管理

3. **PubSubによるイベント駆動アーキテクチャ**  
   → `pypubsub`ライブラリを利用し、疎結合なイベント通信を実現

4. **プロセッサによるビジネスロジックの分離**  
   → 複雑な処理やビジネスロジックを各プロセッサに分割

5. **シングルトンパターンの活用**  
   → 依存関係の簡略化とグローバルステートの一元管理

以下、各層ごとにコードとその役割、そしてサブクラスで必ず上書きすべきメソッドに対して@abstractmethodを付与した内容をご紹介します。

---

## 1. フォルダ構成

```txt
my_tkinter_app/
├── core/
│   ├── __init__.py
│   ├── pubsub_base.py       # PubSubの基底クラス（抽象メソッド付き）
│   ├── store.py             # ステート管理
│   ├── models.py            # Pydanticモデル
│   └── topics.py            # トピック定義
├── processors/
│   ├── __init__.py
│   ├── root_processor.py    # 全プロセッサを管理（抽象メソッド付き）
│   ├── task_processor.py    # タスク関連処理
│   └── user_processor.py    # ユーザー関連処理
├── ui/
│   ├── __init__.py
│   ├── base/
│   │   ├── __init__.py
│   │   ├── component_base.py      # 共通基底クラス（抽象メソッド付き）
│   │   ├── container_base.py      # コンテナ基底クラス（抽象メソッド付き）
│   │   └── presentational_base.py # プレゼンテーショナル基底クラス（抽象メソッド付き）
│   ├── presentational/            # 見た目だけのコンポーネント
│   │   ├── __init__.py
│   │   ├── task_item.py
│   │   └── ...
│   └── containers/                # ロジック結合コンポーネント
│       ├── __init__.py
│       ├── task_list_container.py
│       └── ...
├── app.py                   # アプリケーションのルートクラス
└── main.py                  # エントリーポイント
```

各ディレクトリは以下の役割を担います。

- **core/**:  
  アプリケーションの中核機能（モデル定義、PubSubの仕組み、ステート管理など）を配置します。

- **processors/**:  
  ビジネスロジックを担当する各プロセッサを格納します。  
  ※`RootProcessor`の初期設定メソッドはサブクラスでの実装が必須です。

- **ui/**:  
  UIコンポーネント群。プレゼンテーショナル（表示専用）とコンテナ（ロジック連携）の2種類に分かれています。  
  各基底クラス内の初期化やデータ更新メソッドは、サブクラスで必ず実装してください。

- **app.py**:  
  Tkinterのルートウィンドウおよび画面切替え、終了処理などの基本設定を実装しています。

- **main.py**:  
  アプリケーション起動のエントリーポイントです。

---

## 2. コア機能（core/）のプログラム

### 2.1 `models.py`

Pydanticを利用して、アプリケーション全体の状態を管理する基本モデルを定義しています。

```python
from typing import Dict, List, Optional, Any
from pydantic import BaseModel, Field

class AppState(BaseModel):
    """
    アプリケーションの状態を管理するベースモデル。
    実際のアプリケーションではこの中に必要なデータモデルを追加します。
    """
    # アプリケーション固有のデータをここに追加
    ui_state: Dict[str, Any] = Field(default_factory=dict)
    
    class Config:
        arbitrary_types_allowed = True
```

### 2.2 `topics.py`

イベント発行・購読に使用するトピックを列挙型で定義し、タイプミス防止と保守性を向上させます。

```python
from enum import StrEnum, auto

class TopicEnum(StrEnum):
    """
    アプリケーションで使用するイベントトピックの定義。
    実際のアプリケーションでは必要なトピックを追加します。
    """
    # ステート変更通知の基本トピック
    STATE_CHANGED = "state.changed"
    
    # UIアクション関連の基本トピック
    UI_ACTION = "ui.action"
    
    # エラー通知など共通トピック
    ERROR = "error"
    INFO = "info"
```

### 2.3 `pubsub_base.py`

`pypubsub`ライブラリを利用し、イベントの購読・発行処理を共通化するための抽象基底クラスです。  
サブクラスでは必ず`setup_subscriptions()`を実装してください。

```python
from abc import ABC, abstractmethod
from typing import Any, Callable, List, Dict, Optional
from pypubsub import pub

class PubSubBase(ABC):
    """PubSubの管理を簡略化する基底クラス"""
    
    def __init__(self):
        self._subscriptions: List[Dict[str, Any]] = []
    
    def subscribe(self, topic: str, handler: Callable, **kwargs) -> None:
        """
        トピックの購読を登録
        
        Args:
            topic: 購読するトピック名
            handler: コールバック関数
            **kwargs: pub.subscribeに渡す追加引数
        """
        pub.subscribe(handler, topic, **kwargs)
        self._subscriptions.append({
            "topic": topic,
            "handler": handler
        })
    
    def unsubscribe(self, topic: str, handler: Callable) -> None:
        """
        特定のトピックとハンドラの購読を解除
        
        Args:
            topic: 購読解除するトピック名
            handler: 解除するコールバック関数
        """
        pub.unsubscribe(handler, topic)
        self._subscriptions = [
            sub for sub in self._subscriptions 
            if not (sub["topic"] == topic and sub["handler"] == handler)
        ]
    
    def unsubscribe_all(self) -> None:
        """すべての購読を解除"""
        for sub in self._subscriptions[:]:  # コピーでループ
            pub.unsubscribe(sub["handler"], sub["topic"])
        self._subscriptions.clear()
    
    def send_message(self, topic: str, **kwargs) -> None:
        """
        メッセージを送信
        
        Args:
            topic: トピック名
            **kwargs: メッセージデータ
        """
        pub.sendMessage(topic, **kwargs)
    
    @abstractmethod
    def setup_subscriptions(self) -> None:
        """サブクラスでオーバーライドして購読設定を行う"""
        pass
    
    def teardown(self) -> None:
        """リソース解放"""
        self.unsubscribe_all()
```

### 2.4 `store.py`

アプリケーションの状態を管理するシングルトンの`Store`クラスです。  
型安全なステート更新やリスト操作を実装し、更新時にはPubSubで通知を行います。

```python
from typing import Dict, List, Optional, Any, TypeVar, Type, Union
from core.models import AppState
from core.topics import TopicEnum
from pypubsub import pub

T = TypeVar('T')

class Store:
    """型安全なシングルトンStore"""
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super(Store, cls).__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    def __init__(self):
        if self._initialized:
            return
        self._state = AppState()
        self._initialized = True

    @property
    def state(self) -> AppState:
        """型ヒントを保持したまま現在のステートを返す"""
        return self._state
    
    def update_state(self, attr_path: Any, new_value: Any) -> None:
        """
        型安全なステート更新
        
        使用例:
        Store().update_state(Store().state.user, new_user_obj)
        """
        # 属性パスから変更トピックを生成
        path_info = self._resolve_attribute_path(attr_path)
        if not path_info:
            raise ValueError(f"Invalid attribute path: {attr_path}")
        
        attribute_str = path_info["path"]
        
        # 古い値を保存
        old_value = self._get_value_by_path(attribute_str)
        
        # 値の検証
        if hasattr(new_value, "model_validate"):
            # Pydanticオブジェクトの場合はそのまま使用
            validated_value = new_value
        elif path_info["type"] and hasattr(path_info["type"], "model_validate"):
            # パスの型情報が利用可能な場合は検証
            try:
                validated_value = path_info["type"].model_validate(new_value)
            except Exception as e:
                raise ValueError(f"Invalid value for {attribute_str}: {e}")
        else:
            # 型情報がない場合はそのまま使用
            validated_value = new_value
        
        # ステート更新
        self._set_value_by_path(attribute_str, validated_value)
        
        # 通知トピックを生成
        notification_topic = f"{TopicEnum.STATE_CHANGED}.{attribute_str}"
        
        # 変更通知
        pub.sendMessage(notification_topic, 
                       old_value=old_value,
                       new_value=validated_value)
        
        # 基本通知も送信
        pub.sendMessage(str(TopicEnum.STATE_CHANGED), 
                       path=attribute_str,
                       old_value=old_value,
                       new_value=validated_value)
    
    def add_to_list(self, list_attr: List[T], item: T) -> None:
        """
        型安全なリスト追加操作
        
        使用例:
        Store().add_to_list(Store().state.tasks, new_task)
        """
        # 属性パスを取得
        path_info = self._resolve_attribute_path(list_attr)
        if not path_info:
            raise ValueError(f"Invalid list attribute: {list_attr}")
        
        attribute_str = path_info["path"]
        item_type = path_info.get("item_type")
        
        # 値の検証
        if item_type and hasattr(item_type, "model_validate") and not isinstance(item, item_type):
            try:
                validated_item = item_type.model_validate(item)
            except Exception as e:
                raise ValueError(f"Invalid item for {attribute_str}: {e}")
        else:
            validated_item = item
        
        # 現在のリストを取得
        current_list = self._get_value_by_path(attribute_str)
        if not isinstance(current_list, list):
            current_list = []
        
        # 新しいリストを作成して項目を追加
        new_list = current_list.copy()
        new_list.append(validated_item)
        
        # ステート更新
        self._set_value_by_path(attribute_str, new_list)
        
        # 基本の変更通知
        changed_topic = f"{TopicEnum.STATE_CHANGED}.{attribute_str}"
        pub.sendMessage(changed_topic, 
                       old_value=current_list,
                       new_value=new_list)
        
        # 追加専用の通知
        added_topic = f"{TopicEnum.STATE_CHANGED}.{attribute_str}.added"
        pub.sendMessage(added_topic, 
                       item=validated_item, 
                       index=len(new_list)-1)
    
    def _resolve_attribute_path(self, attr_ref: Any) -> Optional[Dict[str, Any]]:
        """
        属性参照からパス情報を解決
        
        返り値:
        {
            "path": "属性パス文字列",
            "type": 属性の型,
            "item_type": リスト内の項目の型（リストの場合）
        }
        """
        # これは単純化された実装です
        # 実際にはより複雑なリフレクションが必要かもしれません
        
        # AppStateの各属性をチェック
        state_dict = self._state.model_dump()
        model_fields = self._state.model_fields
        
        for key, field_info in model_fields.items():
            if getattr(self._state, key) is attr_ref:
                # 型情報を取得
                field_type = field_info.annotation
                item_type = None
                
                # リストの場合、項目の型を取得
                if hasattr(field_type, "__origin__") and field_type.__origin__ is list:
                    if hasattr(field_type, "__args__") and field_type.__args__:
                        item_type = field_type.__args__[0]
                
                return {
                    "path": key,
                    "type": field_type,
                    "item_type": item_type
                }
        
        # ネストされた属性も検索（簡略化）
        return None
    
    def _get_value_by_path(self, path: str) -> Any:
        """パスを指定して値を取得"""
        parts = path.split('.')
        current = self._state.model_dump()
        
        for part in parts:
            if isinstance(current, dict) and part in current:
                current = current[part]
            else:
                return None
        
        return current
    
    def _set_value_by_path(self, path: str, value: Any) -> None:
        """パスを指定して値を設定"""
        parts = path.split('.')
        
        # ステートの複製を作成
        state_dict = self._state.model_dump()
        
        # ネストされた辞書を更新
        current = state_dict
        for i, part in enumerate(parts[:-1]):
            if part not in current:
                current[part] = {}
            current = current[part]
        
        # 最後の部分を更新
        current[parts[-1]] = value if not hasattr(value, "model_dump") else value.model_dump()
        
        # 新しいステートオブジェクトを作成
        self._state = AppState.model_validate(state_dict)
```

---

## 3. ビジネスロジック層（processors/）

### 3.1 `root_processor.py`

全プロセッサを一元管理する中央クラスです。  
初期設定のための`setup()`はサブクラスでの実装が必須となるため、@abstractmethodを付与しています。

```python
from abc import abstractmethod
from core.pubsub_base import PubSubBase

class RootProcessor(PubSubBase):
    """全プロセッサを管理する中央クラス"""
    
    def __init__(self):
        super().__init__()
        # サブプロセッサを初期化
        self.processors = {}
        # 初期設定
        self.setup()
    
    @abstractmethod
    def setup(self):
        """初期設定 - サブクラスでオーバーライド"""
        pass
    
    def register_processor(self, name: str, processor: PubSubBase):
        """プロセッサを登録"""
        self.processors[name] = processor
    
    def teardown(self):
        """全プロセッサのリソース解放"""
        for processor in self.processors.values():
            processor.teardown()
        super().teardown()
```

---

## 4. UI層（ui/）のプログラム

UI層は、表示専用のプレゼンテーショナルコンポーネントと、ロジック・ステート連携を担うコンテナコンポーネントに分かれています。

### 4.1 `ui/base/component_base.py`

Tkinterの`Frame`を継承し、全UIコンポーネントの共通初期化処理を行います。  
UIの初期化は各サブクラスで実装すべきため、`setup_ui()`に@abstractmethodを付与しています。

```python
from abc import ABC, abstractmethod
import tkinter as tk
from typing import Any, Dict

class ComponentBase(tk.Frame, ABC):
    """全UIコンポーネントの共通基底クラス"""
    
    def __init__(self, parent, **kwargs):
        super().__init__(parent, **kwargs)
        self.parent = parent
        self.setup_ui()
    
    @abstractmethod
    def setup_ui(self):
        """UIコンポーネントを初期化 - サブクラスでオーバーライド"""
        pass
    
    def destroy(self):
        """リソース解放"""
        super().destroy()
```

### 4.2 `ui/base/presentational_base.py`

Tkinterに依存するプレゼンテーショナルコンポーネントの基底クラスです。  
表示データの更新用の`update_data()`は必ずサブクラスで実装してください。

```python
from abc import abstractmethod
from typing import Any, Callable, Dict
from ui.base.component_base import ComponentBase

class PresentationalComponent(ComponentBase):
    """Tkinterのみに依存するプレゼンテーショナルコンポーネント"""
    
    def __init__(self, parent, **kwargs):
        self._handlers = {}
        super().__init__(parent, **kwargs)
    
    def register_handler(self, event_name: str, handler: Callable) -> None:
        """イベントハンドラーを登録"""
        self._handlers[event_name] = handler
    
    def trigger_event(self, event_name: str, **event_data) -> None:
        """イベントを発火"""
        if event_name in self._handlers:
            self._handlers[event_name](**event_data)
    
    @abstractmethod
    def update_data(self, data: Dict[str, Any]) -> None:
        """表示データを更新 - サブクラスでオーバーライド"""
        pass
```

### 4.3 `ui/base/container_base.py`

ロジックとステート管理を行うコンテナコンポーネントの基底クラスです。  
ステート変更の購読設定(`setup_subscriptions()`)とUI再描画用の`refresh_from_state()`は必ずサブクラスで実装してください。

```python
from abc import abstractmethod
from ui.base.component_base import ComponentBase
from core.pubsub_base import PubSubBase
from core.store import Store

class ContainerComponent(ComponentBase, PubSubBase):
    """ロジックとステートを処理するコンテナコンポーネント"""
    
    def __init__(self, parent, **kwargs):
        ComponentBase.__init__(self, parent, **kwargs)
        PubSubBase.__init__(self)
        self.store = Store()
        self.setup_subscriptions()
    
    @abstractmethod
    def setup_subscriptions(self):
        """ステート変更の購読を設定 - サブクラスでオーバーライド"""
        pass
    
    @abstractmethod
    def refresh_from_state(self):
        """ステートから表示を更新 - サブクラスでオーバーライド"""
        pass
    
    def destroy(self):
        """リソース解放"""
        self.teardown()  # PubSubBaseのメソッド
        ComponentBase.destroy(self)
```

---

## 5. アプリケーションのエントリーポイント

### 5.1 `app.py`

アプリケーション全体のルートウィンドウを表現するクラスです。  
ウィンドウの基本設定、シングルトンの`Store`、および`RootProcessor`の初期化、画面切替え処理などを実装しています。

```python
import tkinter as tk
from processors.root_processor import RootProcessor
from core.store import Store

class Application(tk.Tk):
    """アプリケーションのルートクラス"""
    
    def __init__(self):
        super().__init__()
        
        # ウィンドウの基本設定
        self.title("Tkinter Application")
        self.geometry("800x600")
        
        # シングルトンストアの初期化
        self.store = Store()
        
        # ルートプロセッサの初期化（※具体的なサブクラスを実装する必要があります）
        self.root_processor = RootProcessor()
        
        # メインコンテナの設定
        self.main_frame = tk.Frame(self)
        self.main_frame.pack(fill=tk.BOTH, expand=True)
        
        # 現在アクティブなコンテナ
        self.active_container = None
    
    def switch_container(self, container_class, *args, **kwargs):
        """
        画面を切り替える
        
        Args:
            container_class: 表示するコンテナコンポーネントのクラス
            *args, **kwargs: コンテナクラスのコンストラクタに渡す引数
        """
        # 現在のコンテナを破棄
        if self.active_container:
            self.active_container.destroy()
        
        # 新しいコンテナを作成
        self.active_container = container_class(self.main_frame, *args, **kwargs)
        self.active_container.pack(fill=tk.BOTH, expand=True)
    
    def run(self):
        """アプリケーションを実行"""
        # 最初のコンテナを表示
        # self.switch_container(HomeContainer)
        
        # メインループを開始
        self.mainloop()
    
    def on_closing(self):
        """アプリケーション終了時の処理"""
        # プロセッサのリソースを解放
        self.root_processor.teardown()
        
        # アプリケーションを終了
        self.destroy()
```

### 5.2 `main.py`

アプリケーション起動のエントリーポイントです。

```python
from app import Application

if __name__ == "__main__":
    app = Application()
    app.protocol("WM_DELETE_WINDOW", app.on_closing)
    app.run()
```

---

## 6. このアーキテクチャの採用によるメリット

- **UIとビジネスロジックの明確な分離**  
  プレゼンテーショナルコンポーネント（表示専用）とコンテナコンポーネント（ロジックやステート更新）の役割を分けることで、変更の影響範囲を限定し保守性を向上させます。

- **型安全なステート管理**  
  Pydanticを利用した`AppState`とシングルトン`Store`により、データの整合性と安全な更新が可能です。

- **PubSubによるイベント駆動**  
  `pypubsub`を利用して、各コンポーネント間の疎結合な通信を実現し、機能追加や変更が柔軟に行えます。

- **プロセッサによるビジネスロジックの集中管理**  
  `RootProcessor`を通じたプロセッサの一元管理により、初期化やリソース解放の処理がシンプルになり、全体の制御が容易です。

- **シングルトンパターンの活用**  
  グローバルに一貫した状態管理が可能となり、どのコンポーネントからでも同じステートにアクセスでき、依存関係が簡略化されます。

---

## 7. アプリ実装例

### 7.1 ホーム画面コンテナの実装  

ユーザーは、**ContainerComponent** を継承してホーム画面用のコンテナを実装します。  
以下の例では、ステートからテキストを取得して、ホーム画面の表示を更新する処理を実装しています。

**ファイル: `ui/containers/home_container.py`**

```python
from ui.base.container_base import ContainerComponent
from ui.presentational.home_view import HomeView

class HomeContainer(ContainerComponent):
    def setup_subscriptions(self):
        # 例として、"ui_state" 以下の変更通知を購読
        self.subscribe("state.changed.ui_state", self.on_state_changed)

    def refresh_from_state(self):
        # ストアから現在の状態を取得し、プレゼンテーショナルコンポーネントに反映
        current_text = self.store.state.ui_state.get("home_text", "Default Home Text")
        self.home_view.update_data({"text": current_text})

    def on_state_changed(self, **kwargs):
        # ステート変更時のハンドラ。画面更新を行う。
        self.refresh_from_state()

    def setup_ui(self):
        # ホーム画面用のプレゼンテーショナルコンポーネントを作成して配置
        self.home_view = HomeView(self)
        self.home_view.pack(fill="both", expand=True)
```

### 7.2 ホーム画面用プレゼンテーショナルコンポーネントの実装  

次に、**PresentationalComponent** を継承して、実際に画面に表示するコンポーネント（ここではシンプルなラベル）を実装します。

**ファイル: `ui/presentational/home_view.py`**

```python
import tkinter as tk
from ui.base.presentational_base import PresentationalComponent

class HomeView(PresentationalComponent):
    def setup_ui(self):
        # シンプルなラベルを作成し、初期テキストを表示
        self.label = tk.Label(self, text="Initial Home Text")
        self.label.pack(padx=10, pady=10)

    def update_data(self, data: dict):
        # データ更新時にラベルのテキストを変更
        new_text = data.get("text", "Default Text")
        self.label.config(text=new_text)
```

### 7.3 ルートプロセッサの実装例  

ビジネスロジックの統括として、**RootProcessor** を継承したシンプルなルートプロセッサの実装例です。  
この例では、追加のプロセッサ登録は行わず、セットアップ完了のメッセージを出力しています。

**ファイル: `processors/my_root_processor.py`**

```python
from processors.root_processor import RootProcessor

class MyRootProcessor(RootProcessor):
    def setup(self):
        # この例では、追加プロセッサの登録は行わずにセットアップ完了を示す
        print("MyRootProcessor setup complete.")
```

### 7.4 アプリケーション起動モジュールの実装例  

最後に、上記で実装したホーム画面コンテナを利用する例を示します。  
このモジュールは、アプリケーション起動時にホーム画面コンテナに切り替え、状態を更新することで動作例を確認できます。

**ファイル: `example_usage.py`**

```python
from app import Application
from ui.containers.home_container import HomeContainer
from core.store import Store

def run_example():
    app = Application()
    
    # ホーム画面コンテナに切り替え
    app.switch_container(HomeContainer)
    
    # 例として、Storeの状態を更新しホーム画面に反映
    store = Store()
    store.update_state(store.state.ui_state, {"home_text": "Welcome to Home!"})
    
    app.protocol("WM_DELETE_WINDOW", app.on_closing)
    app.run()

if __name__ == "__main__":
    run_example()
```

---

## 8. まとめ

本アーキテクチャは、**UI構造の整理**、**型安全なステート管理**、**イベント駆動の実現**、**ビジネスロジックの一元管理**、および**シングルトンパターンの活用**といった特徴を備えています。  
各基底クラスにおいて、サブクラスで必ず実装すべきメソッドには `@abstractmethod` を付与しており、実装漏れを防止するとともに設計意図を明示しています。  
今回提示したホーム画面コンテナ（`HomeContainer`）、プレゼンテーショナルコンポーネント（`HomeView`）、およびルートプロセッサ（`MyRootProcessor`）の実装例は、ユーザーがこのアーキテクチャを用いてアプリケーションをどのように構築すればよいかの一例となります。  
これらを参考に、具体的な機能を実装・拡張することで、保守性・拡張性の高いTkinterアプリケーションを構築してください。
