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

## 開発手法

### TDD (テスト駆動開発) アプローチ

TDDは以下のサイクルで進める開発手法です：

1. **Red**: まず失敗するテストを書く
2. **Green**: テストが通るように最小限の実装をする
3. **Refactor**: コードをリファクタリングして改善する

#### テスト実装の順序

テストは以下の順序で実装するとよいでしょう：

1. **Assert**: まず期待する結果（アサーション）を定義
2. **Act**: 次に操作（テスト対象の処理）を定義
3. **Arrange**: 最後に準備（テスト環境のセットアップ）を定義

これは実行順序（Arrange → Act → Assert）とは逆順ですが、期待する結果から考えることで目的を明確にしてから実装できます。

#### テスト名の付け方

テスト名は「状況→操作→結果」の形式で記述します：

```text
「{状況}の場合に{操作}をすると{結果}になること」
```

例：
- "When submitting a valid form, it should save data successfully"
- "When token is invalid, it should return authentication error"

### TypeFirst アプローチ

複雑な機能開発は、実装前に型を設計することで、要件とインターフェースを明確にします：

1. まずドメインの型とインターフェースを定義
2. 型が正しいことを型チェックで確認
3. インターフェースを使ったテストコードを作成
4. テストを満たす実装を追加

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
// Server Component (Container)
import { UserProfilePresenter } from './UserProfile.presenter';

export async function UserProfile({ userId }: { userId: string }) {
  const user = await fetchUser(userId);
  
  return <UserProfilePresenter user={user} />;
}

// Client Component (Presenter)
'use client';

export function UserProfilePresenter({ user }: { user: User }) {
  // Interactive UI implementation
  return <div>{/* UI implementation */}</div>;
}
```tsx
// Server Component (Container)
import { UserProfilePresenter } from './UserProfile.presenter';

export async function UserProfile({ userId }: { userId: string }) {
  const user = await fetchUser(userId);
  
  return <UserProfilePresenter user={user} />;
}

// Client Component (Presenter)
'use client';

export function UserProfilePresenter({ user }: { user: User }) {
  // Interactive UI implementation
  return <div>{/* UI implementation */}</div>;
}
```

## コード構造

### ディレクトリ構成

```
src/
  ├── app/                     # App Router のルート
  │   ├── (routes)/            # グループ化されたルート
  │   │   ├── some-route/      # 特定のルート
  │   │   │   ├── _actions/    # ルート固有のサーバーアクション
  │   │   │   ├── _queries/    # ルート固有のデータフェッチ関数
  │   │   │   ├── _components/ # ルート固有のコンポーネント
  │   │   │   └── page.tsx     # ページコンポーネント
  │   │   └── [...]/
  │   └── layout.tsx
  ├── components/              # グローバルコンポーネント
  │   ├── features/            # 機能ごとのコンポーネント
  │   ├── ui/                  # 共通UIコンポーネント
  │   └── shared/              # 複数機能で共有するコンポーネント
  ├── lib/                     # グローバルユーティリティ
  │   ├── actions/             # 共通サーバーアクション
  │   ├── db/                  # データベース操作
  │   └── utils/               # ヘルパー関数
  ├── hooks/                   # カスタムフック
  ├── types/                   # グローバル型定義
  └── tests/                   # テストヘルパー
```

この構造の利点:

1. **ローカライズされた関連ファイル**: 特定のルートに関連するコンポーネント、アクション、データフェッチロジックは、そのルートディレクトリ内の専用サブディレクトリに配置
2. **明確な名前空間**: アンダースコアプレフィックス (`_`) により、ルート固有の実装詳細であることを明示
3. **コロケーション**: 関連するコードが近くに配置され、コンテキストの理解と保守が容易に
4. **スケーラビリティ**: 機能の増加に伴って自然に構造化が進む
5. **再利用性**: グローバルに再利用されるコンポーネントとユーティリティはルート外に配置

### アンダースコアプレフィックスのディレクトリ規約

- `_actions/`: ルート固有のサーバーアクション
- `_queries/`: ルート固有のデータフェッチ関数
- `_components/`: ルート固有のコンポーネント
- `_hooks/`: ルート固有のカスタムフック
- `_types/`: ルート固有の型定義
- `_lib/`: ルート固有のユーティリティ関数

### コンポーネントのコロケーション例

