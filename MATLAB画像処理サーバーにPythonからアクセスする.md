# MATLAB と Python を用いた画像処理 TCP サーバーの連携例

このサンプルでは、以下のようなプロトコルに基づいて通信を行います。

- **リクエストメッセージ**  
  - 先頭4バイト：リクエストID（int32）  
  - 次の4バイト：ペイロード長（int32）  
  - その後：JPEG形式の画像データ

- **レスポンスメッセージ**  
  - 先頭4バイト：リクエストID（int32）  
  - 次の4バイト：レスポンスペイロード長（int32）  
  - その後：処理済み（例：グレースケール変換）のJPEG画像データ

クライアントは各リクエストに対して、ヘッダーのリクエストIDでレスポンスを照合できる仕様とします。

---

## 1. MATLAB側のTCPサーバープログラム

以下は、MATLABの`tcpserver`関数を用いた画像処理サーバーのサンプルコードです。このサーバーは、接続してきたクライアントから連続したリクエストを受信し、各リクエストに対して対応するレスポンス（ヘッダーにリクエストIDを含む）を返します。

```matlab
function tcpImageProcessingServer(port)
    % ポートが指定されなければ55000を使用
    if nargin < 1
        port = 55000;
    end
    % "0.0.0.0"：全インターフェースで待ち受け
    server = tcpserver("0.0.0.0", port, ...
        "ConnectionChangedFcn", @connectionFcn);
    fprintf("TCP画像処理サーバーをポート%dで起動しました。\n", port);
end

function connectionFcn(src, event)
    if strcmp(event.Connected, "true")
        fprintf("クライアント接続: %s\n", src.ClientAddress);
        % UserDataフィールドにバッファを初期化
        src.UserData.buffer = uint8([]);
        % 少なくとも1バイトごとにコールバック（細かく処理）
        configureCallback(src, "byte", 1, @readFcn);
    else
        fprintf("クライアント切断\n");
    end
end

function readFcn(src, ~)
    % 新規受信データをバッファに追加
    newData = read(src, src.NumBytesAvailable, "uint8");
    if isempty(src.UserData.buffer)
        src.UserData.buffer = newData;
    else
        src.UserData.buffer = [src.UserData.buffer; newData];
    end
    buffer = src.UserData.buffer;
    headerSize = 8;  % リクエストID (4バイト) + ペイロード長 (4バイト)
    
    % バッファ内にヘッダー分のデータがあるか確認
    while numel(buffer) >= headerSize
        % ヘッダーの読み出し
        reqID = typecast(buffer(1:4), 'int32');
        payloadLen = typecast(buffer(5:8), 'int32');
        totalMsgLen = headerSize + double(payloadLen);
        if numel(buffer) < totalMsgLen
            % 完全なリクエストが揃っていない場合は待機
            break;
        end
        
        % JPEG画像データ部分を抽出
        payload = buffer(9:totalMsgLen);
        try
            img = imdecode(payload, "jpg");
        catch ME
            fprintf("画像デコード失敗 (ReqID %d): %s\n", reqID, ME.message);
            % エラー時は該当メッセージを削除して次へ
            buffer(1:totalMsgLen) = [];
            continue;
        end
        
        % 【画像処理例】RGB画像ならグレースケール変換
        if size(img,3) == 3
            processedImg = rgb2gray(img);
        else
            processedImg = img;
        end
        
        % 処理済み画像をJPEGに再エンコード
        processedPayload = imencode(processedImg, "jpg");
        responsePayloadLen = int32(numel(processedPayload));
        
        % レスポンスヘッダー作成（リクエストIDとペイロード長）
        responseHeader = [typecast(reqID, 'uint8'); typecast(responsePayloadLen, 'uint8')];
        % レスポンスメッセージ
        responseMsg = [responseHeader; processedPayload];
        
        % レスポンス送信
        write(src, responseMsg);
        fprintf("リクエストID %d のレスポンスを送信 (ペイロード %dバイト)\n", reqID, responsePayloadLen);
        
        % 処理済み部分をバッファから削除
        buffer(1:totalMsgLen) = [];
    end
    % 不完全なデータを再保存
    src.UserData.buffer = buffer;
end
```

### MATLAB Compilerでのexe化

上記の`tcpImageProcessingServer`関数をスタンドアロンアプリとしてexe化する場合、MATLAB Compilerを利用します。たとえば、コマンドウィンドウから以下のように実行します。

```bash
mcc -m tcpImageProcessingServer.m -d compiled_app
```

※ exe化後は、`compiled_app`フォルダ内に実行可能ファイルが生成されます。実行時にはMATLAB Runtime（対応バージョン）が必要です。

---

## 2. Python側の実装例（pydanticモデルとTCPクライアント）

次に、Python側ではpydanticを使ってリクエスト／レスポンスのデータモデルを定義し、socketを用いたTCP通信クラスを実装します。

### 2.1 pydanticによるデータモデル

