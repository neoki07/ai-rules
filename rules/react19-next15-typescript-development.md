# React 19 / Next.js 15 / TypeScript 開発ガイドライン

## 概要

このドキュメントは、React 19、Next.js 15、TypeScriptを使用したプロジェクト開発のためのガイドラインです。サーバーコンポーネントとサーバーアクションを活用し、Container/Presenterパターンを採用したTDD開発を前提としています。

## 技術スタック

- **TypeScript**: 5.x
- **React**: 19.0.0
- **Next.js**: 15.1.x
- **状態管理**: jotai 2.x
- **フォーム管理**: conform + zod
- **スタイリング**: TailwindCSS 4.x
- **ORM**: Drizzle ORM
- **認証**: Clerk
- **テスト**: Vitest (ユニットテスト)
- **リンター**: ESLint 9.x
- **フォーマッター**: Prettier 3.x

## アーキテクチャ

### 基本方針

- Next.jsのApp Routerを使用
- API Routeは使わず、サーバーコンポーネントとサーバーアクションで実装
- Container/Presenterパターンでのコンポーネント設計
- TDDアプローチによる段階的な開発
- クライアントコンポーネントは最小限に抑え、ユーザーインタラクションが必要な部分のみに使用

### コンポーネント構造

```
components/
  ├── features/     # 機能ごとのコンポーネント
  │   └── {feature}/
  │       ├── {Component}.tsx              # Container (データ取得・状態管理)
  │       ├── {Component}.presenter.tsx    # Presenter (UI表示のみ)
  │       └── {Component}.test.tsx         # テスト
  ├── ui/           # 共通UIコンポーネント
  └── shared/       # 複数機能で使用するコンポーネント
```

### データフロー

1. サーバーコンポーネント（Container）がデータを取得
2. Presenterコンポーネントに必要なデータを渡す
3. ユーザー操作はサーバーアクションまたはクライアントコンポーネントのハンドラーで処理

### Server Components vs Client Components

- **サーバーコンポーネント**: データ取得、初期表示、SEOが必要な要素
- **クライアントコンポーネント**: インタラクティブなUI、状態を持つコンポーネント
- **原則**: クライアントコンポーネントはユーザーインタラクションが必要なUIパーツのみに限定する

```tsx
// サーバーコンポーネント (Container)
import { UserProfilePresenter } from './UserProfile.presenter';

export async function UserProfile({ userId }: { userId: string }) {
  const user = await fetchUser(userId);
  
  return <UserProfilePresenter user={user} />;
}

// クライアントコンポーネント (Presenter)
'use client';

export function UserProfilePresenter({ user }: { user: User }) {
  return <div>...</div>;
}
```

## コード構造

### ディレクトリ構成

```
src/
  ├── app/               # Appルーティング
  │   └── (routes)/      # ルート定義
  ├── components/        # コンポーネント
  ├── lib/               # ユーティリティ
  │   ├── actions/       # サーバーアクション
  │   ├── db/            # データベース処理
  │   └── utils/         # ヘルパー関数
  ├── hooks/             # カスタムフック
  ├── types/             # 型定義
  └── test/              # テスト用ヘルパー
```

### 命名規則

- **ファイル**: 
  - コンポーネント: PascalCase.tsx
  - ユーティリティ: camelCase.ts
  - テスト: *.test.tsx または *.spec.tsx
  - Presenter: *.presenter.tsx

- **型**:
  - インターフェース: PascalCase (例: `UserData`)
  - 型エイリアス: PascalCase (例: `ButtonVariant`)
  - Props: コンポーネント名 + Props (例: `UserProfileProps`)

## コーディング規約

### TypeScript

- 暗黙的なanyを使用しない (`noImplicitAny: true`)
- 厳格な型チェックを有効化する (`strict: true`)
- 型アサーションは最小限に抑える

```tsx
// 良い例
const user: User = { id: 1, name: 'John' };

// 避けるべき例
const user = { id: 1, name: 'John' } as any;
```

### React & Next.js

