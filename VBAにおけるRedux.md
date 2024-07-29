# Redux on VBA

## サンプルコード

```vba
' アクションタイプモジュール (ActionTypes.bas)
Public Enum ActionType
    INCREMENT = 0
    DECREMENT = 1
    RESET = 2
    ADD = 3
    NOOP = 4
    ASYNC_INCREMENT_START = 5
    ASYNC_INCREMENT_COMPLETE = 6
    THUNK = 7
End Enum

' アクションモジュール (Actions.bas)
Public Type Action
    Type As ActionType
    Payload As Variant
End Type

' ストアモジュール (Store.bas)
Public Type State
    Count As Integer
    IsLoading As Boolean
    PreviousActionType As ActionType
End Type

Private CurrentState As State
Private Middlewares As Collection
Private Listeners As Collection

Public Function GetState() As State
    GetState = CurrentState
End Function

Public Sub AddMiddleware(ByVal Middleware As String)
    If Middlewares Is Nothing Then
        Set Middlewares = New Collection
    End If
    Middlewares.Add Middleware
End Sub

Public Sub AddListener(ByVal Listener As Object)
    If Listeners Is Nothing Then
        Set Listeners = New Collection
    End If
    Listeners.Add Listener
End Sub

Public Sub Dispatch(ByVal Action As Action)
    Dim FinalAction As Action
    FinalAction = ApplyMiddlewares(Action)
    
    CurrentState = Reducer(CurrentState, FinalAction)
    
    NotifyListeners
End Sub

Private Function ApplyMiddlewares(ByVal Action As Action) As Action
    Dim Middleware As Variant
    Dim Result As Action
    Result = Action
    
    If Not Middlewares Is Nothing Then
        For Each Middleware In Middlewares
            Result = Application.Run(Middleware, Result)
        Next Middleware
    End If
    
    ApplyMiddlewares = Result
End Function

Private Sub NotifyListeners()
    Dim Listener As Variant
    
    If Not Listeners Is Nothing Then
        For Each Listener In Listeners
            Listener.OnStateChange CurrentState
        Next Listener
    End If
    
    ' 標準モジュールの処理をThunkとして実行
    Dispatch ThunkAction("ModuleListenerThunk")
End Sub

Public Function Reducer(ByVal State As State, ByVal Action As Action) As State
    Dim NewState As State
    NewState = State
    
    ' アクションタイプを記録
    NewState.PreviousActionType = Action.Type
    
    Select Case Action.Type
        Case ActionType.INCREMENT
            NewState.Count = State.Count + 1
        Case ActionType.DECREMENT
            NewState.Count = State.Count - 1
        Case ActionType.RESET
            NewState.Count = 0
        Case ActionType.ADD
            If Not IsEmpty(Action.Payload) Then
                NewState.Count = State.Count + CLng(Action.Payload)
            End If
        Case ActionType.ASYNC_INCREMENT_START
            NewState.IsLoading = True
        Case ActionType.ASYNC_INCREMENT_COMPLETE
            NewState.Count = State.Count + 1
            NewState.IsLoading = False
    End Select
    
    Reducer = NewState
End Function

' クラスモジュール (ClassListener.cls)
Option Explicit

Public Sub OnStateChange(ByVal NewState As State)
    Select Case NewState.PreviousActionType
        Case ActionType.INCREMENT, ActionType.DECREMENT, ActionType.ADD
            Debug.Print "ClassListener: Count changed to " & NewState.Count
        Case ActionType.RESET
            Debug.Print "ClassListener: Count was reset to " & NewState.Count
        Case ActionType.ASYNC_INCREMENT_START
            Debug.Print "ClassListener: Async increment started"
        Case ActionType.ASYNC_INCREMENT_COMPLETE
            Debug.Print "ClassListener: Async increment completed, new count is " & NewState.Count
    End Select
End Sub

' ユーザーフォーム (CounterForm.frm)
Option Explicit

Private Sub UserForm_Initialize()
    AddListener Me
    UpdateUI
End Sub

Private Sub IncrementButton_Click()
    Dispatch IncrementAction()
End Sub

Private Sub DecrementButton_Click()
    Dispatch DecrementAction()
End Sub

Private Sub ResetButton_Click()
    Dispatch ResetAction()
End Sub

Private Sub AddButton_Click()
    Dispatch AddAction(10)
End Sub

Private Sub AsyncIncrementButton_Click()
    Dispatch AsyncIncrementAction()
End Sub

Private Sub ComplexOperationButton_Click()
    Dispatch ThunkAction("ComplexOperation")
End Sub

Public Sub OnStateChange(ByVal NewState As State)
    UpdateUI
    
    ' 前回のアクションタイプに基づいて追加の処理を行う
    Select Case NewState.PreviousActionType
        Case ActionType.RESET
            MsgBox "Counter has been reset!", vbInformation
        Case ActionType.ASYNC_INCREMENT_COMPLETE
            MsgBox "Async increment completed!", vbInformation
    End Select
End Sub

Private Sub UpdateUI()
    CountLabel.Caption = "Count: " & GetState().Count
    If GetState().IsLoading Then
        StatusLabel.Caption = "Loading..."
    Else
        StatusLabel.Caption = "Ready"
    End If
End Sub

' 標準モジュール (Main.bas)
Sub Main()
    ' ミドルウェアを追加
    AddMiddleware "LoggingMiddleware"
    AddMiddleware "ValidationMiddleware"
    AddMiddleware "AsyncMiddleware"
    AddMiddleware "ThunkMiddleware"
    
    ' リスナーを追加
    Dim ClassListener As New ClassListener
    AddListener ClassListener
    
    ' 初期状態を設定
    Dispatch ResetAction()
    
    ' フォームを表示
    CounterForm.Show
End Sub

' 標準モジュールの処理をThunkとして実行する例
Public Sub ModuleListenerThunk()
    Dim State As State
    State = GetState()
    
    Select Case State.PreviousActionType
        Case ActionType.INCREMENT
            Debug.Print "ModuleListenerThunk: Increment occurred, new count is " & State.Count
        Case ActionType.DECREMENT
            Debug.Print "ModuleListenerThunk: Decrement occurred, new count is " & State.Count
        Case ActionType.RESET
            Debug.Print "ModuleListenerThunk: Counter was reset"
        Case ActionType.ADD
            Debug.Print "ModuleListenerThunk: Value was added, new count is " & State.Count
        Case ActionType.ASYNC_INCREMENT_START
            Debug.Print "ModuleListenerThunk: Async increment started"
        Case ActionType.ASYNC_INCREMENT_COMPLETE
            Debug.Print "ModuleListenerThunk: Async increment completed, new count is " & State.Count
    End Select
End Sub
```

