# Python と MATLAB の大規模データ効率的やり取り入門  
〜 数百万行のデータを秒で往復する！実践ガイド 〜

---

## はじめに

近年のデータ解析現場では「PythonでDBや機械学習、MATLABでアルゴリズム開発や制御」と役割を分担するケースが増えています。  
一方で、両者の間で“数百万行〜億行”クラスのデータをやり取りすると、**意外なほど時間がかかる**、または**メモリが足りなくなる**といった課題に直面しがちです。

このドキュメントでは、**データ転送コストをほぼ無視できるレベルまで下げる**ためのノウハウを、パターン別にまとめます。  
MATLAB と Python の連携を“速く・スマート”に実現したい方、必見です。

---

## 1. そもそも何がボトルネック？

多くの人が以下のようなパターンに悩みます。

- PandasのDataFrameをmatlabengineで直接渡すと激遅
- テキスト/CSV経由やdict変換だと小さいデータは良いが大規模だと秒→分へ
- 逆にMATLABからtable/timetableを返すと、Pythonでどう受ければ速いか分からない

**ポイント**は「PythonとMATLABの間で“いかに型変換・データコピーを減らすか」です。

---

## 2. 主なデータ転送方式の比較

| 方式 | 典型速度 (100 万行×20 列) | 必要条件 | 長所 | 短所 |
|------|-------------------------|----------|------|------|
| **Arrow C-Stream** | **1–2 秒** | MATLAB R2024b+ & PyArrow ≥ 14 | ゼロコピー／異種型もOK | 新しい環境必須 |
| **Parquet (tmpfs)** | 2–4 秒 | R2020b+ | 列指向・圧縮／古い環境でも可 | 一時ファイル生成が必要 |
| NumPy → `matlab.double` | 6–10 秒 | R2023a+ | 数値だけなら速い | 異種型NG・列名情報が消える |
| DataFrame → dict → `table` | 数十秒〜分 | どのバージョンでも | シンプル・安心 | ループ地獄で激遅・非推奨 |

**結論**  
“速さ”重視なら、**Arrow C-Stream** か **Parquet** 一択です。

---

## 3. Python → MATLAB へのデータ転送

### 3.1 Arrow C-Stream（現状最速）

#### 仕組み
Apache Arrowは、列指向かつプラットフォーム共通の“生メモリバッファ”をそのままやりとりできる技術。  
**シリアライズ/デシリアライズのコストがほぼゼロ**になるのが最大のメリットです。

#### 実装例

```python
# df_to_matlab_arrow.py
"""
DataFrame を MATLAB 関数へゼロコピー送信
"""
import pyarrow as pa, pandas as pd, matlab.engine

df: pd.DataFrame = get_big_dataframe()
tbl = pa.Table.from_pandas(df, preserve_index=False)
capsule = tbl._export_to_c()  # PyCapsuleでゼロコピー

eng = matlab.engine.start_matlab()
mat_tbl = eng.feval("arrow.importRecordBatchStream", capsule)
out  = eng.myMatlabFunc(mat_tbl, nargout=1)