- 関数コンポーネントは `function` キーワードで宣言する（`const` + アロー関数は使用しない）
- コンポーネント内の関数はアロー関数で記述する
- サーバーコンポーネントをデフォルトとし、必要な場合のみクライアントコンポーネントを使用
- コンポーネントは単一責任の原則に従う
- サーバーアクションは明示的に分離する

```tsx
// 良い例 - function宣言とアロー関数
function UserProfile({ user }: UserProfileProps) {
  const handleClick = () => {
    console.log('Clicked');
  };
  
  return <button onClick={handleClick}>View Profile</button>;
}

// 避けるべき例
const UserProfile = ({ user }: UserProfileProps) => {
  function handleClick() {
    console.log('Clicked');
  }
  
  return <button onClick={handleClick}>View Profile</button>;
};
```

### Container/Presenterパターン

- **Container**:
  - データフェッチング
  - ビジネスロジック
  - サーバーアクションの呼び出し
  - Presenterへのデータ受け渡し

- **Presenter**:
  - UIのレンダリングのみ
  - 受け取ったpropsの表示
  - UIイベントの処理
  - 最小限の状態管理

```tsx
// Container (UserProfile.tsx)
import { UserProfilePresenter } from './UserProfile.presenter';

export async function UserProfile({ userId }: { userId: string }) {
  const user = await db.user.findUnique({ where: { id: userId } });
   if (!user) return <div>User not found</div>;
  
  return <UserProfilePresenter user={user} />;
}

// Presenter (UserProfile.presenter.tsx)
'use client';

interface UserProfilePresenterProps {
  user: User;
}

export function UserProfilePresenter({ user }: UserProfilePresenterProps) {
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### 状態管理 (jotai)

- アトムは機能ごとにファイルを分割
- 派生アトムを活用して計算済みの状態を管理
- サーバーコンポーネントとの連携を考慮

```tsx
// atoms/userAtoms.ts
import { atom } from 'jotai';

export const userAtom = atom<User | null>(null);
export const isLoggedInAtom = atom(get => get(userAtom) !== null);

// 使用例 (クライアントコンポーネント)
'use client';
import { useAtom } from 'jotai';
import { userAtom } from '@/atoms/userAtoms';

export function UserStatus() {
  const [user] = useAtom(userAtom);
    return <div>{user ? `Logged in: ${user.name}` : 'Not logged in'}</div>;
}
```

### 言語とテキスト

- アプリに表示するテキストはすべて英語で統一する
- コメントやテストケースも英語で記述する

```tsx
// 良い例
function Greeting() {
  return <h1>Welcome to our application</h1>;
}

// 避けるべき例
function Greeting() {
  return <h1>アプリケーションへようこそ</h1>;
}
```

## パフォーマンス最適化

### サーバーコンポーネント最適化

- 静的データを利用する部分はサーバーコンポーネントで実装
- `fetch`を活用したキャッシング
- サーバーアクションの適切なキャッシュ無効化

```tsx
export async function ProductList() {
  const products = await fetch('https://api.example.com/products');
  
  const latestProducts = await fetch('https://api.example.com/latest', {
    next: { revalidate: 60 }
  });
  
  // ...
}
```

### クライアントサイド最適化

- React の `memo`、`useMemo`、`useCallback` を適切に使用
- 不要なレンダリングを防ぐ
- コンポーネントの適切な分割

```tsx
'use client';
import { memo, useMemo } from 'react';

export const ExpensiveComponent = memo(function ExpensiveComponent({ data }: Props) {
  const processedData = useMemo(() => {
    return data.map(item => /* 重い処理 */);
  }, [data]);
  
  return <div>...</div>;
});
```

### フォーム管理とバリデーション

- フォーム管理には `conform` と `zod` を使用する
- バリデーションスキーマは明示的に定義し、再利用可能にする
- サーバーサイドでのバリデーションを必須とする

```tsx
// フォームスキーマの定義
import { z } from 'zod';

export const userFormSchema = z.object({
  name: z.string().min(2, { message: 'Name must be at least 2 characters long' }),
  email: z.string().email({ message: 'Invalid email address' }),
  age: z.number().min(18, { message: 'You must be at least 18 years old' }),
});

