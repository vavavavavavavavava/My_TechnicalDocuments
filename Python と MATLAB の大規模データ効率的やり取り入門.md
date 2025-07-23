# Python と MATLAB で “大規模データ” をストレスなくやり取りするベストプラクティス

---

## はじめに

Python（pandas, polars など）と MATLAB の間で大量データをやり取りするシーンは増えています。しかし、「DataFrame をそのまま渡す」や「型変換をエンジンに任せる」だけでは、**数万行を超えると一気に転送が遅くなる**という問題に直面します。

ここでは、**MATLAB 2025a など最新環境も踏まえた現実的かつ最速の連携手法**をまとめます。  
実験・実務の両方で実証された、“秒”で終わるデータ連携の決定版です。

---

## 1. ありがちな失敗例と現状

### ◎ Python DataFrame → MATLAB table
- **matlabengine** を使えば pandas の DataFrame も自動で MATLAB table に変換できる（R2024a〜R2025aで公式サポート）。
- しかし、**数万行レベルから一気に遅くなり、10秒以上かかる**（ユーザー実体験＋MathWorks フォーラムでも報告多数）。
- Python と MATLAB 間で“逐次コピー＋型変換”が発生しているため、データが大きくなるほど致命的に。

---

## 2. 大規模データに最適な方法＝**Parquet一時ファイル経由**

### なぜ Parquet か？
- Parquet は “列指向の圧縮フォーマット”。pandas, polars, MATLAB（R2020b以降）すべてが高速に読み書き可能。
- RAM ディスク（Linux なら `/dev/shm`）を使えば I/O が実質ゼロコピーに。
- “ファイルを1回書いて渡す”だけなのに、**数百万行でも2〜4秒で渡せる**実績あり。
- 変換エラーや型不一致、バージョン差異に悩まされることがない。

---

## 3. ベストプラクティス・レシピ

### 【Python→MATLAB】大規模データを渡す

```python
import pandas as pd, matlab.engine, tempfile, os

# 任意のデータフレーム（例：数百万行）
df = get_big_dataframe()  

with tempfile.NamedTemporaryFile(dir="/dev/shm", suffix=".parquet", delete=False) as f:
    parquet_path = f.name
    df.to_parquet(parquet_path, compression="zstd", index=False)

eng = matlab.engine.start_matlab()
result = eng.myMatlabFunc(parquet_path, nargout=1)
os.remove(parquet_path)
````

* **ポイント**

  * `/dev/shm`（Linux）や RAM ディスクを指定すると、ファイルI/O遅延がほぼゼロ。
  * Windowsでも`tempfile`で十分速いが、専用RAMディスクの導入も効果的。

---

### 【MATLAB 側】Parquetファイルを読む

```matlab
function outTbl = myMatlabFunc(parquetPath)
    T = parquetread(parquetPath);   % 数百万行も一瞬
    % 必要な解析・加工
    % 例: T.meanVal = mean(T.val, 2);
    outTbl = T;                     % 必要ならテーブル返す
end
```

* **ポイント**

  * `parquetread`はMATLAB R2020b以降で標準サポート。
  * テーブル（table）で返す場合も、Python側でpandas.DataFrameに自動変換される（R2024a以降）。

---

### 【MATLAB→Python】解析後のデータを受け取る場合

* MATLAB関数の返り値としてテーブル（table）を返すのが最も簡単。
* `matlab.engine`では自動でpandas.DataFrameに変換される（R2024a以降）。

```python
df_result = eng.myMatlabFunc(parquet_path, nargout=1)
# df_resultはpandas.DataFrame型として戻る
```

* 返すデータが巨大な場合は、**MATLABからもParquetファイルに書き出してパスだけ返す**のもアリ。
* 例：`parquetwrite(outPath, T);` → Pythonで`pd.read_parquet(outPath)`。

---

## 4. 他の方法や注意点

* **DataFrameをそのままMATLAB関数に渡すのは、行数が多い場合おすすめできない**（極端に遅くなる）。
* **Arrow C-Streamや他の“ゼロコピー”APIは、現状MATLABエンジン＋Pythonでは不安定／バージョン依存や未対応あり**（R2025aでもパブリックに案内されていない）。
* **NumPy配列・dict変換経由も“数値だけ/小規模”ならアリだが、大規模データや異種型カラムではパフォーマンスが悪い**。

---

## 5. ベストプラクティスまとめ

* **Python ⇄ MATLAB で大規模データをやり取りする最適解は「Parquet一時ファイル経由」**

  * pandas, polars, MATLAB いずれでも高速かつバージョン互換性が高い
  * RAMディスクを使えばI/Oコストも実質無視できる
* **DataFrameの“そのまま変換”は小規模以外は避ける**
* **MATLAB table をそのまま返せる（R2024a以降）ので、解析結果もDataFrameで楽々受け取れる**

---

## 6. 参考リンク

* [MathWorks公式：Python Pandas DataFrames を MATLAB で使用](https://jp.mathworks.com/help/matlab/matlab_external/python-pandas-dataframes.html)
* [Parquetサポートに関する MathWorks Blog](https://blogs.mathworks.com/matlab/2023/05/05/working-efficiently-with-data-parquet-files-and-the-needle-in-a-haystack-problem/)
* [pandas.DataFrame.to\_parquet()ドキュメント](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_parquet.html)

---

## 7. おわりに

「Python から MATLAB へ大規模データを渡して解析し、また結果をPythonへ返す」
この流れを**最速・最小コスト**で回すには、**ParquetファイルをRAM上に書いてパスだけやりとりする**のが現時点での鉄板です。
バージョンやOS差異にも強く、今後しばらくの間は“最適解”として使えます。

**大規模データのやり取りに困ったら、まずはこの手法を試してみてください！**