```python
from pydantic import BaseModel, Field
import struct

# リクエストデータモデル
class ImageRequest(BaseModel):
    request_id: int = Field(..., description="リクエストID (int32)")
    image_data: bytes = Field(..., description="送信するJPEG画像データ")
    
    @property
    def payload_length(self) -> int:
        return len(self.image_data)
    
    def to_bytes(self) -> bytes:
        # ヘッダー：リトルエンディアンで request_id と payload_length を4バイトずつパック
        header = struct.pack("<ii", self.request_id, self.payload_length)
        return header + self.image_data

# レスポンスデータモデル
class ImageResponse(BaseModel):
    request_id: int = Field(..., description="リクエストID (int32)")
    processed_image_data: bytes = Field(..., description="処理済みJPEG画像データ")
    
    @property
    def payload_length(self) -> int:
        return len(self.processed_image_data)
    
    @classmethod
    def from_bytes(cls, data: bytes) -> "ImageResponse":
        # ヘッダーが8バイト必要
        if len(data) < 8:
            raise ValueError("受信データが不足しています")
        req_id, payload_len = struct.unpack("<ii", data[:8])
        if len(data) < 8 + payload_len:
            raise ValueError("受信データのサイズがヘッダーと一致しません")
        processed_data = data[8:8+payload_len]
        return cls(request_id=req_id, processed_image_data=processed_data)
```

### 2.2 TCP送受信用クラス

```python
import socket
from typing import Optional

class TCPImageClient:
    def __init__(self, host: str, port: int, timeout: Optional[float] = 10.0):
        self.host = host
        self.port = port
        self.timeout = timeout
        self.sock = socket.create_connection((host, port), timeout=timeout)
    
    def send_request(self, request: ImageRequest) -> None:
        data = request.to_bytes()
        self.sock.sendall(data)
    
    def receive_response(self) -> ImageResponse:
        # まずヘッダー（8バイト）を受信
        header = self._recv_exact(8)
        req_id, payload_len = struct.unpack("<ii", header)
        payload = self._recv_exact(payload_len)
        full_msg = header + payload
        response = ImageResponse.from_bytes(full_msg)
        return response
    
    def _recv_exact(self, num_bytes: int) -> bytes:
        chunks = []
        bytes_recd = 0
        while bytes_recd < num_bytes:
            chunk = self.sock.recv(min(num_bytes - bytes_recd, 2048))
            if chunk == b'':
                raise RuntimeError("ソケット接続が切断されました")
            chunks.append(chunk)
            bytes_recd += len(chunk)
        return b''.join(chunks)
    
    def close(self):
        self.sock.close()

# 利用例
if __name__ == "__main__":
    # サーバー接続先（例：localhost:55000）
    client = TCPImageClient("localhost", 55000)
    
    # ローカルファイルからJPEG画像を読み込みリクエスト作成
    with open("input.jpg", "rb") as f:
        img_bytes = f.read()
    
    req = ImageRequest(request_id=1, image_data=img_bytes)
    client.send_request(req)
    print("リクエスト送信完了")
    
    resp = client.receive_response()
    print(f"レスポンス受信: request_id={resp.request_id}, payload_length={resp.payload_length}")
    
    with open("processed_output.jpg", "wb") as f:
        f.write(resp.processed_image_data)
    
    client.close()
```

---

## 3. 全体の動作解説

1. **MATLAB側サーバー**  
   - `tcpImageProcessingServer`を起動すると、指定ポート（例：55000）で待ち受け開始します。  
   - クライアント接続時に、受信データを内部バッファに蓄積し、ヘッダー（リクエストID＋ペイロード長）を元にメッセージを分割。  
   - 画像データを`imdecode`でJPEGとして読み込み、必要に応じた画像処理（例：グレースケール変換）を行い、`imencode`で再度JPEGデータに変換。  
   - レスポンスメッセージは、リクエストIDと処理済み画像のバイト長のヘッダー＋JPEGデータとしてクライアントに返送されるため、クライアント側でリクエストIDに基づく対応が可能となります。

2. **MATLAB Compilerでのexe化**  
   - 上記サーバーコードを`tcpImageProcessingServer.m`として保存し、MATLAB Compilerコマンド（例：`mcc -m tcpImageProcessingServer.m -d compiled_app`）を実行することで、スタンドアロンexeが生成されます。  
   - exe実行時は、MATLAB Runtimeが必要です。

3. **Python側クライアント**  
   - pydanticモデルで、リクエスト／レスポンスの構造を定義。  
   - `TCPImageClient`クラスにより、指定ホスト・ポートにTCP接続を行い、ImageRequestのto_bytes()で生成されたバイナリを送信。  
   - サーバーから受信したバイナリデータを、先頭のヘッダー情報を元にImageResponseモデルにパースし、処理済み画像データを取得する。  
   - その後、取得したJPEG画像データをローカルファイルに保存するなどして利用する。

---

## 4. まとめ

このサンプルでは、MATLAB側のtcpserverを用いて画像処理サーバーを実装し、クライアントからのリクエストに対してリクエストIDをそのままレスポンスヘッダーに返すことで、どのレスポンスがどのリクエストに対応しているかを明示しています。  
また、MATLAB Compilerでexe化する手法も示しており、Python側ではpydanticによるデータモデルとTCP通信クラスを用いて、サーバーとのリクエスト／レスポンス通信を実現しています。

このような実装例をもとに、用途に合わせたエラー処理やプロトコルの拡張を検討してください。

---

【参考リンク】  
- MATLAB tcpserver: [https://www.mathworks.com/help/matlab/ref/tcpserver.html](https://www.mathworks.com/help/matlab/ref/tcpserver.html)   
- MATLAB Compilerの使用方法: [https://www.mathworks.com/help/compiler/](https://www.mathworks.com/help/compiler/)

