# SQL Query and VBA Function Documentation

vba上では以下のように1モジュールでクエリ操作をまとめる
```
' QueryBuilder Module
Option Explicit

' 定数定義
Private Const CONNECTION_STRING As String = "Your connection string here"

' クエリ定義
Public Function GetUserByID(userID As Long) As String
    GetUserByID = "SELECT * FROM Users WHERE UserID = " & userID
End Function

Public Function InsertNewUser(userName As String, email As String) As String
    InsertNewUser = "INSERT INTO Users (UserName, Email) VALUES ('" & userName & "', '" & email & "')"
End Function

Public Function UpdateUserEmail(userID As Long, newEmail As String) As String
    UpdateUserEmail = "UPDATE Users SET Email = '" & newEmail & "' WHERE UserID = " & userID
End Function

Public Function DeleteUser(userID As Long) As String
    DeleteUser = "DELETE FROM Users WHERE UserID = " & userID
End Function

' クエリ実行関数
Public Function ExecuteQuery(query As String) As ADODB.Recordset
    Dim conn As ADODB.Connection
    Dim rs As ADODB.Recordset
    
    Set conn = New ADODB.Connection
    conn.Open CONNECTION_STRING
    
    Set rs = New ADODB.Recordset
    rs.Open query, conn, adOpenStatic, adLockOptimistic
    
    Set ExecuteQuery = rs
    
    conn.Close
    Set conn = Nothing
End Function

' トランザクション管理
Public Sub ExecuteTransactionQueries(queries As Variant)
    Dim conn As ADODB.Connection
    Dim i As Long
    
    Set conn = New ADODB.Connection
    conn.Open CONNECTION_STRING
    
    conn.BeginTrans
    
    On Error GoTo ErrorHandler
    
    For i = LBound(queries) To UBound(queries)
        conn.Execute queries(i)
    Next i
    
    conn.CommitTrans
    
    conn.Close
    Set conn = Nothing
    Exit Sub
    
ErrorHandler:
    conn.RollbackTrans
    conn.Close
    Set conn = Nothing
    Err.Raise Err.Number, Err.Source, Err.Description
End Sub
```

## Query Name: [クエリの名前や目的]

### 説明
[クエリの目的や機能の詳細な説明]

### 使用箇所
- [アプリケーション内でこのクエリが使用される場所や状況]

### 関連トランザクション
- [このクエリが他のクエリと共にトランザクションを形成する場合、その詳細]

### SQL クエリ
```sql
[ここにSQLクエリを記述]
```

### VBA 関数
```vba
Public Function [関数名](パラメータ1 As データ型, パラメータ2 As データ型, ...) As String
    [関数名] = "[SQLクエリ文字列]"
End Function

' 実行用関数（必要に応じて）
Public Function Execute[関数名](パラメータ1 As データ型, パラメータ2 As データ型, ...) As [戻り値の型]
    Dim conn As ADODB.Connection
    Dim rs As ADODB.Recordset
    Dim queryStr As String
    
    Set conn = New ADODB.Connection
    conn.Open CONNECTION_STRING
    
    queryStr = [関数名](パラメータ1, パラメータ2, ...)
    
    Set rs = conn.Execute(queryStr)
    
    ' 結果の処理
    ' ...
    
    conn.Close
    Set conn = Nothing
    
    ' 戻り値の設定
    ' Set Execute[関数名] = ...
End Function
```

### パラメータ
- `[パラメータ名]`: [パラメータの説明、型、制約など]

### 戻り値
[クエリが返すデータの説明、形式など]

### 使用例
```vba
' VBA関数の使用例
Dim result As [戻り値の型]
result = Execute[関数名](値1, 値2, ...)
```

### 注意事項
- [実行時の注意点、パフォーマンスに関する考慮事項など]

---