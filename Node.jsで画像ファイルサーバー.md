# Node.jsで特定のフォルダ内の画像ファイルを提供するサーバー構築手順

## 概要

本手順書では、Node.js環境において「ある特定のフォルダ内」に配置した画像ファイルをWebサーバーとして提供する簡易的な方法を示します。  
静的ファイル配信にはExpressフレームワークの `express.static()` 機能を用います。

## 前提条件

- Node.jsがインストール済みであること
- npmコマンドが利用可能であること

## ディレクトリ構成例

以下は一例です。`public` ディレクトリ以下に画像やその他静的ファイル(HTML, CSSなど)を配置します。

```
project/
  ├─ server.js
  └─ public/
      ├─ images/
      │  ├─ example.jpg
      │  └─ icons/
      │     └─ icon.png
      └─ index.html
```

`public/images` ディレクトリ配下に画像ファイルを配置しておくと、  
`http://localhost:3000/images/example.jpg` のようなURLでアクセスできるようになります。  
また、サブフォルダ(`icons`ディレクトリ)も構造そのままでアクセス可能です。

## 手順

### 1. Node.js環境の用意

Node.jsが未インストールの場合は、公式サイト(https://nodejs.org/)からLTSバージョンをダウンロード・インストールしてください。

### 2. プロジェクトの初期化

コマンドラインでプロジェクト用ディレクトリへ移動し、`npm init`で`package.json`を生成します。対話形式が面倒な場合は`-y`オプションでスキップ可能です。

```bash
cd project
npm init -y
```

### 3. 必要なパッケージのインストール

Expressをインストールします。

```bash
npm install express
```

### 4. サーバースクリプト `server.js` の作成

`server.js`を以下のように作成します。`express.static()` により `public` ディレクトリ以下が静的配信されます。

```javascript
const express = require('express');
const path = require('path');

const app = express();

// "public" ディレクトリ内を静的ファイルとして配信する
app.use(express.static(path.join(__dirname, 'public')));

// ポート番号(環境変数がなければ3000番ポート)
const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Server is running on http://localhost:${PORT}`);
});
```

### 5. サーバーの起動

```bash
node server.js
```

出力例:  
```
Server is running on http://localhost:3000
```

### 6. ブラウザでアクセス

以下のようなURLで公開した画像にアクセスできます。

- `http://localhost:3000/images/example.jpg` → `public/images/example.jpg` へのアクセス  
- `http://localhost:3000/images/icons/icon.png` → `public/images/icons/icon.png` へのアクセス

サブフォルダもそのまま対応可能です。

## ポイントと応用

1. **`express.static()` の利用**  
   `express.static()` を利用すると、指定ディレクトリ内のファイルがそのまま静的なコンテンツとして配信されます。  
   ディレクトリ構造はURLパスに反映されるため、サブディレクトリを自由に切って画像を整理可能です。

2. **MIMEタイプの自動判定**  
   Expressは拡張子に基づいて適切なContent-Typeヘッダーを返すため、画像ファイルは画像として表示可能です。

3. **セキュリティ・アクセス制御**  
   本手順はあくまでシンプルなファイルサーバー例です。アクセス制御や認証が必要な場合は、`express.static()` の前にミドルウェアを挟む、あるいは別途ルーティングして機能を拡張します。

4. **ポート番号の変更**  
   環境変数`PORT`で指定するか、`server.js`内のハードコードを変更することでポート番号を自由に変更できます。

## まとめ

本手順書ではNode.jsとExpressを利用して、`public` ディレクトリ以下の画像ファイルを簡易的に提供するウェブサーバーの構築例を示しました。  
`express.static()`を利用することで、特定フォルダ内の静的ファイルを簡便に配信できます。  
このベースを発展させれば、認証やデータベース連携など、より複雑なWebアプリケーションに拡張することも可能です。