// サーバーアクション内での使用
'use server';

import { conform } from '@conform-to/react';
import { parse } from '@conform-to/zod';

export async function createUser(formData: FormData) {
  const submission = parse(formData, { schema: userFormSchema });
  
  if (!submission.value) {
    return {
      status: 'error',
      errors: submission.error,
    };
  }
  
  try {
    // データベース処理
    await db.user.create({ data: submission.value });
    return { status: 'success' };
  } catch (error) {
    return {
      status: 'error',
      errors: {
        _form: ['Failed to create user. Please try again.'],
      },
    };
  }
}
```

### エラーハンドリング

- フォーム送信時のエラーは、ユーザー向けの適切なメッセージに変換して表示する
- システムエラーとユーザーエラーを明確に区別する
- エラーメッセージは具体的で解決方法を示唆するものにする

```tsx
// フォームコンポーネントでのエラー表示
'use client';

import { useForm } from '@conform-to/react';
import { userFormSchema } from './schema';

function UserForm() {
  const [form, fields] = useForm({
    onValidate({ formData }) {
      return parse(formData, { schema: userFormSchema });
    },
  });
  
  return (
    <form id={form.id} onSubmit={form.onSubmit}>
      <div>
        <label htmlFor={fields.name.id}>Name</label>
        <input name={fields.name.name} id={fields.name.id} />
        {fields.name.errors && (
          <div className="error-message">{fields.name.errors}</div>
        )}
      </div>
      
      {/* Other fields */}
      
      {fields.form.errors && (
        <div className="form-error">
          {fields.form.errors.map((error) => (
            <p key={error}>{error}</p>
          ))}
        </div>
      )}
      
      <button type="submit">Submit</button>
    </form>
  );
}
```

## テスト戦略

### テストの種類

- **ユニットテスト**: 個別の関数やコンポーネントのテスト
- **統合テスト**: 複数のコンポーネントやモジュールの連携テスト

### TDDアプローチ

1. 失敗するテストを書く
2. テストを通過するコードを書く
3. コードをリファクタリングする

### テスト実装のルール

- テストファイルはテスト対象と同じディレクトリに配置
- コンポーネント名と同じ名前で `.test.tsx` 拡張子を使用

### Presenterコンポーネントのテスト

```tsx
```tsx
// UserProfile.presenter.test.tsx
import { render, screen } from '@testing-library/react';
import { UserProfilePresenter } from './UserProfile.presenter';

describe('UserProfilePresenter', () => {
  it('displays user name and email', () => {
    const user = {
      id: '1',
      name: 'Test User',
      email: 'test@example.com',
    };
    
    render(<UserProfilePresenter user={user} />);
    
    expect(screen.getByText('Test User')).toBeInTheDocument();
    expect(screen.getByText('test@example.com')).toBeInTheDocument();
  });
});
```
```

### サーバーコンポーネントのテスト

サーバーコンポーネントのテストは、主に以下の方法でアプローチします：

1. **Presenterコンポーネントを重点的にテスト**
   - UIロジックの大部分はPresenterにあるため、こちらを中心にテスト

2. **Containerコンポーネントのロジックは分離してテスト**
   - データ取得ロジックを別関数に抽出し、その関数をテスト

3. **モック化**
   - `fetch`や`db`アクセスなどのサーバー側の処理はモック化

```typescript
// データ取得ロジックを分離した例
// lib/users.ts
export async function getUserById(id: string) {
  return db.user.findUnique({ where: { id } });
}

// components/features/user/UserProfile.tsx (Container)
import { getUserById } from '@/lib/users';

export async function UserProfile({ userId }: { userId: string }) {
  const user = await getUserById(userId);
  return <UserProfilePresenter user={user} />;
}

// テスト (lib/users.test.ts)
import { describe, it, expect, vi } from 'vitest';
import { getUserById } from './users';
import { db } from '@/lib/db';

vi.mock('@/lib/db', () => ({
  db: {
    user: {
      findUnique: vi.fn(),
    },
  },
}));

