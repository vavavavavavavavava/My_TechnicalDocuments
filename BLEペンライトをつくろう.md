# BLEペンライトをつくろう

---

## 背景

近年、ライブパフォーマンスにおいて、観客の座席と連動した照明演出が注目されています。本プロジェクトは、各座席に搭載されたペンライトが自動的に同期制御され、統一感のある美しい光の演出を実現するシステムです。観客が個々に操作する必要がなく、ステージ側からの遠隔指示により、会場全体が一体となった演出が可能となります。

---

## プロジェクト概要

本システムは、内蔵の5mm RGB LEDを用いた携帯型ペンライトを、BLE（Bluetooth Low Energy）通信を介して遠隔制御するものです。ESP32搭載のペリフェラル（ペンライト本体）は、ボタン操作によりBLE広告の開始・接続管理を行います。接続されたセントラル（Raspberry Pi）は、HTTP経由で受信したライト命令をBLEでペリフェラルに転送し、RGB LEDの色や点灯状態を制御します。これにより、ステージ側からの統一的なライト制御が実現され、ライブ演出に新たな可能性をもたらします。

---

## システムアーキテクチャ

本システムは、以下の3つの主要コンポーネントで構成されています。

1. **ペリフェラル（ESP32搭載ペンライト）**  
   - **機能**：  
     - ボタンAを押すとBLE広告を開始し、接続待機状態に入る。  
     - ボタンBを押すと、接続中のセントラルとのペアリングを切断する。  
     - 接続後、BLEキャラクタリスティック経由で受信した「カラーコード, on/off」形式の命令に基づき、RGB LEDの色や点灯状態をPWM制御で変更する。

2. **セントラル（Raspberry Pi搭載 FastAPIサーバー＋Bleakライブラリ）**  
   - **機能**：  
     - 常時バックグラウンドでBLEスキャンを実施し、RSSI（受信信号強度）が一定以上（例：–60dBm以上）のペリフェラルを検出する。  
     - 検出したペリフェラルと自動でペアリングし、接続が維持される間、PCやノートPCからHTTPで受信したライト命令をBLE経由で転送する。  
     - 瞬断対策として、一定のタイムアウト期間内に接続が再確立されなければ再スキャンを開始する。

3. **PC/ノートPC（HTTPクライアント）**  
   - **機能**：  
     - FastAPIサーバーへ対して、ライトの指示（カラーコードおよびon/off）をJSON形式のHTTP POSTリクエストで送信する。

この構成により、ライブ演出において座席ごとのペンライトが統一された光の演出を実現し、遠隔操作による同期的な制御が可能となります。

---

## 用語解説

- **ペリフェラル**  
  BLE通信において、広告パケットを送信し、接続要求に応答する側のデバイスです。本プロジェクトでは、ESP32搭載のペンライトが該当します。

- **セントラル**  
  BLE通信で、ペリフェラルから送信される広告パケットをスキャンし、接続要求を行う側のデバイスです。ここでは、Raspberry Piがその役割を担います。

- **BLE (Bluetooth Low Energy)**  
  消費電力を抑えた短距離無線通信規格で、主にウェアラブルデバイスやセンサで利用されます。本プロジェクトのペンライト制御にも採用されています。

- **RSSI (Received Signal Strength Indicator)**  
  受信信号の強度を示す指標で、数値が高いほど物理的に近いデバイスと判断されます。

- **キャラクタリスティック**  
  BLEサービス内で、データの読み書きや通知を行う項目です。ペリフェラルとセントラル間での命令伝達に利用されます。

- **広告パケット**  
  ペリフェラルが自らの情報（デバイス名、サービスUUIDなど）を定期的に送信するパケットです。セントラルはこれを受信してデバイスを検出します。

- **FastAPI**  
  Python製のWebフレームワークで、高速なHTTPサーバーとして動作します。セントラル側でライト命令の受信に利用されます。

---

## 必要部材

### ペリフェラル側（ペンライト本体）
- **ESP32開発ボード（MicroPython対応）**  
  BLE通信機能を搭載しており、ペンライトの制御ユニットとして使用します。

- **5mm RGB LEDモジュール（例：OSTAMA5B31Aタイプ）**  
  カラーチェンジおよび点灯/消灯を行う光源として採用します。

- **タクトスイッチ ×2**  
  - 【ボタンA】：BLE広告開始用（例：GPIO12に接続）  
  - 【ボタンB】：接続中の切断用（例：GPIO13に接続）

- **ソルダーレスプロトタイピング用ブレッドボード**  
  はんだ付け不要で回路の試作やデバッグが可能なため、初期開発に最適です。

- **ジャンパーワイヤー**  
  各部品間の接続に使用します。

- **電池ボックスおよびバッテリー**  
  ペンライトの電源供給用。例として、単三電池×2本用の電池ボックスが考えられます。

