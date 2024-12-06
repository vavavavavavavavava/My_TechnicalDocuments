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
pm2 start venv\Scripts\python.exe -- script.py
```

これで`venv`内のPythonで`script.py`が実行され、PM2によって管理されます。

### Ecosystemファイルで指定する例

`ecosystem.config.js`ファイルで設定することで、`pm2 start ecosystem.config.js`だけで仮想環境付きPythonスクリプトを起動可能です。

```js
module.exports = {
  apps: [
    {
      name: "my-python-app",
      script: "script.py",
      interpreter: "C:\\path\\to\\project\\venv\\Scripts\\python.exe",
      watch: false
    }
  ]
};
```

上記のように`interpreter`で仮想環境のPythonパスを指定することで、`script.py`がその仮想環境で動作します。

### ログ・再起動・停止など
基本的な使い方はNode.jsアプリケーションと同様です。

- `pm2 logs my-python-app` でログ確認
- `pm2 restart my-python-app` で再起動
- `pm2 stop my-python-app` で停止

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

これらを活用することで、開発・運用環境におけるプロセス管理・デプロイがより容易かつ安定したものになります。