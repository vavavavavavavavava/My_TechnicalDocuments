# Windows 32ビット環境におけるPandasおよびNumPy導入レポート

**調査日:** 2025-03-06

## 1. 概要

本レポートでは、Windows 32ビット環境でPython（32bit版）を利用し、PandasおよびNumPyをpip（もしくはPoetry経由）で導入する際の最適なバージョン組み合わせについて調査した結果をまとめています。  
特に、`No module named 'numpy.core_umath'` や `No module named 'pandas._libs.interval'` といった具体的なエラーに着目し、バージョン不整合や32bit非対応が原因と考えられる点に注目しています。

## 2. 推奨バージョン組み合わせ

Windows 32bit環境では、最新バージョンのライブラリが32bit対応のホイールを提供していない場合が多いため、以下のバージョンが安定して動作する組み合わせとして推奨されます。  
ここでは、Pythonのバージョンが高い順に記載しています。

### 2.1 Python 3.11 (32bit)
- **Pandas:** 2.0.3  
- **NumPy:** 1.24.3  

**インストール手順（コマンドプロンプトで実行）:**
```bash
pip install numpy==1.24.3
pip install pandas==2.0.3
```
*説明:*  
Python 3.11は32bit環境で利用可能な最新のPythonです。Pandas 2.0.3はWindows 32bit向けホイールが提供されており、NumPyは1.24.3が32bit対応の最後の安定版とされています。

### 2.2 Python 3.10 (32bit)
- **Pandas:** 2.0.3  
- **NumPy:** 1.24.3  

**インストール手順:**
```bash
pip install numpy==1.24.3
pip install pandas==2.0.3
```
*説明:*  
Python 3.10環境でも上記の組み合わせが動作することが確認されています。

### 2.3 Python 3.9 (32bit)
- **Pandas:** 2.0.3  
- **NumPy:** 1.24.3（または1.24.x系）

**インストール手順:**
```bash
pip install numpy==1.24.3
pip install pandas==2.0.3
```
*説明:*  
Python 3.9でも、32bit用ホイールが提供されているため、問題なく利用できます。

### 2.4 Python 3.8 (32bit)
- **Pandas:** 2.0.3  
- **NumPy:** 1.24.3  

**インストール手順:**
```bash
pip install numpy==1.24.3
pip install pandas==2.0.3
```
*注意:*  
Pandas 2.0.3はPython 3.8以上が必須です。Python 3.7以下の場合は、Pandas 1系のバージョンを使用する必要がありますが、Python自体のサポートも終了しているため、可能であればアップグレードを検討してください。

## 3. よくあるエラーと対処方法

### 3.1 `No module named 'numpy.core_umath'` および `No module named 'pandas._libs.interval'`
**原因:**
- ライブラリのインストールが不完全、もしくは依存関係のバージョン不整合
- 32bit環境に適さない最新バージョンのライブラリがインストールされている

**対策:**
- 上記の推奨バージョン（Pandas 2.0.3 と NumPy 1.24.3）を明示的に指定してインストールする  
- インストールキャッシュをクリアし、再インストールを試みる

### 3.2 インポート時のDLLエラー
**原因:**
- 必要なMicrosoft Visual C++ 再頒布可能パッケージがインストールされていない

**対策:**
- 「Microsoft Visual C++ 2015-2022 再頒布可能パッケージ (x86)」をインストールする

### 3.3 メモリ不足 (MemoryError)
**原因:**
- 32bit環境ではプロセスが使用できるメモリ空間が約2GB前後に制限される

**対策:**
- データの分割処理、データ型の圧縮（例: `float64` → `float32`）などでメモリ消費を抑える  
- 必要な場合は64bit環境への移行を検討する

## 4. Poetryでの利用について

上記のバージョン指定は、Poetry経由でも有効です。Poetryは内部でpipを利用するため、`pyproject.toml` の `[tool.poetry.dependencies]` セクションに以下のように記述すれば、同じバージョンのライブラリがインストールされます。

```toml
[tool.poetry.dependencies]
python = ">=3.8,<3.12"
pandas = "==2.0.3"
numpy = "==1.24.3"
```

このように明示的にバージョン指定を行うことで、依存関係の自動解決による誤ったアップグレードを防ぎ、Windows 32bit環境でも安定した導入が可能となります。

## 5. 結論

- **推奨環境:** Windows 32bit上でPython 3.8～3.11の環境
- **推奨組み合わせ:** Pandas 2.0.3 と NumPy 1.24.3（または1.24.x系）
- **対策:** 明示的にバージョンを指定し、必要なランタイム（Microsoft Visual C++ 再頒布可能パッケージ）の確認を行うこと  
- **Poetry利用:** 上記バージョン指定を `pyproject.toml` に記述することで有効に動作する

これにより、`No module named 'numpy.core_umath'` や `No module named 'pandas._libs.interval'` といったエラーを回避し、安定した開発環境の構築が期待できます。