- **ペン型エンクロージャ**  
  市販のキットや3Dプリントによる自作ケースを利用し、ペンライトとしての筐体を形成します。

### セントラル側（Raspberry Pi）
- **Raspberry Pi Zero W（またはRaspberry Pi 3/4）**  
  BLE通信とFastAPIサーバーの実行環境として使用します。

- **必要アクセサリ**  
  電源アダプター、microSDカード、ケースなど。

### PC/ノートPC
- **PCまたはノートPC**  
  FastAPIサーバーへHTTPリクエストを送信するクライアントとして利用します。開発環境（Python、HTTPクライアントツールなど）も整備します。

---

## サンプルプログラム

### ① ペリフェラル側（ESP32 / MicroPython）

以下のプログラムは、ボタン操作によりBLE広告の開始・接続管理を行い、接続後にキャラクタリスティックへの書き込みでRGB LEDを制御します。

```python
import uasyncio as asyncio
import aioble
from micropython import const
import bluetooth
from machine import Pin, PWM

# サービス・キャラクタリスティックのUUID（ペリフェラルとセントラルで共通）
_SERVICE_UUID = bluetooth.UUID('12345678-1234-5678-1234-56789abcdef0')
_CHAR_UUID    = bluetooth.UUID('12345678-1234-5678-1234-56789abcdef1')

_FLAG_READ   = const(0x0002)
_FLAG_WRITE  = const(0x0008)
_FLAG_NOTIFY = const(0x0010)

# LED用PWMピン設定（例：赤:GPIO15、緑:GPIO2、青:GPIO4）
pwm_red   = PWM(Pin(15), freq=5000)
pwm_green = PWM(Pin(2),  freq=5000)
pwm_blue  = PWM(Pin(4),  freq=5000)

def set_led_color(hex_color, on):
    """カラーコード（例: "#RRGGBB"）とon/offでLED制御"""
    if on.lower() == "off":
        pwm_red.duty(0)
        pwm_green.duty(0)
        pwm_blue.duty(0)
        print("LED OFF")
    else:
        try:
            r = int(hex_color[1:3], 16)
            g = int(hex_color[3:5], 16)
            b = int(hex_color[5:7], 16)
            duty_r = int(r * 1023 / 255)
            duty_g = int(g * 1023 / 255)
            duty_b = int(b * 1023 / 255)
            pwm_red.duty(duty_r)
            pwm_green.duty(duty_g)
            pwm_blue.duty(duty_b)
            print("LED Color set to", hex_color)
        except Exception as e:
            print("色解析エラー:", e)

# ボタン設定（例：Button A:GPIO12、Button B:GPIO13）
button_a = Pin(12, Pin.IN, Pin.PULL_UP)
button_b = Pin(13, Pin.IN, Pin.PULL_UP)

# BLE初期化
ble = bluetooth.BLE()
ble.active(True)
aioble.init(ble)
service = aioble.Service(_SERVICE_UUID)
characteristic = aioble.Characteristic(service, _CHAR_UUID, _FLAG_READ | _FLAG_WRITE | _FLAG_NOTIFY)
aioble.register_services(service)

current_connection = None

async def advertise_task():
    """ボタンA押下で広告を開始するタスク"""
    adv_data = aioble.advertising_payload(name="ESP32_BLE_Device", services=[_SERVICE_UUID])
    print("広告開始")
    await aioble.advertise(100, adv_data)
    print("広告終了")

async def button_monitor():
    """ボタン状態を監視し、広告開始／切断を実行する"""
    global current_connection
    while True:
        if not button_a.value():
            print("Button A pressed: Start advertising")
            if current_connection is None:
                asyncio.create_task(advertise_task())
            await asyncio.sleep(0.5)
        if not button_b.value():
            print("Button B pressed: Disconnect")
            if current_connection:
                try:
                    current_connection.disconnect()
                except Exception as e:
                    print("切断エラー:", e)
                current_connection = None
            await asyncio.sleep(0.5)
        await asyncio.sleep(0.1)

async def connection_handler():
    """BLE接続を受け付け、キャラクタリスティックの書き込み処理を実行する"""
    global current_connection
    while True:
        connection = await aioble.accept()
        current_connection = connection
        print("セントラル接続")
        try:
            async for write in connection.read_writes(characteristic):
                data = write.value.decode().strip()
                print("受信:", data)
                try:
                    color, state = data.split(',')
                    set_led_color(color, state)
                except Exception as e:
                    print("コマンド解析エラー:", e)
        except Exception as e:
            print("接続エラー:", e)
        finally:
            print("接続終了")
            current_connection = None

async def main():
    asyncio.create_task(button_monitor())
    asyncio.create_task(connection_handler())
    while True:
        await asyncio.sleep(1)

try:
    asyncio.run(main())
except Exception as e:
    print("エラー:", e)
finally:
    asyncio.new_event_loop()
```