```tsx
// app/(routes)/users/[id]/_components/user-details.tsx
import { UserDetailsPresenter } from './user-details.presenter';
import { getUser } from '../_queries/get-user';

export async function UserDetails({ userId }: { userId: string }) {
  const user = await getUser(userId);
  return <UserDetailsPresenter user={user} />;
}

// app/(routes)/users/[id]/_queries/get-user.ts
import { db } from '@/lib/db';

export async function getUser(id: string) {
  return db.user.findUnique({ where: { id } });
}

// app/(routes)/users/[id]/_actions/update-user.ts
'use server';
import { revalidatePath } from 'next/cache';
import { db } from '@/lib/db';

export async function updateUser(id: string, data: UserUpdateData) {
  await db.user.update({ where: { id }, data });
  revalidatePath(`/users/${id}`);
}

// app/(routes)/users/[id]/page.tsx
import { UserDetails } from './_components/user-details';

export default function UserPage({ params }: { params: { id: string } }) {
  return <UserDetails userId={params.id} />;
}
```

### 命名規則

- **ファイル**: 
  - コンポーネント: kebab-case.tsx
  - ユーティリティ: kebab-case.ts
  - テスト: *.test.tsx
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
- ブランデッド型で型安全性を強化する（必要な場合）

```tsx
// Good example
const user: User = { id: 1, name: "John" };

// Bad example
const user = { id: 1, name: "John" } as any;

// Branded type example (for enhanced type safety)
type Branded<T, Brand> = T & { __brand: Brand };
type UserId = Branded<string, "UserId">;

function createUserId(id: string): UserId {
  // Validation logic here
  return id as UserId;
}
```

### React & Next.js

- 関数コンポーネントは `function` キーワードで宣言する（`const` + アロー関数は使用しない）
- コンポーネント内の関数はアロー関数で記述する
- サーバーコンポーネントをデフォルトとし、必要な場合のみクライアントコンポーネントを使用
- コンポーネントは単一責任の原則に従う
- サーバーアクションは明示的に分離する

```tsx
// Good example - function declaration and arrow functions
function UserProfile({ user }: UserProfileProps) {
  const handleClick = () => {
    // Event handler with arrow function
    console.log('Clicked');
  };
  
  return <button onClick={handleClick}>View Profile</button>;
}

// Bad example
const UserProfile = ({ user }: UserProfileProps) => {
  // Component defined as arrow function (avoid)
  function handleClick() {
    // Regular function declaration (avoid)
  }
  
  return <button onClick={handleClick}>View Profile</button>;
};
```

### 言語とテキスト

- アプリに表示するテキストはすべて英語で統一する
- コメントやテストケースも英語で記述する
- 翻訳が必要な場合は、別途i18nライブラリを導入する

```tsx
// Good example
function Greeting() {
  return <h1>Welcome to our application</h1>;
}

// Bad example
function Greeting() {
  return <h1>アプリケーションへようこそ</h1>;
}

// Comments in English
function calculateTotal(items: Item[]) {
  // Calculate the sum of all item prices
  return items.reduce((sum, item) => sum + item.price, 0);
}
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

// Usage example (Client Component)
'use client';
import { useAtom } from 'jotai';
import { userAtom } from '@/atoms/userAtoms';

export function UserStatus() {
  const [user] = useAtom(userAtom);
  return <div>{user ? `Logged in: ${user.name}` : 'Not logged in'}</div>;
}
```

### フォーム管理とバリデーション

- フォーム管理には `conform` と `zod` を使用する
- バリデーションスキーマは明示的に定義し、再利用可能にする
- サーバーサイドでのバリデーションを必須とする

```tsx
// Form schema definition
import { z } from 'zod';

export const userFormSchema = z.object({
  name: z.string().min(2, { message: 'Name must be at least 2 characters long' }),
  email: z.string().email({ message: 'Invalid email address' }),
  age: z.number().min(18, { message: 'You must be at least 18 years old' }),
});

// Usage in server action
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
    // Database operation
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
- 型付きのResult型パターンを使用してエラーハンドリングを強化する

```tsx
// Error handling in form component
'use client';

import { useForm } from '@conform-to/react';
import { userFormSchema } from './schema';