## Reduxという概念

### 登場人物

1. Action (操作や事象): ユーザーの操作やシステムイベントなど、アプリケーション内で発生する出来事を表します。

2. Store (データの管理所): アプリケーション全体のデータ（State）を一元管理し、データの変更を制御します。

3. Middleware Chain (前処理の連鎖): Actionが状態更新関数に渡される前に、ログ記録や非同期処理などの追加処理を行う仕組みです。

4. Reducer (状態更新関数): 現在の状態とActionを基に、新しい状態を生成する純粋な関数です。直接状態を変更せず、新しい状態オブジェクトを返します。

5. State (アプリの状態): アプリケーション全体の現在の状態を表すデータ構造です。

6. Listeners (変更監視役): 状態の変更を監視し、変更があった場合に通知を受け取るオブジェクトです。

7. UI / UserForm (画面表示): ユーザーに情報を表示し、操作を受け付けるインターフェースです。状態の変更に応じて自動的に更新されます。

### Reduxの処理フロー

```mermaid
graph TD
    A["Action 
    (操作や事象)"] -->|"Dispatched to 
    (送信)"| S["Store 
    (データの管理所)"]
    S -->|"Passes action to 
    (前処理へ渡す)"| M["Middleware Chain 
    (前処理の連鎖)"]
    M -->|"Modified action 
    (加工された操作)"| R["Reducer 
    (状態更新関数)"]
    R -->|"Updates 
    (更新)"| ST["State 
    (アプリの状態)"]
    ST -->|"Notifies 
    (変更を通知)"| L["Listeners 
    (変更監視役)"]
    L -->|"Update UI 
    (画面更新)"| U["UI / UserForm 
    (画面表示)"]
    U -->|"Triggers 
    (発生源)"| A
    S -.->|"GetState 
    (状態取得)"| U

classDef defaultStyle fill:#f9f,stroke:#333,stroke-width:2px;
class A,S,M,R,ST,L,U defaultStyle;
```

