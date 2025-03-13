# React-Konvaの魅力：高度なキャンバスベースUIを簡単に

Reactアプリケーションでインタラクティブなグラフィックスを実装したいと考えたことはありませんか？SVGやCanvas APIでの実装に悩んだ経験はないでしょうか？React-Konvaは、そんな悩みを解決する強力なソリューションです。

## React-Konvaとは？

React-Konvaは、HTML5 Canvasをベースとした高性能グラフィックスライブラリである[Konva.js](https://konvajs.org/)をReactで簡単に使えるようにしたラッパーライブラリです。これにより、宣言的なReactのアプローチを保ちながら、Canvasの高度な描画機能とパフォーマンスを活用できます。

## React-Konvaの主な魅力

### 1. 宣言的なAPIでCanvasを操作

通常のCanvas APIは命令型で、コードが複雑になりがちです。React-Konvaは宣言的なアプローチを提供し、Reactの哲学に沿った方法でキャンバスコンテンツを定義できます。

```jsx
<Stage width={500} height={500}>
  <Layer>
    <Circle x={200} y={200} radius={50} fill="red" />
    <Text text="Hello Canvas!" x={10} y={10} />
  </Layer>
</Stage>
```

### 2. Reactのエコシステムとの統合

React-Konvaは、ReactのHooksやコンテキストといった最新機能と完全に互換性があります。コンポーネントベースの設計と状態管理の恩恵をそのまま受けられます。

```jsx
const [position, setPosition] = useState({ x: 100, y: 100 });

// ...

<Circle 
  x={position.x} 
  y={position.y}
  draggable
  onDragEnd={(e) => {
    setPosition({ x: e.target.x(), y: e.target.y() });
  }}
/>
```

### 3. 高度なインタラクション

ドラッグ＆ドロップ、ズーム、回転、選択など、複雑なインタラクションを簡単に実装できます。これらの機能は組み込みでサポートされており、カスタム実装の必要がありません。

### 4. パフォーマンス

HTML要素やSVGと比較して、Canvasは多数の要素を扱う場合に優れたパフォーマンスを発揮します。React-Konvaは効率的なレンダリングを行い、スムーズなアニメーションや大量のオブジェクトの表示を可能にします。

### 5. グループ化と階層構造

`Group`コンポーネントを使用して、関連するシェイプをグループ化できます。これにより、複雑な構造を持つUI要素を簡単に管理できます。

```jsx
<Group x={100} draggable>
  <Circle radius={50} fill="green" />
  <Text text="Drag me!" y={-60} />
</Group>
```

## 実践的なコンポーネント例

以下は、React-Konvaを使用した高度なインタラクティブマップコンポーネントの例です。このコンポーネントは、背景画像上にドラッグ可能な点を配置し、座標系を持ち、リサイズ機能も備えています。

```tsx
import React, { useState, useRef, useEffect } from 'react';
import { Stage, Layer, Circle, Text, Transformer, Group, Image, Rect, Line } from 'react-konva';
import Konva from 'konva';

// 点のデータ型定義
export interface Point {
  id: string | number;
  x: number;
  y: number;
  name: string;
}

// 座標系定義
export interface CoordinateSystem {
  topRight: { x: number; y: number }; // 右上の座標
}

// コンポーネントのProps定義
interface DraggablePointsCanvasProps {
  points: Point[];
  width?: number;
  height?: number;
  backgroundImage?: string; // 背景画像のURL
  coordinateSystem?: CoordinateSystem;
  onPointMove?: (id: string | number, x: number, y: number) => void;
  onPointNameChange?: (id: string | number, newName: string) => void;
  onCoordinateSystemChange?: (newCoordSystem: CoordinateSystem) => void;
}

// ハンドルの位置を示す型
type HandlePosition = 'topLeft' | 'topRight' | 'bottomLeft' | 'bottomRight' | 'top' | 'right' | 'bottom' | 'left';

const DraggablePointsCanvas: React.FC<DraggablePointsCanvasProps> = ({
  points,
  width = 800,
  height = 600,
  backgroundImage,
  coordinateSystem = { topRight: { x: 100, y: 100 } },
  onPointMove,
  onPointNameChange,
  onCoordinateSystemChange
}) => {
  // 選択中の点のID
  const [selectedId, setSelectedId] = useState<string | number | null>(null);
  
  // 編集中のテキスト関連のstate
  const [editingText, setEditingText] = useState<boolean>(false);
  const [editingPointId, setEditingPointId] = useState<string | number | null>(null);
  const [editText, setEditText] = useState<string>('');

  // 画像とキャンバスサイズ関連のstate
  const [image, setImage] = useState<HTMLImageElement | null>(null);
  const [canvasSize, setCanvasSize] = useState({ width, height });
  
  // リサイズハンドル関連のstate
  const [isResizing, setIsResizing] = useState<boolean>(false);
  const [activeHandle, setActiveHandle] = useState<HandlePosition | null>(null);
  const [originalAspectRatio, setOriginalAspectRatio] = useState<number>(1);
  const [originalCoordSystem, setOriginalCoordSystem] = useState<CoordinateSystem>(coordinateSystem);
  
  // Refs
  const stageRef = useRef<Konva.Stage>(null);
  const transformerRef = useRef<Konva.Transformer>(null);
  const textEditRef = useRef<HTMLInputElement>(null);
  const textNodesRef = useRef<{ [key: string]: Konva.Text }>({});

  // 背景画像の読み込み
  useEffect(() => {
    if (backgroundImage) {
      const img = new window.Image();
      img.src = backgroundImage;
      img.onload = () => {
        setImage(img);
        // 画像のサイズに合わせてキャンバスのサイズを調整
        setCanvasSize({
          width: img.width,
          height: img.height
        });
        // アスペクト比を記録
        setOriginalAspectRatio(img.width / img.height);
      };
    }
  }, [backgroundImage]);

  // 座標系が更新されたら記録
  useEffect(() => {
    setOriginalCoordSystem(coordinateSystem);
  }, [coordinateSystem]);

  // Transformerの設定を更新する
  useEffect(() => {
    if (selectedId !== null && transformerRef.current && stageRef.current) {
      try {
        const selectedNode = stageRef.current.findOne(`#point-${selectedId}`);
        if (selectedNode) {
          transformerRef.current.nodes([selectedNode as any]);
          transformerRef.current.getLayer()?.batchDraw();
        }
      } catch (error) {
        console.error("Error setting transformer:", error);
      }
    } else if (transformerRef.current) {
      transformerRef.current.nodes([]);
      transformerRef.current.getLayer()?.batchDraw();
    }
  }, [selectedId]);

  // 座標変換関数（論理座標からキャンバス座標へ）
  const transformCoordinates = (x: number, y: number) => {
    // 左下が(0,0)、右上がcoordinateSystem.topRightとなるように変換
    const canvasX = (x / coordinateSystem.topRight.x) * canvasSize.width;
    const canvasY = canvasSize.height - (y / coordinateSystem.topRight.y) * canvasSize.height;
    return { x: canvasX, y: canvasY };
  };

  // 逆座標変換関数（キャンバス座標から論理座標へ）
  const inverseTransformCoordinates = (canvasX: number, canvasY: number) => {
    // キャンバス座標から論理座標に変換
    const x = (canvasX / canvasSize.width) * coordinateSystem.topRight.x;
    const y = ((canvasSize.height - canvasY) / canvasSize.height) * coordinateSystem.topRight.y;
    return { x, y };
  };

  // ドラッグ終了時のハンドラ（点の移動）
  const handleDragEnd = (e: Konva.KonvaEventObject<DragEvent>, id: string | number) => {
    const canvasX = e.target.x();
    const canvasY = e.target.y();
    
    // キャンバス座標から論理座標に変換
    const { x, y } = inverseTransformCoordinates(canvasX, canvasY);
    
    // コールバックを呼び出す（論理座標で通知）
    if (onPointMove) {
      onPointMove(id, x, y);
    }
  };

  // リサイズハンドルのドラッグ開始ハンドラ
  const handleResizeStart = (position: HandlePosition) => {
    setIsResizing(true);
    setActiveHandle(position);
  };

  // リサイズハンドルのドラッグ中ハンドラ
  const handleResizeDrag = (e: Konva.KonvaEventObject<DragEvent>, position: HandlePosition) => {
    if (!isResizing || !activeHandle) return;
    
    // キャンバスのサイズを取得
    const stageWidth = canvasSize.width;
    const stageHeight = canvasSize.height;
    
    // ドラッグ位置を取得
    const pointerPos = e.target.getStage()?.getPointerPosition();
    if (!pointerPos) return;
    
    // 新しいサイズを計算（左下固定）
    let newWidth = stageWidth;
    let newHeight = stageHeight;
    let newTopRightX = coordinateSystem.topRight.x;
    let newTopRightY = coordinateSystem.topRight.y;
    
    // アスペクト比を維持するための係数
    const aspectRatio = originalAspectRatio;
    
    switch (position) {
      case 'topRight':
        newWidth = pointerPos.x;
        newHeight = stageHeight - (newWidth / aspectRatio);
        break;
      case 'bottomRight':
        newWidth = pointerPos.x;
        newHeight = pointerPos.y;
        break;
      case 'topLeft':
        // 左上は左下固定なので特殊処理
        newWidth = stageWidth - pointerPos.x;
        newHeight = stageHeight - (newWidth / aspectRatio);
        break;
      case 'bottomLeft':
        // 左下は固定点なので動かない
        return;
      case 'right':
        newWidth = pointerPos.x;
        newHeight = newWidth / aspectRatio;
        break;
      case 'top':
        newHeight = stageHeight - pointerPos.y;
        newWidth = newHeight * aspectRatio;
        break;
      case 'left':
        // 左辺は左下固定なので特殊処理
        newWidth = stageWidth - pointerPos.x;
        newHeight = newWidth / aspectRatio;
        break;
      case 'bottom':
        newHeight = pointerPos.y;
        newWidth = newHeight * aspectRatio;
        break;
    }
    
    // 最小サイズの制限
    newWidth = Math.max(newWidth, 100);
    newHeight = Math.max(newHeight, 100);
    
    // サイズ変更に合わせて座標系も調整
    const scaleFactor = newWidth / stageWidth;
    newTopRightX = originalCoordSystem.topRight.x * scaleFactor;
    newTopRightY = originalCoordSystem.topRight.y * (newHeight / stageHeight);
    
    // 状態を更新
    if (onCoordinateSystemChange) {
      onCoordinateSystemChange({ topRight: { x: newTopRightX, y: newTopRightY } });
    }
  };

  // リサイズハンドルのドラッグ終了ハンドラ
  const handleResizeEnd = () => {
    setIsResizing(false);
    setActiveHandle(null);
    setOriginalCoordSystem(coordinateSystem);
  };

  // リサイズハンドルの描画
  const renderResizeHandles = () => {
    // キャンバスの4辺と4角にハンドルを配置
    const handleRadius = 8;
    const frameStrokeWidth = 2;
    
    // キャンバスサイズを取得
    const stageWidth = canvasSize.width;
    const stageHeight = canvasSize.height;
    
    // フレーム
    const frame = (
      <Rect
        x={0}
        y={0}
        width={stageWidth}
        height={stageHeight}
        stroke="#00A0FF"
        strokeWidth={frameStrokeWidth}
        dash={[5, 5]}
      />
    );
    
    // ハンドル位置の設定
    const handles = [
      { position: 'topLeft', x: 0, y: 0 },
      { position: 'topRight', x: stageWidth, y: 0 },
      { position: 'bottomLeft', x: 0, y: stageHeight },
      { position: 'bottomRight', x: stageWidth, y: stageHeight },
      { position: 'top', x: stageWidth / 2, y: 0 },
      { position: 'right', x: stageWidth, y: stageHeight / 2 },
      { position: 'bottom', x: stageWidth / 2, y: stageHeight },
      { position: 'left', x: 0, y: stageHeight / 2 }
    ];
    
    return (
      <>
        {frame}
        {handles.map((handle) => (
          <Circle
            key={handle.position}
            x={handle.x}
            y={handle.y}
            radius={handleRadius}
            fill="#00A0FF"
            stroke="#FFFFFF"
            strokeWidth={2}
            draggable
            onMouseDown={() => handleResizeStart(handle.position as HandlePosition)}
            onDragMove={(e) => handleResizeDrag(e, handle.position as HandlePosition)}
            onDragEnd={handleResizeEnd}
          />
        ))}
      </>
    );
  };

  // テキスト編集を開始
  const startTextEdit = (id: string | number, initialText: string) => {
    setEditingPointId(id);
    setEditText(initialText);
    setEditingText(true);
    
    // 該当するテキストノードの位置を取得
    const textNode = textNodesRef.current[id.toString()];
    if (textNode && stageRef.current) {
      const textPosition = textNode.getAbsolutePosition();
      const stageContainer = stageRef.current.container();
      
      // 入力フィールドを配置
      if (textEditRef.current && stageContainer) {
        const containerRect = stageContainer.getBoundingClientRect();
        textEditRef.current.style.position = 'absolute';
        textEditRef.current.style.top = `${containerRect.top + textPosition.y}px`;
        textEditRef.current.style.left = `${containerRect.left + textPosition.x}px`;
        textEditRef.current.style.width = `${textNode.width()}px`;
        textEditRef.current.style.height = `${textNode.height()}px`;
        textEditRef.current.style.fontSize = `${textNode.fontSize()}px`;
        textEditRef.current.style.display = 'block';
        textEditRef.current.focus();
      }
    }
  };

  // テキスト編集を完了
  const completeTextEdit = () => {
    setEditingText(false);
    
    if (editingPointId !== null && onPointNameChange) {
      onPointNameChange(editingPointId, editText);
    }
    
    setEditingPointId(null);
    
    if (textEditRef.current) {
      textEditRef.current.style.display = 'none';
    }
  };

  return (
    <div style={{ position: 'relative' }}>
      {/* テキスト編集用の入力フィールド */}
      <input
        ref={textEditRef}
        value={editText}
        onChange={(e) => setEditText(e.target.value)}
        onBlur={completeTextEdit}
        onKeyDown={(e) => {
          if (e.key === 'Enter') {
            completeTextEdit();
          }
        }}
        style={{
          position: 'absolute',
          display: 'none',
          zIndex: 1,
          background: 'white',
          border: '1px solid black'
        }}
      />
      
      <Stage
        width={canvasSize.width}
        height={canvasSize.height}
        ref={stageRef}
        onMouseDown={(e) => {
          // 背景をクリックした場合、選択解除
          if (e.target === e.target.getStage()) {
            setSelectedId(null);
          }
        }}
      >
        <Layer>
          {/* 背景画像 */}
          {image && (
            <Image 
              image={image}
              width={canvasSize.width}
              height={canvasSize.height}
            />
          )}
          
          {/* リサイズハンドル */}
          {renderResizeHandles()}
          
          {/* 座標軸（参考用） */}
          <Line
            points={[0, canvasSize.height, canvasSize.width, canvasSize.height]}
            stroke="#666666"
            strokeWidth={1}
            dash={[2, 2]}
          />
          <Line
            points={[0, canvasSize.height, 0, 0]}
            stroke="#666666"
            strokeWidth={1}
            dash={[2, 2]}
          />
          
          {/* 点とラベル */}
          {points.map((point) => {
            // 論理座標からキャンバス座標に変換
            const canvasCoords = transformCoordinates(point.x, point.y);
            
            return (
              <Group key={point.id}>
                <Circle
                  id={`point-${point.id}`}
                  x={canvasCoords.x}
                  y={canvasCoords.y}
                  radius={10}
                  fill="blue"
                  stroke="black"
                  strokeWidth={1}
                  draggable
                  onClick={() => setSelectedId(point.id)}
                  onTap={() => setSelectedId(point.id)}
                  onDragEnd={(e) => handleDragEnd(e, point.id)}
                />
                <Text
                  ref={(node) => {
                    if (node) {
                      textNodesRef.current[point.id.toString()] = node;
                    }
                  }}
                  x={canvasCoords.x + 15}
                  y={canvasCoords.y - 15}
                  text={point.name}
                  fontSize={14}
                  fill="black"
                  onDblClick={() => startTextEdit(point.id, point.name)}
                />
              </Group>
            );
          })}
          
          {/* 選択用のTransformer */}
          <Transformer
            ref={transformerRef}
            rotateEnabled={false}
            enabledAnchors={[]}
            boundBoxFunc={(oldBox, newBox) => {
              // 位置の移動のみ許可（サイズ変更を無効化）
              return oldBox;
            }}
          />
        </Layer>
      </Stage>
    </div>
  );
};

