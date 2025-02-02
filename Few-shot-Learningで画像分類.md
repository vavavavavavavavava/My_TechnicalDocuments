
# Few-Shot Learning 手法の技術解説と実装ガイド

本技術解説では、**Few-Shot Learning** の代表的手法である **Prototypical Networks (ProtoNet)**、**Model-Agnostic Meta-Learning (MAML)**、**Relation Network** の概要と特徴、使用シーン、およびサンプル実装例を詳しく解説します。実装例は **Python 3** と **PyTorch** を用いており、独自のデータセットを準備して動作検証が可能です。

## 目次

- [Few-Shot Learning 手法の技術解説と実装ガイド](#few-shot-learning-手法の技術解説と実装ガイド)
  - [目次](#目次)
  - [1. Few-Shot Learning とは](#1-few-shot-learning-とは)
  - [2. 3手法の概要・比較](#2-3手法の概要比較)
    - [2.1 Prototypical Networks (ProtoNet)](#21-prototypical-networks-protonet)
      - [概要](#概要)
      - [動作の流れ](#動作の流れ)
      - [Mermaid による図解](#mermaid-による図解)
      - [特徴と有効なケース](#特徴と有効なケース)
    - [2.2 Model-Agnostic Meta-Learning (MAML)](#22-model-agnostic-meta-learning-maml)
      - [概要](#概要-1)
      - [動作の流れ](#動作の流れ-1)
      - [Mermaid による図解](#mermaid-による図解-1)
      - [特徴と有効なケース](#特徴と有効なケース-1)
    - [2.3 Relation Network](#23-relation-network)
      - [概要](#概要-2)
      - [動作の流れ](#動作の流れ-2)
      - [Mermaid による図解](#mermaid-による図解-2)
      - [特徴と有効なケース](#特徴と有効なケース-2)
    - [2.4 各手法の比較まとめ](#24-各手法の比較まとめ)
  - [3. データセットの用意方法](#3-データセットの用意方法)
  - [4. 実装例のサンプルコード](#4-実装例のサンプルコード)
    - [4.1 共通パート：カスタムデータセットと Few-Shot タスク作成](#41-共通パートカスタムデータセットと-few-shot-タスク作成)
      - [4.1.1 PyTorch Dataset](#411-pytorch-dataset)
      - [4.1.2 N-way K-shot タスク作成関数](#412-n-way-k-shot-タスク作成関数)
    - [4.2 Prototypical Networks の実装例](#42-prototypical-networks-の実装例)
      - [4.2.1 埋め込みネットワーク](#421-埋め込みネットワーク)
      - [4.2.2 Prototypical Loss](#422-prototypical-loss)
      - [4.2.3 トレーニングループ](#423-トレーニングループ)
    - [4.3 Model-Agnostic Meta-Learning (MAML) の実装例](#43-model-agnostic-meta-learning-maml-の実装例)
      - [4.3.1 簡易 CNN モデル](#431-簡易-cnn-モデル)
      - [4.3.2 内ループ（fast adaptation）](#432-内ループfast-adaptation)
      - [4.3.3 メタトレーニングループ（外ループ）](#433-メタトレーニングループ外ループ)
    - [4.4 Relation Network の実装例](#44-relation-network-の実装例)
      - [4.4.1 埋め込みネットワーク](#441-埋め込みネットワーク)
      - [4.4.2 Relation Module](#442-relation-module)
      - [4.4.3 Relation Network の Loss](#443-relation-network-の-loss)
      - [4.4.4 トレーニングループ](#444-トレーニングループ)
  - [5. まとめ](#5-まとめ)

---

## 1. Few-Shot Learning とは

**Few-Shot Learning** とは、少数のラベル付きデータ（例：1クラスあたり 1 枚～数枚程度）から新しいクラスを認識・推論できるように学習する手法群の総称です。  
通常のディープラーニングは、各クラスに大量のラベル付きデータが必要になりますが、少数データしか用意できない場合に **Few-Shot Learning** が有効です。

Few-Shot Learning では、**N-way K-shot** という形式でタスクを定義することが一般的です。  

- **N-way**: クラス数  
- **K-shot**: 各クラスの学習用データ枚数（サポートセット）

学習時には「メタトレーニング」と呼ばれる工程を行い、異なるクラス・タスクを何度もシミュレーションして（episode あるいは task と呼ぶ）モデルに汎用的な学習をさせます。その上で、新たに登場したクラスに対して **K** 枚のサンプルのみで分類が可能なようにする、という流れです。

---

## 2. 3手法の概要・比較

本章では、Few-Shot Learning の代表的な 3 つの手法について、単なる概要に留まらず、各手法がどのような考え方に基づいているのか、実際にどのような流れで動作するのかを分かりやすく説明します。また、Mermaid を使った図解を用いることで、全体の流れや各手法の特徴を直感的に理解できるようにします。

---

### 2.1 Prototypical Networks (ProtoNet)

#### 概要

Prototypical Networks では、各クラスの「サポートセット」にある画像から **特徴ベクトル（埋め込み）** を抽出し、同じクラスに属する画像の特徴を平均して **プロトタイプ（代表ベクトル）** を作成します。  
新たな画像（クエリ画像）の特徴と各クラスのプロトタイプとの **ユークリッド距離**（またはその他の距離）を計算し、最も近いプロトタイプに属するクラスと判断します。

#### 動作の流れ

1. **サポートセットの処理**  
   - 各クラスごとに画像を用意する（例：5クラス、各1枚の場合 → 5枚）。
   - 画像を CNN などで特徴ベクトルに変換。
   - 同一クラスの特徴を平均して、そのクラスの「プロトタイプ」を作成。

2. **クエリ画像の分類**  
   - クエリ画像も同様に特徴ベクトルに変換。
   - 各クラスのプロトタイプとのユークリッド距離を計算。
   - 距離が最も小さいクラスに分類。

#### Mermaid による図解

```mermaid
flowchart TD
    A[サポートセットの各画像]
    B[特徴抽出（CNN など）]
    C[各クラスごとに特徴の平均を計算]
    D[プロトタイプ（クラス代表ベクトル）を生成]
    E[クエリ画像を入力]
    F[クエリ画像も特徴抽出]
    G[各プロトタイプとのユークリッド距離計算]
    H[最も近いプロトタイプのクラスを予測]

    A --> B
    B --> C
    C --> D
    E --> F
    F --> G
    D --> G
    G --> H
```

#### 特徴と有効なケース

- **シンプルで計算負荷が低い：** 学習・推論ともに高速。
- **直感的：** 「各クラスの中心」を求めるので、クラス内のばらつきが大きくない場合に効果的。
- **使用例：** 比較的明確な特徴を持つ画像分類タスク、エピソードベースの学習の導入実験など。

---

### 2.2 Model-Agnostic Meta-Learning (MAML)

#### 概要

MAML は「新しいタスクに対して、**少数回の勾配更新** で素早く適応できる初期パラメータ」を学習する手法です。  
「内ループ」でサポートセットに対して数回の勾配更新を行い、タスク固有のパラメータ（fast adaptation）を得た上で、クエリセットでの誤差をもとに「外ループ」で初期パラメータ自体を改善します。

#### 動作の流れ

1. **初期パラメータの学習（メタ学習）**  
   - 複数のタスク（エピソード）で共通の初期パラメータを更新。

2. **タスクごとの適応（内ループ）**  
   - 各タスクのサポートセットで数回の勾配更新を実施し、タスクに合わせたパラメータ（fast weights）を作成。

3. **タスク評価と外ループ更新**  
   - 生成された fast weights でクエリセットに対する損失を計算し、初期パラメータに関する勾配を逆伝搬。

#### Mermaid による図解

```mermaid
flowchart TD
    subgraph Outer Loop [外ループ：メタトレーニング]
        A[初期パラメータ]
        B[タスクをサンプリング]
        C[各タスクで内ループを実施]
        D[各タスクのクエリ損失を計算]
        E[初期パラメータの更新]
    end

    subgraph Inner Loop [内ループ：タスク適応]
        F[サポートセット入力]
        G[数回の勾配更新]
        H[タスク固有のパラメータ（fast weights）]
    end

    A --> B
    B --> F
    F --> G
    G --> H
    H --> D
    D --> E
```

#### 特徴と有効なケース

- **汎用性が高い：** 画像分類のみならず、回帰や強化学習など多様なタスクに適用可能。
- **初期パラメータの学習：** 新しいタスクへの適応が数ステップで可能となるため、タスク間の変化が大きい場合に有効。
- **計算コスト：** 内ループと外ループの二重構造のため、計算量が多くなりやすい。

---

### 2.3 Relation Network

#### 概要

Relation Network は、サポートセットとクエリ画像間の **類似度（relation）** をニューラルネットワークで学習する手法です。  
固定の距離（ユークリッド距離）を使うのではなく、学習可能な **Relation Module** を用いて、サポート画像の特徴とクエリ画像の特徴を組み合わせたときの「関係性スコア」を出力します。

#### 動作の流れ

1. **特徴抽出**  
   - サポートセットとクエリ画像を、それぞれ CNN などで特徴ベクトルまたは特徴マップに変換する。

2. **サポートプロトタイプの生成**  
   - 各クラスごとに、サポートセットの特徴の平均をとる（または複数のサポート画像と個別に組み合わせる場合もある）。

3. **Relation Module による類似度推定**  
   - クエリ画像の特徴と各クラスのサポート特徴を **連結（concat）** し、Relation Module で類似度スコア（0～1）を出力。
   - このスコアにより、最も関連性の高いクラスを予測する。

#### Mermaid による図解

```mermaid
flowchart TD
    A[サポート画像]
    B[特徴抽出（CNN）]
    C[各クラスごとに特徴を平均または個別に保持]
    D[サポートプロトタイプの生成]
    E[クエリ画像]
    F[特徴抽出（CNN）]
    G[クエリ特徴とサポート特徴を連結]
    H[Relation Module に入力]
    I[類似度スコアを出力]
    J[最も高いスコアのクラスを予測]

    A --> B
    B --> C
    C --> D
    E --> F
    D --> G
    F --> G
    G --> H
    H --> I
    I --> J
```

#### 特徴と有効なケース

- **学習可能な類似度関数：** 固定のユークリッド距離では捉えにくい、複雑な特徴間の関係性を学習可能。
- **柔軟性：** 色調や背景など、画像の見かけが大きく異なる場合でも、関係性を捉えることができる。
- **計算リソース：** 学習パラメータが増え、計算量も多くなりがちなため、データセットの規模やタスクの複雑さに合わせた調整が必要。

---

### 2.4 各手法の比較まとめ

以下の表は、3 つの手法の特徴を簡単に比較したものです。

| 手法                    | 主な考え方                                      | 特徴                                            | 適用例                                               |
|-------------------------|-------------------------------------------------|-------------------------------------------------|------------------------------------------------------|
| **Prototypical Networks** | クラスごとに「中心（プロトタイプ）」を計算し、距離で分類  | シンプル・計算負荷が低い                         | クラス内ばらつきが小さい画像分類、エピソード学習の導入実験  |
| **MAML**                | 初期パラメータを学習し、少数の勾配更新で各タスクに適応     | タスク間の適応性が高く、汎用的                      | タスク間の違いが大きい場合、回帰や強化学習への応用           |
| **Relation Network**    | 学習可能な Relation Module で類似度を算出                | 固定距離では捉えにくい関係性も学習可能、柔軟性あり      | 背景や条件が大きく異なる画像認識、複雑な関係性が存在する場合   |

また、以下の Mermaid 図は、3 つの手法の処理の違いを直感的に理解するための概略フローです。

```mermaid
flowchart LR
    subgraph ProtoNet
        A1[サポート画像 → 特徴抽出]
        A2[各クラスで平均 → プロトタイプ生成]
        A3[クエリ画像 → 特徴抽出]
        A4[ユークリッド距離計算 → 分類]
    end

    subgraph MAML
        B1[初期パラメータ]
        B2[各タスクで内ループ適応]
        B3[クエリセットで評価]
        B4[外ループで初期パラメータ更新]
    end

    subgraph RelationNet
        C1[サポート画像・クエリ画像 → 特徴抽出]
        C2[サポート特徴とクエリ特徴を連結]
        C3[Relation Module で類似度推定]
        C4[最も高いスコアで分類]
    end
```

---

## 3. データセットの用意方法

Few-Shot Learning を行う場合、最も一般的なデータ構造は以下のように「クラスごとにフォルダを分割」して画像を配置する方法です。

```
custom_dataset/
  classA/
    img001.jpg
    img002.jpg
    ...
  classB/
    img001.jpg
    ...
  classC/
    ...
  ...
```

- 上記の `custom_dataset` 配下にクラスごとのサブフォルダがあり、その中に画像を配置。  
- 取り扱いクラス数が多ければ、そのクラス数を増やして同様に画像を配置する。  
- N-way K-shot 学習用に、**各クラスに数枚以上**画像があることが望ましい。  
  - 例：5-way 1-shot かつクエリに 5 枚使う場合、最低でも 6 枚（うち 1 枚サポート、5 枚クエリ）が必要。

本書のサンプルコードでは、PyTorch を用いた簡易的な `Dataset` クラスを紹介します。

---

## 4. 実装例のサンプルコード

以下のサンプル実装には **Python 3** と **PyTorch** を利用しています。  
実際に動かすには、`pip install torch torchvision` などで PyTorch と関連ライブラリをインストールしてください。

### 4.1 共通パート：カスタムデータセットと Few-Shot タスク作成

#### 4.1.1 PyTorch Dataset

```python
import os
from PIL import Image
import torch
from torch.utils.data import Dataset
import torchvision.transforms as T

class CustomImageDataset(Dataset):
    def __init__(self, root_dir, transform=None):
        """
        root_dir: データセットのルートディレクトリ (例: "custom_dataset/")
        transform: 前処理 (transforms.Compose など)
        """
        self.root_dir = root_dir
        self.transform = transform if transform else T.ToTensor()

        # クラス名の一覧を取得 (サブフォルダ名)
        self.classes = sorted(os.listdir(root_dir))

        # 画像ファイルパスと所属クラスインデックスをすべてリスト化
        self.samples = []
        for idx, cls_name in enumerate(self.classes):
            cls_folder = os.path.join(root_dir, cls_name)
            if not os.path.isdir(cls_folder):
                continue  # フォルダ以外は無視
            for fname in os.listdir(cls_folder):
                if fname.lower().endswith(('.jpg', '.jpeg', '.png')):
                    img_path = os.path.join(cls_folder, fname)
                    self.samples.append((img_path, idx))

    def __len__(self):
        return len(self.samples)

    def __getitem__(self, idx):
        img_path, label = self.samples[idx]
        image = Image.open(img_path).convert('RGB')
        if self.transform:
            image = self.transform(image)
        return image, label
```

#### 4.1.2 N-way K-shot タスク作成関数

Few-Shot Learning は「クラスをランダムに N 個選び出し、さらに各クラスから K+Q 枚の画像をサンプリングし、K 枚をサポートセット・Q 枚をクエリセットにする」という操作を頻繁に行います。以下の関数 `make_few_shot_task` は、その操作を簡単に実装したものです。

```python
import random

def make_few_shot_task(dataset, N=5, K=1, Q=5):
    """
    dataset から N-way K-shot タスクを作成し、
    クエリとして各クラス Q 枚ずつサンプリング。
    戻り値: (support_paths, support_labels), (query_paths, query_labels)
    ※ ここでは実際のイメージTensorを返すのではなく、画像パスを返す。
       実際に学習時に transform をかけてロードするようにするため。
    """
    classes = dataset.classes
    # N クラスをランダムに選択
    selected_classes = random.sample(classes, N)
    # 選択されたクラス名 -> 新たなラベル(0〜N-1)
    class_to_new_label = {cls_name: i for i, cls_name in enumerate(selected_classes)}

    # 選択クラスの画像ファイルをまとめる
    class_to_images = {c: [] for c in selected_classes}
    for img_path, label_idx in dataset.samples:
        cls_name = dataset.classes[label_idx]
        if cls_name in selected_classes:
            class_to_images[cls_name].append(img_path)

    support_images, support_labels = [], []
    query_images, query_labels = [], []

    # 各クラスから (K + Q) 枚サンプリング
    for cls_name in selected_classes:
        all_imgs = class_to_images[cls_name]
        selected_imgs = random.sample(all_imgs, K + Q)

        # 前半K枚 -> サポート, 後半Q枚 -> クエリ
        s = selected_imgs[:K]
        q = selected_imgs[K:K+Q]

        new_label = class_to_new_label[cls_name]
        for sp in s:
            support_images.append(sp)
            support_labels.append(new_label)
        for qp in q:
            query_images.append(qp)
            query_labels.append(new_label)

    return (support_images, support_labels), (query_images, query_labels)
```

---

### 4.2 Prototypical Networks の実装例

#### 4.2.1 埋め込みネットワーク

入力画像を高次元の特徴ベクトルに埋め込む CNN（`EmbeddingNet`）を用意します。  
以下では ResNet18 をベースにし、最終層を取り除いた 512 次元の埋め込みを取得する例を示します。

```python
import torch
import torch.nn as nn
from torchvision.models import resnet18

class EmbeddingNet(nn.Module):
    def __init__(self):
        super().__init__()
        base_model = resnet18(pretrained=False)
        # 最終の全結合層(fc)を除去
        self.features = nn.Sequential(*list(base_model.children())[:-1])

    def forward(self, x):
        x = self.features(x)  # [B, 512, 1, 1]  (ResNet18の場合)
        x = x.view(x.size(0), -1)  # [B, 512]
        return x
```

#### 4.2.2 Prototypical Loss

Prototypical Networks では「サポートセットからクラスごとにプロトタイプ（平均ベクトル）を求め、クエリとの距離で分類」を行います。  
そのため、クエリ画像との**ユークリッド距離**を計算し、それを負の値にして softmax で各クラスの確率を求め、クロスエントロピー損失を計算します。

```python
def prototypical_loss(support_embeddings, support_labels, query_embeddings, query_labels, N, K):
    """
    support_embeddings: shape [N*K, embed_dim]
    support_labels:     shape [N*K]
    query_embeddings:   shape [N*Q, embed_dim]
    query_labels:       shape [N*Q]
    N: way
    K: shot
    """
    device = support_embeddings.device
    embed_dim = support_embeddings.size(-1)

    # クラスごとにプロトタイプ(平均ベクトル)を計算
    prototypes = torch.zeros(N, embed_dim).to(device)
    for c in range(N):
        prototypes[c] = support_embeddings[support_labels == c].mean(dim=0)

    # クエリ -> プロトタイプとの距離(ユークリッド)を計算
    distances = torch.cdist(query_embeddings, prototypes)  # [N*Q, N]

    # 距離を負にして softmax -> 各クラスの確率を計算
    log_p_y = nn.functional.log_softmax(-distances, dim=1)  # [N*Q, N]

    # 正解クラスの対数確率を取り出して平均
    loss = -log_p_y[range(len(query_labels)), query_labels].mean()

    # 精度(accuracy)を簡易的に計算
    _, predicted = log_p_y.max(dim=1)
    acc = (predicted == query_labels).float().mean()

    return loss, acc
```

#### 4.2.3 トレーニングループ

実際に**N-way K-shot** のエピソードを繰り返し生成し、ミニバッチとして学習を行います。以下は簡易的な例です。

```python
from PIL import Image
import torch.optim as optim
import torchvision.transforms as T

def train_protonet(dataset_path, num_episodes=1000, N=5, K=1, Q=5):
    # 前処理 (必要に応じて実際の用途に合わせて設定)
    transform = T.Compose([
        T.Resize((224, 224)),
        T.ToTensor()
    ])
    dataset = CustomImageDataset(root_dir=dataset_path, transform=None)  # transformは手動で適用する

    # モデル・オプティマイザの用意
    model = EmbeddingNet().cuda()
    optimizer = optim.Adam(model.parameters(), lr=1e-3)

    for episode in range(1, num_episodes + 1):
        # N-way K-shot タスクを作成
        (support_paths, support_labels), (query_paths, query_labels) = make_few_shot_task(dataset, N, K, Q)

        # 画像をロードして transform + Tensor 化
        support_imgs = torch.stack([transform(Image.open(p).convert('RGB')) for p in support_paths]).cuda()
        query_imgs   = torch.stack([transform(Image.open(p).convert('RGB')) for p in query_paths]).cuda()

        # ラベルを Tensor 化
        support_labels_t = torch.tensor(support_labels).long().cuda()
        query_labels_t   = torch.tensor(query_labels).long().cuda()

        # 埋め込み
        model.train()
        optimizer.zero_grad()
        support_emb = model(support_imgs)  # [N*K, 512]
        query_emb   = model(query_imgs)    # [N*Q, 512]

        # プロトタイプ損失
        loss, acc = prototypical_loss(support_emb, support_labels_t, query_emb, query_labels_t, N, K)

        # 逆伝搬とオプティマイザステップ
        loss.backward()
        optimizer.step()

        if episode % 100 == 0:
            print(f"Episode [{episode}/{num_episodes}]  Loss: {loss.item():.4f}, Acc: {acc.item():.4f}")

    # 学習後のmodelがProtoNetの埋め込みネットワークとして利用可能
    return model
```

---

### 4.3 Model-Agnostic Meta-Learning (MAML) の実装例

MAML は「内ループ」と「外ループ」の二重構造を取ります。  

- **内ループ**: タスク（N-way K-shot）でサポートセットを用いて勾配更新（数ステップ）  
- **外ループ**: 内ループの結果を受けて、クエリセットの損失で初期パラメータを更新

#### 4.3.1 簡易 CNN モデル

以下では、最終的に **N-way** の分類を行うシンプルな CNN を定義します。

```python
import torch.nn as nn

class SimpleCNN(nn.Module):
    def __init__(self, num_classes=5):
        super().__init__()
        self.conv = nn.Sequential(
            nn.Conv2d(3, 32, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(kernel_size=2),

            nn.Conv2d(32, 64, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(kernel_size=2),
        )
        # 入力画像サイズを想定 (例: 224x224 -> 2回プーリングで 56x56)
        self.fc = nn.Linear(64 * 56 * 56, num_classes)

    def forward(self, x):
        x = self.conv(x)
        x = x.view(x.size(0), -1)  # フラット化
        x = self.fc(x)
        return x
```

#### 4.3.2 内ループ（fast adaptation）

以下はサポートセットを用いて数ステップの勾配更新を行い、新しいタスクに適応した「一時的な重み」（fast weights）を生成する処理例です。

```python
def model_forward_with_weights(model, x, weights_dict):
    """
    'weights_dict' で指定されたパラメータを使って forward を行う。
    """
    # モデルの従来パラメータを退避し、weights_dictを一時適用してforward
    original_parameters = {}
    for name, param in model.named_parameters():
        original_parameters[name] = param.data.clone()
        param.data = weights_dict[name].data

    # forward
    outputs = model(x)

    # 元のパラメータに戻す
    for name, param in model.named_parameters():
        param.data = original_parameters[name]

    return outputs

def maml_inner_loop(model, support_imgs, support_labels, loss_fn, inner_lr=0.01, inner_steps=1):
    """
    MAMLの内ループ:
      - サポートセットでモデルを 'inner_steps' 回更新
      - その更新後パラメータ (fast weights) を返す
    
    戻り値:
      fast_weights: { param_name: tensor } のディクショナリ
    """
    # モデルの現在のパラメータを辞書としてコピー
    fast_weights = {name: p for name, p in model.named_parameters()}

    for _ in range(inner_steps):
        # 順伝搬
        outputs = model_forward_with_weights(model, support_imgs, fast_weights)
        loss = loss_fn(outputs, support_labels)

        # 勾配計算
        grads = torch.autograd.grad(loss, fast_weights.values(), create_graph=True)

        # パラメータ更新 (SGD 例)
        updated_fast_weights = {}
        for (name, param), grad in zip(fast_weights.items(), grads):
            updated_fast_weights[name] = param - inner_lr * grad

        fast_weights = updated_fast_weights

    return fast_weights
```

#### 4.3.3 メタトレーニングループ（外ループ）

MAML の外ループでは、クエリセットでの損失を計算し、**初期パラメータ** に関する勾配を更新します。

```python
import copy

def train_maml(dataset_path, meta_lr=1e-3, inner_lr=0.01, outer_steps=1000, inner_steps=1, N=5, K=1, Q=5):
    transform = T.Compose([
        T.Resize((224, 224)),
        T.ToTensor()
    ])
    dataset = CustomImageDataset(root_dir=dataset_path)

    # MAML用モデル (最終出力次元を N にする)
    model = SimpleCNN(num_classes=N).cuda()
    meta_optimizer = optim.Adam(model.parameters(), lr=meta_lr)
    loss_fn = nn.CrossEntropyLoss()

    for step in range(1, outer_steps + 1):
        # 1) N-way K-shot タスクを生成
        (support_paths, support_labels), (query_paths, query_labels) = make_few_shot_task(dataset, N, K, Q)

        # 2) 画像をロード
        support_imgs = torch.stack([transform(Image.open(p).convert('RGB')) for p in support_paths]).cuda()
        query_imgs   = torch.stack([transform(Image.open(p).convert('RGB')) for p in query_paths]).cuda()
        support_labels_t = torch.tensor(support_labels).long().cuda()
        query_labels_t   = torch.tensor(query_labels).long().cuda()

        # 3) 内ループでサポートセットを用いた更新 (fast adaptation)
        fast_weights = maml_inner_loop(
            model, support_imgs, support_labels_t,
            loss_fn, inner_lr=inner_lr, inner_steps=inner_steps
        )

        # 4) クエリセットで loss を計算
        query_outputs = model_forward_with_weights(model, query_imgs, fast_weights)
        query_loss = loss_fn(query_outputs, query_labels_t)

        # 5) クエリの損失を逆伝搬して初期パラメータを更新 (外ループ)
        meta_optimizer.zero_grad()
        query_loss.backward()
        meta_optimizer.step()

        if step % 100 == 0:
            _, predicted = torch.max(query_outputs, dim=1)
            acc = (predicted == query_labels_t).float().mean()
            print(f"Step [{step}/{outer_steps}]  Query Loss: {query_loss.item():.4f}, Acc: {acc.item():.4f}")

    return model
```

---

### 4.4 Relation Network の実装例

#### 4.4.1 埋め込みネットワーク

まずは、サポート画像・クエリ画像を特徴空間に変換するネットワーク `EmbeddingNetRN` を定義します。  
以下は非常にシンプルな CNN 例です。

```python
import torch.nn as nn

class EmbeddingNetRN(nn.Module):
    def __init__(self):
        super().__init__()
        # シンプルな CNN
        self.conv = nn.Sequential(
            nn.Conv2d(3, 64, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(2),

            nn.Conv2d(64, 64, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(2),
        )

    def forward(self, x):
        return self.conv(x)  # [B, 64, H', W']
```

#### 4.4.2 Relation Module

**Relation Module** は、サポート特徴ベクトルとクエリ特徴ベクトルを結合したものを入力として、「類似度スコア（0〜1）」を返すネットワークです。  
以下は CNN + 全結合層でシグモイド出力を返す例です。

```python
class RelationModule(nn.Module):
    def __init__(self, in_channels=128, hidden_dim=64):
        super().__init__()
        self.network = nn.Sequential(
            nn.Conv2d(in_channels, hidden_dim, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(2),

            nn.Conv2d(hidden_dim, hidden_dim, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.AdaptiveAvgPool2d(1),
        )
        self.fc = nn.Linear(hidden_dim, 1)

    def forward(self, x):
        # x: [B, in_channels, H, W]
        out = self.network(x)  # [B, hidden_dim, 1, 1]
        out = out.view(out.size(0), -1)  # [B, hidden_dim]
        out = self.fc(out)               # [B, 1]
        return torch.sigmoid(out)        # 0〜1の範囲
```

#### 4.4.3 Relation Network の Loss

Relation Network では、各クエリ画像に対してすべてのサポートプロトタイプ（またはサポート画像全体）との結合を入力とし、「どのクラスと近いか」を出力スコアで表します。ここでは、ProtoNet のように**各クラスのサポートを平均**して 1 つのプロトタイプにし、そこから Relation Module に通す方法のサンプルを示します。

```python
def relation_network_loss(embedding_net, relation_net, 
                          support_imgs, support_labels, 
                          query_imgs, query_labels,
                          N, K):
    """
    - サポート画像をEmbeddingNetで埋め込み -> クラスごとに平均を取ってサポートプロトタイプを作成
    - クエリ画像もEmbeddingNetで埋め込み
    - (クエリ埋め込み, サポートプロトタイプ) をチャネル方向でconcatし、RelationModuleでスコアを得る
    - スコアを softmax か cross-entropy で学習

    戻り値: (loss, accuracy)
    """
    device = support_imgs.device

    # 埋め込み
    support_features = embedding_net(support_imgs)  # [N*K, 64, h, w]
    query_features = embedding_net(query_imgs)      # [N*Q, 64, h, w]

    # クラスごとにサポートを平均
    prototypes = []
    for c in range(N):
        class_feats = support_features[support_labels == c]  # [K, 64, h, w]
        prototypes.append(class_feats.mean(dim=0, keepdim=True))
    # [N, 64, h, w]
    prototypes = torch.cat(prototypes, dim=0)

    Q = len(query_labels) // N  # 1クラスあたりのクエリ数

    # クエリ(N*Q) x サポートプロトタイプ(N) の組み合わせ
    # Queryを N 回繰り返し、サポートプロトタイプを N*Q 回繰り返して組み合わせる
    query_features_expand = query_features.unsqueeze(1).repeat(1, N, 1, 1, 1)
    query_features_expand = query_features_expand.view(N*Q*N, support_features.size(1), support_features.size(2), support_features.size(3))

    prototypes_expand = prototypes.unsqueeze(0).repeat(N*Q, 1, 1, 1, 1)
    prototypes_expand = prototypes_expand.view(N*Q*N, support_features.size(1), support_features.size(2), support_features.size(3))

    # チャネル方向でconcat -> RelationModuleへ
    relation_pairs = torch.cat([query_features_expand, prototypes_expand], dim=1)  # [N*Q*N, 128, h, w]
    scores = relation_net(relation_pairs)  # [N*Q*N, 1], シグモイド

    # scoresを [N*Q, N] の形に変換
    scores = scores.view(N*Q, N)

    # シグモイド出力を cross-entropy 的に処理するために log_softmax を適用
    log_p_y = torch.nn.functional.log_softmax(scores, dim=1)  # [N*Q, N]

    loss = -log_p_y[range(N*Q), query_labels].mean()

    _, predicted = log_p_y.max(dim=1)
    acc = (predicted == query_labels).float().mean()

    return loss, acc
```

#### 4.4.4 トレーニングループ

実際の学習ループです。

```python
def train_relation_network(dataset_path, num_episodes=1000, N=5, K=1, Q=5):
    transform = T.Compose([
        T.Resize((128, 128)),  # ネットワークの都合でサイズ小さめに
        T.ToTensor()
    ])
    dataset = CustomImageDataset(root_dir=dataset_path)

    embedding_net = EmbeddingNetRN().cuda()
    relation_net = RelationModule(in_channels=128, hidden_dim=64).cuda()

    optimizer = optim.Adam(
        list(embedding_net.parameters()) + list(relation_net.parameters()),
        lr=1e-3
    )

    for episode in range(1, num_episodes + 1):
        # タスクを作成
        (support_paths, support_labels), (query_paths, query_labels) = make_few_shot_task(dataset, N, K, Q)

        # 画像をロード
        support_imgs = torch.stack([transform(Image.open(p).convert('RGB')) for p in support_paths]).cuda()
        query_imgs   = torch.stack([transform(Image.open(p).convert('RGB')) for p in query_paths]).cuda()
        support_labels_t = torch.tensor(support_labels).long().cuda()
        query_labels_t   = torch.tensor(query_labels).long().cuda()

        # 損失と精度の計算
        optimizer.zero_grad()
        loss, acc = relation_network_loss(
            embedding_net, relation_net,
            support_imgs, support_labels_t,
            query_imgs, query_labels_t,
            N, K
        )

        # 逆伝搬とオプティマイザステップ
        loss.backward()
        optimizer.step()

        if episode % 100 == 0:
            print(f"Episode [{episode}/{num_episodes}]  Loss: {loss.item():.4f}, Acc: {acc.item():.4f}")

    return embedding_net, relation_net
```

---

## 5. まとめ

- **Prototypical Networks**  
  - 各クラスの「中心」を求め、その距離を基にクエリ画像を分類します。  
  - シンプルで実装もしやすく、計算負荷も低いため、入門として適しています。

- **MAML**  
  - 複数タスクに対応できるように、少数回の更新でタスク適応が可能な初期パラメータを学習します。  
  - 内外ループの二重構造により、タスク間の変化に柔軟に対応できる反面、計算リソースが必要です。

- **Relation Network**  
  - サポート画像とクエリ画像間の類似度を、学習可能なネットワークで推定します。  
  - 固定の距離尺度では捉えにくい複雑な関係性を学習できるため、見た目や条件が多様なデータに向いています。

以上のように、各手法にはそれぞれの強みと適用シーンが存在します。タスクの性質や利用可能な計算資源、求められる柔軟性などに応じて、最適なアプローチを選択してください。