### メリット

1. データの流れが一方向：
   - データは常に決まった方向に流れるため、アプリケーションの動作が予測しやすくなります。

2. 中央集中型のデータ管理：
   - Storeがアプリケーション全体のデータを一箇所で管理するため、データの把握と制御が容易になります。

3. 決まったプロセスでのデータ更新：
   - データの更新は必ずActionを介して行われるため、変更の追跡と管理が容易になります。

4. 柔軟な機能拡張：
   - Middleware Chainを使うことで、ログ記録や非同期処理などの機能を簡単に追加できます。

5. テストしやすい設計：
   - Reducerは入力に対して常に同じ出力を返す関数なので、動作のテストが容易です。

6. 自動的な画面更新：
   - 状態が変更されると自動的に画面が更新されるため、データと表示の一貫性が保たれます。

## コードのBeforeAfter

### Before

```vba
' UserForm1
Option Explicit

Private Sub UserForm_Initialize()
    txtCounter.Value = 0
    UpdateButtonStates
End Sub

Private Sub btnIncrement_Click()
    txtCounter.Value = txtCounter.Value + 1
    UpdateButtonStates
End Sub

Private Sub btnDecrement_Click()
    txtCounter.Value = txtCounter.Value - 1
    UpdateButtonStates
End Sub

Private Sub btnReset_Click()
    txtCounter.Value = 0
    UpdateButtonStates
End Sub

Private Sub UpdateButtonStates()
    btnDecrement.Enabled = (txtCounter.Value > 0)
End Sub

' Module1
Public Sub PerformComplexOperation()
    ' 複雑な処理
    UserForm1.txtCounter.Value = UserForm1.txtCounter.Value + 10
    UserForm1.UpdateButtonStates
End Sub

' Module2
Public Sub LogCounterValue()
    Debug.Print "現在のカウンター値: " & UserForm1.txtCounter.Value
End Sub
```

### After

```vba
' Store Module
Option Explicit

Private Type State
    Count As Long
End Type

Private CurrentState As State
Private Listeners As Collection

Public Sub InitializeStore()
    Set Listeners = New Collection
    CurrentState.Count = 0
End Sub

Public Function GetState() As State
    GetState = CurrentState
End Function

Public Sub Dispatch(ByVal Action As String, Optional ByVal Payload As Variant)
    Select Case Action
        Case "INCREMENT"
            CurrentState.Count = CurrentState.Count + 1
        Case "DECREMENT"
            If CurrentState.Count > 0 Then
                CurrentState.Count = CurrentState.Count - 1
            End If
        Case "RESET"
            CurrentState.Count = 0
        Case "ADD"
            CurrentState.Count = CurrentState.Count + Payload
    End Select
    NotifyListeners
End Sub

Public Sub AddListener(ByVal Listener As Object)
    Listeners.Add Listener
End Sub

Private Sub NotifyListeners()
    Dim Listener As Variant
    For Each Listener In Listeners
        Listener.OnStateChange CurrentState
    Next Listener
End Sub

' UserForm1
Option Explicit

Private Sub UserForm_Initialize()
    Store.AddListener Me
    UpdateUI
End Sub

Private Sub btnIncrement_Click()
    Store.Dispatch "INCREMENT"
End Sub

Private Sub btnDecrement_Click()
    Store.Dispatch "DECREMENT"
End Sub

Private Sub btnReset_Click()
    Store.Dispatch "RESET"
End Sub

Public Sub OnStateChange(ByVal NewState As State)
    UpdateUI
End Sub

Private Sub UpdateUI()
    Dim CurrentState As State
    CurrentState = Store.GetState
    
    txtCounter.Value = CurrentState.Count
    btnDecrement.Enabled = (CurrentState.Count > 0)
End Sub

' Module1
Public Sub PerformComplexOperation()
    ' 複雑な処理
    Store.Dispatch "ADD", 10
End Sub

' Module2
Public Sub LogCounterValue()
    Dim CurrentState As State
    CurrentState = Store.GetState
    Debug.Print "現在のカウンター値: " & CurrentState.Count
End Sub

' ThisWorkbook
Private Sub Workbook_Open()
    Store.InitializeStore
End Sub
```