export default DraggablePointsCanvas;
```

このコンポーネントが実現する機能：

1. **カスタム座標系**：左下を原点(0,0)とする座標系を使用
2. **ドラッグ可能な点**：ユーザーは点をドラッグして位置を変更できる
3. **名前編集**：ダブルクリックでラベルのテキストを編集可能
4. **リサイズ**：キャンバスの角や辺をドラッグしてサイズ変更（アスペクト比維持）
5. **背景画像**：任意の画像を背景に表示可能

## React-Konvaの欠点とその対処法

React-Konvaにも若干の欠点があります：

1. **学習曲線**：Konvaの独自APIと概念を学ぶ必要があります
2. **DOM操作との違い**：通常のDOM要素とは異なる操作方法が必要です
3. **参照の管理**：refs（参照）の扱いが特殊になります

しかし、これらの欠点は、コンポーネントを適切に抽象化し、再利用可能なパターンを作成することで緩和できます。例えば、上記のコンポーネントは複雑なインタラクションをカプセル化し、シンプルなAPIを提供しています。

## 使用に適したシナリオ

React-Konvaは以下のような場合に特に威力を発揮します：

- 複雑なインタラクティブダイアグラム
- ドラッグ＆ドロップインターフェース
- カスタムマップやビジュアライゼーション
- データに基づく動的グラフィックス
- 多数の要素を持つ高性能UIコンポーネント

## 結論

React-Konvaは、Canvas APIの複雑さを抽象化しながら、そのパワーをReactの宣言的なパラダイムで利用できるようにする強力なライブラリです。高度なグラフィックスやインタラクションを必要とするReactアプリケーションにおいて、React-Konvaはまさに最適な選択と言えるでしょう。

最初の学習コストはかかりますが、一度習得すれば、従来のDOM操作やSVGでは実現が難しかった複雑なUIコンポーネントを、効率的かつエレガントに実装できるようになります。