# Storybook v8 における最適なストーリー作成方法

## 前提条件

- **グローバル設定 (.storybook/preview.ts)**  
  下記のグローバル設定では、全てのプロパティが自動でControlsに反映されます。  
  - `parameters.actions.argTypesRegex` により、`on` で始まる関数型プロップ（例: `onClick`）は自動的にActionsに連携され、Controlsではなくイベントログとして扱われます。  
  - `parameters.controls` により、Controlsパネルは初期状態で展開され、プロパティはアルファベット順にソートされます。  
  - `parameters.docs.autodocs` を有効化することで、型情報やJSDocコメントなどから自動でドキュメントが生成されます。  
  - グローバル設定の `argTypesRegex: '.*'` により、型定義がある全てのプロパティがControlsに反映されます。

  ```ts
  // .storybook/preview.ts
  import type { Preview } from '@storybook/react';

  const preview: Preview = {
    parameters: {
      actions: { argTypesRegex: '^on[A-Z].*' },
      controls: {
        expanded: true,
        sort: 'alpha',
      },
      docs: {
        autodocs: true,
      },
    },
    argTypesRegex: '.*',
  };

  export default preview;
  ```

- コンポーネントは型定義を含むため、onで始まる関数型プロップは自動連携され、Actionsとして動作します。

---

## 生成AI依頼用プロンプト例

以下のプロンプトを生成AIに渡すことで、作成済みのコンポーネントコードを元に最小記述のストーリーファイル（.stories.tsx）を自動生成できます。必要に応じてStoryは適宜増やしても良いという旨も含んでいます。

```
【生成AI依頼用プロンプト】

■ 前提条件
- グローバル設定 (.storybook/preview.ts) は以下の通りとなっており、全てのプロパティは自動でControlsに反映されます.
  ```ts
  // .storybook/preview.ts
  import type { Preview } from '@storybook/react';

  const preview: Preview = {
    parameters: {
      actions: { argTypesRegex: '^on[A-Z].*' },
      controls: {
        expanded: true,
        sort: 'alpha',
      },
      docs: {
        autodocs: true,
      },
    },
    argTypesRegex: '.*',
  };

  export default preview;
  ```
- コンポーネントは型定義を含んでおり、onで始まる関数型プロップはActionsに自動連携されています。

■ 入力欄
{対象のコンポーネントコード}

【入力コンポーネント例】
```tsx
// MyButton.tsx
import React from 'react';

export interface MyButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  label?: string;
  primary?: boolean;
  backgroundColor?: string;
  size?: 'small' | 'medium' | 'large';
}

export const MyButton: React.FC<MyButtonProps> = ({
  label = 'Button',
  primary = false,
  backgroundColor,
  size = 'medium',
  ...props
}) => {
  const mode = primary ? 'storybook-button--primary' : 'storybook-button--secondary';
  return (
    <button
      type="button"
      className={`storybook-button storybook-button--${size} ${mode}`}
      style={backgroundColor ? { backgroundColor } : {}}
      {...props}
    >
      {label}
    </button>
  );
};
```

■ 出力コンポーネント例
```tsx
// MyButton.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { MyButton } from './MyButton';

const meta: Meta<typeof MyButton> = {
  title: 'Example/MyButton',
  component: MyButton,
};

export default meta;

export const Primary: StoryObj<typeof MyButton> = {
  args: {
    primary: true,
    label: 'Primary Button',
  },
};

export const Secondary: StoryObj<typeof MyButton> = {
  args: {
    primary: false,
    label: 'Secondary Button',
  },
};

// ※ 必要に応じてStoryは適宜増やしても良い。
```
``` 

---

この解説とプロンプト例を利用することで、生成AIに対して統一されたグローバル設定下でのストーリー生成を依頼でき、各ストーリー自体は最小限の記述に留めながらも、Controls、Autodocs、Actionsなどのリッチな機能を享受できます。