---

### ② セントラル側（Raspberry Pi / FastAPI＋Bleak）

FastAPIサーバーとして、HTTPリクエストを受信すると、BLEスキャンでRSSI条件を満たすペリフェラルに接続し、ライト命令を転送します。接続が切断された場合は、再びスキャンを開始します。

```python
# fastapi_central.py
import asyncio
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from bleak import BleakScanner, BleakClient
import uvicorn

TARGET_NAME = "ESP32_BLE_Device"
CHAR_UUID = "12345678-1234-5678-1234-56789abcdef1"
RSSI_THRESHOLD = -60  # dBm（環境に合わせて調整）
RECONNECT_TIMEOUT = 5  # タイムアウト（秒）

app = FastAPI()
connected_client = None
client_lock = asyncio.Lock()

class LightCommand(BaseModel):
    color: str  # 例: "#FF0000"
    onoff: str  # "on" または "off"

async def scan_and_connect():
    global connected_client
    while True:
        async with client_lock:
            if connected_client is None or not connected_client.is_connected:
                print("スキャン中...")
                devices = await BleakScanner.discover(timeout=5.0)
                candidates = [d for d in devices if d.name and TARGET_NAME in d.name and d.rssi >= RSSI_THRESHOLD]
                if not candidates:
                    print("RSSI条件を満たすデバイスが見つかりません")
                    await asyncio.sleep(2)
                    continue
                candidates.sort(key=lambda d: d.rssi, reverse=True)
                device = candidates[0]
                print(f"接続対象: {device.name} (RSSI: {device.rssi}, {device.address})")
                try:
                    client = BleakClient(device.address)
                    await client.connect(timeout=10.0)
                    if client.is_connected:
                        connected_client = client
                        print("接続成功")
                        client.set_disconnected_callback(on_disconnect)
                except Exception as e:
                    print("接続失敗:", e)
                    await asyncio.sleep(2)
        await asyncio.sleep(1)

def on_disconnect(client):
    global connected_client
    print("切断検知")
    connected_client = None

@app.post("/set_light")
async def set_light(cmd: LightCommand):
    command_str = f"{cmd.color},{cmd.onoff}"
    async with client_lock:
        if connected_client is None or not connected_client.is_connected:
            raise HTTPException(status_code=500, detail="接続中のデバイスがありません")
        try:
            await connected_client.write_gatt_char(CHAR_UUID, command_str.encode())
            return {"status": "success", "command": command_str}
        except Exception as e:
            raise HTTPException(status_code=500, detail=str(e))

async def connection_monitor():
    global connected_client
    while True:
        async with client_lock:
            if connected_client is not None and not connected_client.is_connected:
                print("接続が失われました")
                connected_client = None
        await asyncio.sleep(RECONNECT_TIMEOUT)

@app.on_event("startup")
async def startup_event():
    asyncio.create_task(scan_and_connect())
    asyncio.create_task(connection_monitor())

if __name__ == '__main__':
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

### ③ PC/ノートPC側（HTTPクライアント例）

以下は、PCまたはノートPCからFastAPIサーバーへライト命令を送信する例です。

```python
# pc_client.py
import requests

# FastAPIサーバーのURL（Raspberry PiのIPアドレスに合わせて設定）
url = "http://192.168.1.100:8000/set_light"

payload = {
    "color": "#00FF00",  # 例：緑色
    "onoff": "on"
}

try:
    response = requests.post(url, json=payload)
    if response.status_code == 200:
        print("コマンド送信成功:", response.json())
    else:
        print("エラー:", response.status_code, response.text)
except Exception as e:
    print("通信エラー:", e)
```

---

## まとめ

本プロジェクトは、ライブ演出において座席と連動したペンライトシステムの自作を目指します。  
- **ペリフェラル（ESP32搭載ペンライト）** は、ボタン操作によりBLE広告の開始および接続管理を行い、受信した「カラーコード, on/off」形式の命令に基づき、RGB LEDの色と点灯状態をPWM制御で変更します。  
- **セントラル（Raspberry Pi搭載 FastAPIサーバー＋Bleakライブラリ）** は、RSSI条件を満たす近距離のペリフェラルと自動でペアリングし、HTTP経由で受信したライト命令をBLE経由で転送します。接続が切断された場合は、タイムアウト期間を設けて再接続を試みます。  
- **PC/ノートPC** からは、HTTPリクエストを介してFastAPIサーバーにライト指令を送信し、全体としてライブ演出における統一された光の制御が実現されます。

各プログラムや必要部材は、実際の環境（ピン配置、UUID、IPアドレス、RSSI閾値など）に合わせて調整してください。