# PM2によるプロセス管理ガイド

## はじめに

PM2は、Node.js製のプロセスマネージャです。Node.jsアプリケーションの常駐実行、クラスタリング、ログ管理、自動再起動、ウォッチモードなど、多くの便利な機能を備えています。  
しかしPM2はNode.jsアプリケーションに限らず、任意のコマンドラインで起動できるプログラムを管理できます。そのため、Python仮想環境で動作するPythonアプリケーションなど、さまざまなプロセスを一元的に扱うことが可能です。

このドキュメントでは、PM2を使ったNode.jsアプリケーションおよびPython仮想環境上のPythonアプリケーションの起動・管理方法を解説します。

---

## PM2のインストール

PM2はNode.jsのパッケージとして配布されています。  
事前に[Node.js](https://nodejs.org/)がインストールされている必要があります。

```bash
npm install pm2 -g
```

上記でグローバルインストール後、以下のコマンドでインストール確認ができます。

```bash
pm2 --version
```

---

## 全てのユーザーで共通のPM2を使用する場合の手順

システム全体でPM2を管理し、全ユーザーが共通で使用できるようにする方法について説明します。この方法では、`pm2-installer` を利用してPM2をサービスとして設定します。ユーザーごとのインストールは不要となります。

### 前提条件

1. **Node.js と npm のインストール**
   - 最新版の [Node.js](https://nodejs.org/) をインストールしてください。npm は Node.js に含まれています。
   - Node.js は全ユーザーに対してインストールされている必要があります。

2. **管理者権限のあるユーザーアカウント**
   - 設定を行う際には、管理者権限が必要です。

3. **Pythonの全ユーザーインストール（Pythonプロセスを実行する場合）**
   - Pythonを使用するプロセスを管理する場合、Pythonは全ユーザーに対してインストールされている必要があります。

### 手順

#### 1. pm2-installer のダウンロード

最新バージョンの `pm2-installer` を[こちら](https://github.com/jessety/pm2-installer/archive/main.zip)からダウンロードしてください。

#### 2. pm2-installer をターゲットマシンにコピー

ダウンロードした ZIP ファイルを解凍し、`pm2-installer` ディレクトリ全体をWindowsマシン上の任意の場所（例：`C:\pm2-installer`）にコピーします。

#### 3. 管理者としてコマンドプロンプトまたは PowerShell を起動

- **コマンドプロンプトの場合**
  - スタートメニューから「コマンドプロンプト」を検索し、右クリックして「管理者として実行」を選択します。
  
- **PowerShell の場合**
  - スタートメニューから「PowerShell」を検索し、右クリックして「管理者として実行」を選択します。

#### 4. pm2-installer ディレクトリに移動

コピーした `pm2-installer` ディレクトリに移動します。例えば、`C:\pm2-installer` にコピーした場合：

```bash
cd C:\pm2-installer
```

#### 5. npm の設定を自動的に行う

以下のコマンドを実行して、npm の `prefix` および `cache` ディレクトリを `C:\ProgramData\npm` に設定します。これにより、`Local Service` ユーザーがアクセス可能になります。

```bash
npm run configure
```

#### 6. PowerShell の実行ポリシーを設定

マシンの PowerShell 実行ポリシーを確認し、必要に応じて `RemoteSigned` に変更します。これにより、`pm2` のスクリプトが正常に実行されるようになります。

```bash
npm run configure-policy
```

#### 7. pm2 をサービスとしてセットアップ

以下のコマンドを実行して、pm2 を Windows サービスとしてインストールします。

```bash
npm run setup
```

このコマンドは以下の操作を行います：

- npm のグローバルファイルを `C:\ProgramData\npm` に設定
- pm2 をグローバルにインストール（オフラインキャッシュが必要な場合は使用）
- `C:\ProgramData\pm2` ディレクトリを作成し、`PM2_HOME` 環境変数を設定
- 必要なフォルダの権限を設定
- `node-windows` を使用して pm2 の Windows サービスをインストール
- サービスを `Local Service` ユーザーとして実行するように設定
- サービスが正常に動作していることを確認
- ログファイルのローテーションを行う `pm2-logrotate` モジュールをインストール

#### 8. pm2 の動作確認

サービスが正常に動作しているか確認するために、以下のコマンドを実行します：

```bash
pm2 status
```

管理者権限のターミナルで実行する必要があります。

#### 9. アプリケーションの追加と保存

アプリケーションを pm2 に追加し、プロセスリストを保存します。例えば、`app.js` を追加する場合：

```bash
pm2 start app.js
pm2 save
```

これにより、システムの再起動後もアプリケーションが自動的に起動します。

### 注意事項

- **Pythonプロセスの実行**
  - Pythonを使用するプロセスを管理する場合、Pythonは全ユーザーに対してインストールされている必要があります。ユーザーごとにインストールされたPythonではなく、システム全体にインストールされたPythonを使用してください。

- **既存のユーザー単位のPM2のアンインストール**
  - システム全体でPM2を管理する場合、各ユーザーアカウントにインストールされているPM2をアンインストールすることを推奨します。これにより、競合や混乱を避けることができます。

- **環境変数の管理**
  - システム全体のPM2とユーザー固有のPM2が異なるパスにある場合、それぞれの環境変数を適切に設定する必要があります。可能であれば、システム全体でのPM2管理に統一することをお勧めします。

---

## Node.jsプロジェクトの管理例

### 前提

- Node.jsアプリケーションのエントリポイントが `app.js` であることを想定

### 起動方法

```bash
pm2 start app.js
```

これだけで、`app.js` をバックグラウンドで実行し、プロセス管理を開始します。

### よく使うコマンド

- `pm2 list`  
  現在管理しているプロセス一覧を表示します。

- `pm2 logs`  
  プロセスのログをリアルタイムで表示します。

- `pm2 stop <id|name>`  
  指定したプロセスを停止します。

- `pm2 restart <id|name>`  
  指定したプロセスを再起動します。

- `pm2 delete <id|name>`  
  管理リストからプロセスを削除します。

### 例: 名前を付けて管理

```bash
pm2 start app.js --name "my-node-app"
pm2 stop my-node-app
pm2 restart my-node-app
```

### 設定ファイル (ecosystem.config.js) の利用

複数のプロセスをまとめて起動・管理したい場合は、`ecosystem.config.js`という設定ファイルで定義できます。

```js
// ecosystem.config.js の例
module.exports = {
  apps: [
    {
      name: "my-node-app",
      script: "./app.js",
      watch: true
    }
  ]
};
```

起動コマンドは以下:

```bash
pm2 start ecosystem.config.js
```

---

## Pythonプロジェクトの管理例 (仮想環境対応)

### 前提

- Python仮想環境が `venv` ディレクトリ内に存在する (Windowsの場合は `venv\Scripts\python.exe` がPythonインタプリタ)
- Pythonスクリプトは `script.py` とする
- `script.py`は`venv`でインストールしたパッケージを使用する

### 仮想環境とPM2

PM2は`activate`を経由せずとも、直接仮想環境のPythonインタプリタを指定することで、仮想環境内の依存関係を使用できます。

### 起動例

```bash
pm2 start .\venv\Scripts\python.exe -- script.py
```

これで`venv`内のPythonで`script.py`が実行され、PM2によって管理されます。

### Ecosystemファイルで指定する例

`ecosystem.config.js`ファイルで設定することで、`pm2 start ecosystem.config.js`だけで仮想環境付きPythonスクリプトを起動可能です。

```js
module.exports = {
  apps: [
    {
      name: "my-python-app",
      script: "C:\\path\\to\\project\\venv\\Scripts\\python.exe",
      args: ["script.py"],
      watch: false
    }
  ]
};
```

### ログ・再起動・停止など

基本的な使い方はNode.jsアプリケーションと同様です。

- `pm2 logs my-python-app` でログ確認
- `pm2 restart my-python-app` で再起動
- `pm2 stop my-python-app` で停止
- `pm2 monit`で監視

---

## 自動起動と永続化

PM2には、システム起動時に自動でプロセスを立ち上げるための機能があります。  
`pm2 startup`コマンドを利用してOS起動時の自動起動を設定し、`pm2 save`で現在のプロセスリストを保存することで、システム再起動後も自動で同じプロセスが立ち上がります。

### 例

```bash
pm2 save
pm2 startup
```

上記に従って表示されるコマンドを実行すると、PM2がシステム起動時にプロセスを復元します。

---

## まとめ

- PM2はNode.jsアプリケーションだけでなく、任意のコマンドを管理できる強力なプロセスマネージャです。
- Node.jsプロジェクトはもちろん、Python仮想環境上のスクリプトも簡単なパス指定でPM2管理下に置くことができます。
- `ecosystem.config.js`を用いれば、複数のプロセスを一元的に定義・管理できます。
- ログ管理、クラスタリング、自動再起動、環境変数管理、OS起動時の自動復旧など、運用を楽にする多くの機能が利用できます。
- 全ユーザーで共通のPM2を使用する場合は、`pm2-installer`を利用してシステム全体での管理を行うことで、管理の一元化と安定性を向上させることができます。
- Pythonプロセスを管理する場合は、Pythonを全ユーザーに対してインストールしておく必要があります。

これらを活用することで、開発・運用環境におけるプロセス管理・デプロイがより容易かつ安定したものになります。

---

**参考リンク**

- [PM2公式ドキュメント](https://pm2.keymetrics.io/)
- [pm2-installer GitHubリポジトリ](https://github.com/jessety/pm2-installer)