test('getUserById returns user data', async () => {
  const mockUser = { id: '1', name: 'Test User' };
  db.user.findUnique.mockResolvedValue(mockUser);
  
  const result = await getUserById('1');
  expect(result).toEqual(mockUser);
});
```

### サーバーアクションのテスト

サーバーアクションは関数としてテスト可能です：

```typescript
// lib/actions/updateUser.ts
'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function updateUser(id: string, data: UserUpdateData) {
  const result = await db.user.update({
    where: { id },
    data,
  });
  
  revalidatePath(`/users/${id}`);
  return result;
}

// テスト
import { describe, it, expect, vi } from 'vitest';
import { updateUser } from './updateUser';
import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

vi.mock('@/lib/db', () => ({
  db: {
    user: {
      update: vi.fn(),
    },
  },
}));

vi.mock('next/cache', () => ({
  revalidatePath: vi.fn(),
}));

test('updateUser updates user and revalidates path', async () => {
  const mockUser = { id: '1', name: 'Updated Name' };
  db.user.update.mockResolvedValue(mockUser);
  
  const result = await updateUser('1', { name: 'Updated Name' });
  
  expect(db.user.update).toHaveBeenCalledWith({
    where: { id: '1' },
    data: { name: 'Updated Name' },
  });
  expect(revalidatePath).toHaveBeenCalledWith('/users/1');
  expect(result).toEqual(mockUser);
});
```

### イベントハンドラのテスト

```typescript
// ButtonCounter.presenter.tsx
'use client';
import { useState } from 'react';

export function ButtonCounterPresenter() {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Click count: {count}
    </button>
  );
}

// ButtonCounter.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { ButtonCounterPresenter } from './ButtonCounter.presenter';

describe('ButtonCounterPresenter', () => {
  it('increments count on click', () => {
    render(<ButtonCounterPresenter />);
    
    const button = screen.getByRole('button');
    expect(button).toHaveTextContent('Click count: 0');
    
    fireEvent.click(button);
    expect(button).toHaveTextContent('Click count: 1');
    
    fireEvent.click(button);
    expect(button).toHaveTextContent('Click count: 2');
  });
});
```

### コンポーネントテストの命名規則

テストの説明文（describe/itブロック）は英語で記述し、何をテストしているかを明確に示す：

```tsx
// UserProfile.test.tsx
import { render, screen } from '@testing-library/react';
import { UserProfilePresenter } from './UserProfile.presenter';

describe('UserProfilePresenter', () => {
  it('displays user name and email', () => {
    const user = {
      id: '1',
      name: 'Test User',
      email: 'test@example.com',
    };
    
    render(<UserProfilePresenter user={user} />);
    
    expect(screen.getByText('Test User')).toBeInTheDocument();
    expect(screen.getByText('test@example.com')).toBeInTheDocument();
  });
});
```

## コードレビューチェックリスト

- [ ] コンポーネントがContainer/Presenterパターンに準拠しているか
- [ ] サーバーコンポーネントとクライアントコンポーネントの使い分けは適切か
- [ ] クライアントコンポーネントが最小限に抑えられているか
- [ ] コンポーネントが`function`キーワードで宣言されているか
- [ ] イベントハンドラがアロー関数で記述されているか
- [ ] アプリ内のテキストがすべて英語で統一されているか
- [ ] コメントとテストが英語で記述されているか
- [ ] フォームバリデーションが適切に実装されているか
- [ ] エラーメッセージがユーザー向けに適切に表示されているか
- [ ] 厳密な型付けがされているか
- [ ] テストは書かれているか、そして十分か
- [ ] パフォーマンス最適化が考慮されているか
- [ ] リンターやフォーマッターのエラーがないか
- [ ] 命名規則に従っているか
- [ ] コメントは適切か

## 参考資料

- [React 19 ドキュメント](https://react.dev/)
- [Next.js 15 ドキュメント](https://nextjs.org/docs)
- [TypeScript ハンドブック](https://www.typescriptlang.org/docs/)
- [jotai ドキュメント](https://jotai.org/)
- [Drizzle ORM ドキュメント](https://orm.drizzle.team/docs/overview)
- [Vitest ドキュメント](https://vitest.dev/guide/)
