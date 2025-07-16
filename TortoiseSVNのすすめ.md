# TortoiseSVNのすすめ

～SharePoint直置き・Gitとの比較から始める、安全で簡単なバージョン管理～

---

## 1. バージョン管理方式の比較

| 項目           | SharePoint直置き | TortoiseSVN | Git（GitHub等） |
| ------------ | ------------- | ----------- | ------------ |
| **導入コスト**    | 既存M365等でゼロ    | 無料(サーバー要)   | サーバー次第(有償有)  |
| **履歴管理**     | バージョンのみ       | 詳細なリビジョン管理  | 最強(全履歴・ブランチ) |
| **差分・競合対応**  | ほぼ無し          | 差分管理/ロック可   | 強力(ブランチ標準)   |
| **同時編集耐性**   | 低(上書き事故多)     | ロックである程度可   | 高(分岐文化)      |
| **大規模開発**    | 不向き           | 中～大規模向き     | 超大規模も可       |
| **自動化(CI等)** | 困難            | 難しいが可能      | 標準で充実        |
| **運用管理**     | 簡単            | サーバー構築・管理要  | サーバー次第       |
| **習熟コスト**    | 低             | 普通          | やや高          |

---

## 2. TortoiseSVN の特徴・おすすめ理由

* Windowsのエクスプローラに統合、直感的な右クリック操作でバージョン管理
* ファイル/フォルダ単位で**差分表示・履歴管理・排他ロック**（バイナリ等）も簡単
* チーム開発、特に\*\*「サーバー一元管理＋履歴の明確化＋監査ログ」\*\*が求められる環境に最適
* サーバー構築もVisualSVN ServerなどGUIツールで手軽
* **SharePoint直置きは「同時編集時の衝突」「履歴の弱さ」「CI等との連携困難」などが弱点**
* **Gitは最強だが、運用ルールや教育コストがやや高い場合が多い**

---

## 3. VisualSVN Server 構築手順

### 3.1 ダウンロードとインストール

1. [VisualSVN Server 公式サイト](https://www.visualsvn.com/server/download/)からインストーラーを入手
2. 管理者権限で実行し、ウィザードに従う

   * 基本はデフォルトでOK（リポジトリパス、サービス自動起動など）
   * 認証方式は「Windows認証」または「独自ユーザー管理」を選択可

### 3.2 リポジトリの作成

1. インストール後、`VisualSVN Server Manager` を起動
2. 左ペインで「Repositories」を右クリック→「Create New Repository」
3. 「Single-project repository」または「Multi-project repository」を選択（普通はSingleでOK）
4. リポジトリ名（例：MyProject）を入力→パスをメモ
5. 権限（ユーザー/グループ）を設定して完了

### 3.3 クライアント接続URLの確認

* 例:

  ```
  https://<サーバー名>/svn/MyProject/
  ```

### 3.4 ユーザー/アクセス権設定

* ユーザーごとに「read/write」権限など細かく設定可能
* Active Directory連携も可能

### 3.5 （推奨）バックアップ・メンテナンス

* VisualSVN Serverの「バックアップと復元」機能、または`svnadmin hotcopy`等で定期的にバックアップ

---

## 4. TortoiseSVN の基本的な使い方

### 4.1 チェックアウト（作業コピーの取得）

1. クライアントPCに[TortoiseSVN](https://tortoisesvn.net/downloads.html)をインストール
2. 作業用フォルダを右クリック→`SVN Checkout`
3. サーバーで確認したリポジトリURLを入力し、チェックアウト先フォルダを指定
4. 完了後、そのフォルダ内で作業開始

### 4.2 ファイル編集とコミット

1. ファイルを編集（アイコンが赤色などに変化）
2. 差分確認は右クリック→`TortoiseSVN → Diff`
3. 完了したら右クリック→`SVN Commit`→コメントを付けて送信

### 4.3 他メンバーの変更取り込み（update）

* 作業フォルダを右クリック→`SVN Update`

### 4.4 競合が発生した場合

* 競合ファイルは「！マーク」アイコンに
* 右クリック→`TortoiseSVN → Resolve`や差分比較ツールで手動解決

---

## 5. 注意点・運用ベストプラクティス

* **リポジトリ本体は必ずVisualSVN Server等の専用サーバー上に設置**
  → 共有ドライブや「file:///」アクセスでの複数人同時利用は、静かに破損します
* **定期的なバックアップ・メンテナンス**を忘れずに！
* **コミットコメントは意味のある内容に**
* **作業開始時・コミット前には必ず`SVN Update`**
* **バイナリ等は「ロック」して編集**（TortoiseSVNで右クリック→Get Lock）

---

## 6. 参考リンク

* [VisualSVN Server公式ガイド](https://www.visualsvn.com/server/)
* [TortoiseSVN公式ドキュメント](https://tortoisesvn.net/docs/release/TortoiseSVN_ja/index.html)
* [Subversion公式FAQ: ネットワーク共有での運用は？](https://svnbook.red-bean.com/en/1.7/svn.reposadmin.create.html#svn.reposadmin.create.sharedrepo)
* [Git vs SVN 比較解説](https://www.linode.com/docs/guides/svn-vs-git/)