function UserForm() {
  const [form, fields] = useForm({
    // Configuration
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

#### Result型パターン

複雑なエラーハンドリングには、Result型パターンを活用します：

```typescript
// types/result.ts
export type Result<T, E> = 
  | { ok: true; value: T }
  | { ok: false; error: E };

export function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

export function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

// Usage example
import { ok, err, type Result } from '@/types/result';

type ApiError = 
  | { type: 'network'; message: string } 
  | { type: 'notFound'; message: string };

async function fetchData(id: string): Promise<Result<Data, ApiError>> {
  try {
    const response = await fetch(`/api/data/${id}`);
    
    if (!response.ok) {
      if (response.status === 404) {
        return err({ type: 'notFound', message: 'Resource not found' });
      }
      return err({ type: 'network', message: `HTTP error: ${response.status}` });
    }
    
    const data = await response.json();
    return ok(data);
  } catch (error) {
    return err({ 
      type: 'network', 
      message: error instanceof Error ? error.message : 'Unknown error' 
    });
  }
}

// Processing
const result = await fetchData('123');
if (result.ok) {
  // Process data
  processData(result.value);
} else {
  // Handle error
  handleError(result.error);
}
```

## パフォーマンス最適化

### サーバーコンポーネント最適化

- 静的データを利用する部分はサーバーコンポーネントで実装
- `fetch`を活用したキャッシング
- サーバーアクションの適切なキャッシュ無効化

```tsx
// Data fetching with caching
export async function ProductList() {
  // Cached by default
  const products = await fetch('https://api.example.com/products');
  
  // Cache control
  const latestProducts = await fetch('https://api.example.com/latest', {
    next: { revalidate: 60 } // Revalidate cache after 60 seconds
  });
  
  // ...
}
```

### クライアントサイド最適化

- React の `memo`、`useMemo`、`useCallback` を適切に使用
- 不要なレンダリングを防ぐ
- コンポーネントの適切な分割

```tsx
// Optimization example
'use client';
import { memo, useMemo } from 'react';

export const ExpensiveComponent = memo(function ExpensiveComponent({ data }: Props) {
  const processedData = useMemo(() => {
    return data.map(item => /* expensive computation */);
  }, [data]);
  
  return <div>{/* rendering */}</div>;
});
```

### コード分割と遅延ロード

- 必要なときだけコンポーネントをロード
- 大きなコンポーネントを分割

```tsx
// Lazy loading
import dynamic from 'next/dynamic';

const DynamicChart = dynamic(() => import('@/components/Chart'), {
  loading: () => <p>Loading chart...</p>,
  ssr: false // Only render on client-side
});
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
- テスト名は「状況→操作→結果」の形式で記述

### Presenterコンポーネントのテスト

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

### サーバーコンポーネントのテスト

サーバーコンポーネントのテストは、主に以下の方法でアプローチします：

1. **Presenterコンポーネントを重点的にテスト**
   - UIロジックの大部分はPresenterにあるため、こちらを中心にテスト

2. **Containerコンポーネントのロジックは分離してテスト**
   - データ取得ロジックを別関数に抽出し、その関数をテスト

3. **モック化**
   - `fetch`や`db`アクセスなどのサーバー側の処理はモック化

```typescript
// Data fetching logic in separate file
// lib/users.ts
export async function getUserById(id: string) {
  // Get user from database
  return db.user.findUnique({ where: { id } });
}

// components/features/user/UserProfile.tsx (Container)
import { getUserById } from '@/lib/users';

export async function UserProfile({ userId }: { userId: string }) {
  const user = await getUserById(userId);
  return <UserProfilePresenter user={user} />;
}

// Test (lib/users.test.ts)
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

// Test
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
  it('increments count when clicked', () => {
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

### コードカバレッジの確認

テスト実行後にコードカバレッジを測定し、テスト品質を確認します：

```bash
vitest run --coverage
```

カバレッジレポートを確認し、未テストのコードパスがあれば追加テストを検討します。

## 品質管理

### リンターとフォーマッター

- コミット前に常にリンターとフォーマッターを実行
- CIパイプラインに組み込む

```bash
bun run quality # Run linter, type check, and tests
```

### リファクタリングサイクル

1. コードカバレッジの確認
2. リンターとタイプチェックの実行
3. 未使用コードの特定と削除
4. パフォーマンス測定と最適化

### ドキュメンテーション

- 複雑なロジックには説明コメントを追加
- プロジェクト全体の構造やデータフローを文書化
- 型定義自体をドキュメントとして扱う

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
- [ ] コードの責任範囲は明確か
- [ ] エラーハンドリングは適切か
- [ ] コード重複はないか
- [ ] 副作用は適切に管理されているか
- [ ] デバッグ用コード（console.logなど）は削除されているか

## Git ワークフロー

TDDプロセスを補完するための効果的なGitワークフローを採用します：

1. 機能ブランチでの開発
  ```bash
  git checkout -b feature/user-authentication
  ```

2. テスト・実装・リファクタリングごとのコミット
  - テスト追加: `git commit -m "test: Add tests for user authentication"`
  - 実装: `git commit -m "feat: Implement user authentication"`
  - リファクタリング: `git commit -m "refactor: Improve authentication logic"`

3. レビュー前のセルフチェック
  ```bash
  bun run quality # Run linter, type check, and tests
  ```

4. プルリクエストの作成とレビュー

## 参考資料

- [React 19 ドキュメント](https://react.dev/)
- [Next.js 15 ドキュメント](https://nextjs.org/docs)
- [TypeScript ハンドブック](https://www.typescriptlang.org/docs/)
- [jotai ドキュメント](https://jotai.org/)
- [Drizzle ORM ドキュメント](https://orm.drizzle.team/docs/overview)
- [Vitest ドキュメント](https://vitest.dev/guide/)
