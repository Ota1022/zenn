---
title: "Raycast Extension 開発のすすめ"
emoji: "🧩"
type: "tech"
topics: ["Raycast", "TypeScript", "React"]
published: true
published_at: 2025-12-14 00:00
---

本記事は [3-shake Advent Calendar 2025](https://qiita.com/advent-calendar/2025/3-shake) 14日目の記事です。

[Raycast Advent Calendar 2025](https://qiita.com/advent-calendar/2025/raycast) でも2025年10月下旬に行われたRaycast Community Japan 主催イベントに3連続で参加した話を書きます。Raycast Extension開発やコミュニティに興味を持ったきっかけとなったイベントなので、よければ読んでください。

この記事ではRaycast Extension をローカルで作ってStoreに出すまでの手順を解説します。

## 1. Extensionを作るべき理由

### Raycastとは

![Raycast](/images/20251214/2025-12-14-3.03.02.png)

[Raycast](https://www.raycast.com/) は macOS / Windows 向けのランチャーアプリです。アプリの起動やファイル検索はもちろん、クリップボード履歴、ウィンドウ管理、スニペットなど、日常的な作業を高速化する機能が揃っています。

Raycast の真骨頂は **Extension（拡張機能）** にあります。Store には 2,000 以上の Extension が公開されており、GitHub、Notion、Slack、AWS など、様々なサービスと連携できます。

### Extension作成のメリット

Store には沢山のExtension が公開されていますが、自分で作る価値は十分にあります。

**1. 自分のワークフローに最適化できる**

既存の Extension だと機能が多すぎる、逆に足りない、UIが好みじゃない……。自分で作れば、社内ツールとの連携や、特殊なユースケースにも対応できる最強の Extension を Raycast に組み込めます。

**2. 作るハードルが低い**

Raycast Extension は **TypeScript + React** で開発します。Raycast が提供する API や UI コンポーネントが非常に充実しており、ちょっとしたコードの組み合わせで実用的な Extension が作れます。Vibe Coding であればサクッと作れます。

**3. Store公開で他のユーザーにも使ってもらえる**

作った Extension は Raycast Store に公開できます。PR レビューを経て、世界中の Raycast ユーザーに届けられます。自分が欲しかった機能は、きっと他の誰かも欲しがっているはずですし、OSS開発のような感じで他のユーザーが作った Extension にも Contribute できます。

---

実際に Extension を作って、その手軽さを体感していきましょう！

## 2. 作るもの

### 題材

ありふれた ToDoリスト を想定します。
既にStoreにもありそうな題材ですが競合する気はないので、練習用に捉えていただければ🙏

### 機能

* **Add Todo**：ToDo を追加するフォーム
* **Todos**：ToDo 一覧（検索・完了/未完了切替・削除・全削除・Markdownコピー）

### 技術スタック

* **言語**: TypeScript、React
* **ストレージ**: Raycast の LocalStorage
* **クラウド・API**: 不要、ローカルで完結

### 環境

* macOS
* Raycast
* Node.js（18+ 推奨）
* npm

## 3. サンプル用ディレクトリを作る

まず初めにディレクトリを作ります。

```bash
mkdir -p ~/Documents/raycast-extension-dev
```

## 4. Raycast から Extension の雛形を作る

### 4.1 Raycast で `Create Extension` を実行

Raycast を開き、検索窓から `Create Extension` を探します。

![Create Extension の検索結果](/images/20251214/2025-12-13-1.26.05.png)

以下の項目を入力していきます。

* Organization: `None`（個人開発の場合は None でOK）
* Extension Title: `Simple Todos`
* Description: `A minimal local todo list`
* Categories: （任意、空欄でもOK）
* Platforms: `macOS & Windows`（または `macOS` のみでもOK）
* Location: `~/Documents/raycast-extension-dev`
* Command Title: `Todos`
* Subtitle: （任意、空欄でもOK）
* Description: `Browse your todos`
* Template: **Show List**

![Create Extension フォーム（1）](/images/20251214/2025-12-13-1.41.42.png)
![Create Extension フォーム（2）](/images/20251214/2025-12-13-1.43.43.png)

埋め終わったら、右下の`Actions` -> `Crate Extension`を押すと以下の画面が表示され、
![Extension 作成完了画面](/images/20251214/2025-12-13-1.45.04.png)
`~/Documents/raycast-extension-dev/simple-todos` 配下に雛形が作成されます。

![作成された雛形のディレクトリ構造](/images/20251214/2025-12-13-1.47.01.png)

### 4.2 開発モードの起動

```bash
cd ~/Documents/raycast-extension-dev/simple-todos
npm install
npm run dev
```

`npm run dev` すると自動で Raycast が開き、作った Extension が Root Search に出てきます。

もう拡張機能が作れた感じが出てテンションが上がりますね！

![Todosコマンド](/images/20251214/2025-12-13-1.56.20.png)

## 5. コードの編集

編集するファイルは3つです。

| ファイル | 役割 |
|----------|------|
| `src/lib/todos.ts` | **データ層**：LocalStorage を使った Todo の保存・取得・削除などの処理 |
| `src/add-todo.tsx` | **Add Todo コマンド**：フォームから Todo を追加する UI |
| `src/todos.tsx` | **Todos コマンド**：Todo 一覧の表示と各種アクション |

React + TypeScript で書きます。
`@raycast/api` パッケージで `Form`、`List`、`ActionPanel` などの UI コンポーネントを提供されているので、これらを組み合わせて Extension のインターフェースを構築します。

### 5.1 `src/lib/todos.ts`（データ層）

まずはデータを扱う層を作ります。Raycast の `LocalStorage` はキーバリュー形式の永続化ストレージで、Extension ごとに隔離されています。

```bash
mkdir -p src/lib && touch src/lib/todos.ts
```

`src/lib/todos.ts` に以下を記述します。

```ts
import { LocalStorage } from "@raycast/api";
import { randomUUID } from "crypto";

const STORAGE_KEY = "simple-todos.items";

export type Todo = {
  id: string;
  title: string;
  isDone: boolean;
  createdAt: string; // ISO
};

export async function loadTodos(): Promise<Todo[]> {
  const raw = await LocalStorage.getItem<string>(STORAGE_KEY);
  if (!raw) return [];
  try {
    const parsed = JSON.parse(raw) as Todo[];
    return Array.isArray(parsed) ? parsed : [];
  } catch {
    return [];
  }
}

async function saveTodos(todos: Todo[]): Promise<void> {
  await LocalStorage.setItem(STORAGE_KEY, JSON.stringify(todos));
}

export async function addTodo(title: string): Promise<Todo> {
  const trimmed = title.trim();
  if (!trimmed) throw new Error("EMPTY_TITLE");

  const todo: Todo = {
    id: randomUUID(),
    title: trimmed,
    isDone: false,
    createdAt: new Date().toISOString(),
  };

  const todos = await loadTodos();
  todos.unshift(todo);
  await saveTodos(todos);
  return todo;
}

export async function toggleTodo(id: string): Promise<void> {
  const todos = await loadTodos();
  const next = todos.map((t) => (t.id === id ? { ...t, isDone: !t.isDone } : t));
  await saveTodos(next);
}

export async function deleteTodo(id: string): Promise<void> {
  const todos = await loadTodos();
  await saveTodos(todos.filter((t) => t.id !== id));
}

export async function clearTodos(): Promise<void> {
  await saveTodos([]);
}
```

`LocalStorage.getItem` / `setItem` で JSON 文字列として保存・取得しています。Todo の ID は `randomUUID()` で一意に生成します。各関数は `async` なので、UI 側からは `await` で呼び出します。

### 5.2 `src/add-todo.tsx`（Add Todo コマンド）

フォーム入力で Todo を追加するコマンドです。Raycast API が提供する `Form` コンポーネントを使って、ユーザー入力を受け付ける UI を構築します。

```bash
touch src/add-todo.tsx
```

`src/add-todo.tsx` に以下を記述します。

```tsx
import { Action, ActionPanel, Form, Toast, popToRoot, showToast } from "@raycast/api";
import { addTodo } from "./lib/todos";

type Values = {
  title: string;
};

export default function AddTodoCommand() {
  async function onSubmit(values: Values) {
    const title = values.title?.trim();
    if (!title) {
      await showToast({ style: Toast.Style.Failure, title: "Title is empty" });
      return;
    }

    await addTodo(title);
    await showToast({ style: Toast.Style.Success, title: "Added" });

    // Root Search に戻る（連打で追加しやすい）
    await popToRoot({ clearSearchBar: true });
  }

  return (
    <Form
      navigationTitle="Add Todo"
      actions={
        <ActionPanel>
          <Action.SubmitForm title="Add" onSubmit={onSubmit} />
        </ActionPanel>
      }
    >
      <Form.TextField id="title" title="Title" placeholder="Write a todo..." autoFocus />
    </Form>
  );
}
```

Raycast API が提供する `Form` と `Form.TextField` コンポーネントでテキスト入力フォームを作成し、`ActionPanel` と `Action.SubmitForm` で送信ボタンを配置しています。
また、`showToast` で成功/失敗のフィードバックをユーザーに伝え、`popToRoot` で Root Search に戻ることで連続して Todo を追加しやすくしています。

### 5.3 `src/todos.tsx`（Todos コマンド）

Todo の一覧表示と、完了/未完了の切り替え・削除などのアクションを提供するコマンドです。Raycast の `List` コンポーネントを使います。

既にある `src/todos.tsx` を **全部置き換え**ます。

`src/todos.tsx`

```tsx
import {
  Action,
  ActionPanel,
  Alert,
  confirmAlert,
  Icon,
  List,
  Toast,
  showToast,
} from "@raycast/api";
import { useEffect, useMemo, useState } from "react";
import { clearTodos, deleteTodo, loadTodos, toggleTodo, Todo } from "./lib/todos";

function toMarkdown(todo: Todo) {
  const box = todo.isDone ? "x" : " ";
  return `- [${box}] ${todo.title}`;
}

export default function TodosCommand() {
  const [isLoading, setIsLoading] = useState(true);
  const [todos, setTodos] = useState<Todo[]>([]);

  async function reload() {
    setIsLoading(true);
    try {
      setTodos(await loadTodos());
    } finally {
      setIsLoading(false);
    }
  }

  useEffect(() => {
    void reload();
  }, []);

  const [open, done] = useMemo(() => {
    const o: Todo[] = [];
    const d: Todo[] = [];
    for (const t of todos) {
      (t.isDone ? d : o).push(t);
    }
    return [o, d];
  }, [todos]);

  async function onToggle(id: string) {
    await toggleTodo(id);
    await showToast({ style: Toast.Style.Success, title: "Updated" });
    await reload();
  }

  async function onDelete(id: string) {
    await deleteTodo(id);
    await showToast({ style: Toast.Style.Success, title: "Deleted" });
    await reload();
  }

  async function onClearAll() {
    const ok = await confirmAlert({
      title: "Clear all todos?",
      message: "This cannot be undone.",
      primaryAction: { title: "Clear All", style: Alert.ActionStyle.Destructive },
    });
    if (!ok) return;

    await clearTodos();
    await showToast({ style: Toast.Style.Success, title: "Cleared" });
    await reload();
  }

  return (
    <List isLoading={isLoading} searchBarPlaceholder="Search todos...">
      {open.length === 0 && done.length === 0 && !isLoading ? (
        <List.EmptyView
          icon={Icon.CheckCircle}
          title="No todos"
          description='Run "Add Todo" to create your first task.'
        />
      ) : null}

      {open.length > 0 ? (
        <List.Section title="Open">
          {open.map((todo) => (
            <List.Item
              key={todo.id}
              title={todo.title}
              icon={Icon.Circle}
              actions={
                <ActionPanel>
                  <Action title="Mark as Done" icon={Icon.Checkmark} onAction={() => onToggle(todo.id)} />
                  <Action.CopyToClipboard title="Copy Markdown" content={toMarkdown(todo)} />
                  <Action
                    title="Delete"
                    icon={Icon.Trash}
                    style={Action.Style.Destructive}
                    onAction={() => onDelete(todo.id)}
                  />
                  <ActionPanel.Section>
                    <Action title="Reload" icon={Icon.ArrowClockwise} onAction={reload} />
                    <Action
                      title="Clear All"
                      icon={Icon.XmarkCircle}
                      style={Action.Style.Destructive}
                      onAction={onClearAll}
                    />
                  </ActionPanel.Section>
                </ActionPanel>
              }
            />
          ))}
        </List.Section>
      ) : null}

      {done.length > 0 ? (
        <List.Section title="Done">
          {done.map((todo) => (
            <List.Item
              key={todo.id}
              title={todo.title}
              icon={Icon.CheckCircle}
              actions={
                <ActionPanel>
                  <Action title="Mark as Open" icon={Icon.Circle} onAction={() => onToggle(todo.id)} />
                  <Action.CopyToClipboard title="Copy Markdown" content={toMarkdown(todo)} />
                  <Action
                    title="Delete"
                    icon={Icon.Trash}
                    style={Action.Style.Destructive}
                    onAction={() => onDelete(todo.id)}
                  />
                  <ActionPanel.Section>
                    <Action title="Reload" icon={Icon.ArrowClockwise} onAction={reload} />
                    <Action
                      title="Clear All"
                      icon={Icon.XmarkCircle}
                      style={Action.Style.Destructive}
                      onAction={onClearAll}
                    />
                  </ActionPanel.Section>
                </ActionPanel>
              }
            />
          ))}
        </List.Section>
      ) : null}
    </List>
  );
}
```

`List`、`List.Section`、`List.Item` を組み合わせて一覧を表示しています。

* `List.Item` の `icon` で完了/未完了を視覚的に区別（✅ / ⬜）しています。`ActionPanel` に複数の `Action` を配置し、それぞれにキーボードショートカットを割り当てています。
* データ取得には `@raycast/utils` の `useCachedPromise` を使うことで、取得と再読み込みを簡潔に実装できます。データがない時は `List.EmptyView` でメッセージを表示します。

---

### 5.4 `package.json` の commands に追加

最後に `package.json` を開き、`commands` 配列に2つ目のコマンドを追加します。

**追加前：**

```json
"commands": [
  {
    "name": "todos",
    "title": "Todos",
    "description": "Browse your todos",
    "mode": "view"
  }
]
```

**追加後：**

```json
"commands": [
  {
    "name": "todos",
    "title": "Todos",
    "description": "Browse your todos",
    "mode": "view"
  },
  {
    "name": "add-todo",
    "title": "Add Todo",
    "description": "Add a new todo",
    "mode": "view"
  }
]
```

`name` は `src/<name>.tsx` に対応するというところがポイントです。

## 6. 動作確認

`npm run dev` が動いている状態で、

1. Raycast を開く
2. `Add Todo` を実行して ToDo を追加
3. `Todos` を実行して一覧が出る
4. Action で Done/Open を切り替えられる

これで完成です 🎉🎉🎉

![Add Todo](/images/20251214/2025-12-13-2.32.06.png)

![Todos](/images/20251214/2025-12-13-2.34.39.png)

## 7. Store への公開

作った Extension を Raycast Store に公開する手順です。公開すると世界中の Raycast ユーザーがインストールできるようになります。

### 7.1 公開前の準備

#### `package.json` の確認

Store 公開には以下の設定が必要です。

| フィールド | 設定内容 |
|------------|----------|
| `author` | Raycast アカウントのユーザー名 |
| `license` | `MIT`（必須） |
| `icon` | Extension のアイコン（`assets/` 配下に配置） |

#### 命名規則

* **Extension Title**：機能が伝わる明確な名前（例：`Todo List`）
* **Command Title**：`<動詞> <名詞>` 形式（例：`Add Todo`、`Search Issues`）

詳細は公式の [Prepare an Extension for Store](https://developers.raycast.com/basics/prepare-an-extension-for-store) を参照。

### 7.2 ビルドと Lint

```bash
npm run build
npm run lint
```

エラーがあれば修正してから次へ進みます。

### 7.3 `npm run publish` で提出

```bash
npm run publish
```

このコマンドを実行すると GitHub で認証後、`raycast/extensions` リポジトリに Pull Request が自動作成されます。

### 7.4 自動チェック

PR が作成されると、GitHub Actions による CI が走ります。

| チェック項目 | 内容 |
|-------------|------|
| **Build** | `npm run build` でビルドが通るか |
| **Lint** | `npm run lint` でコードスタイルに問題がないか |
| **Dependencies** | `package-lock.json` が含まれているか、不要な依存がないか |

**Greptile** という AI レビューボットが自動でコードをチェックし、以下のような指摘をコメントします。

* **style**: コードスタイルの問題（古い設定形式、型の安全性など）
* **logic**: ロジックの問題（存在しないファイルの参照、未宣言の依存など）

### 7.5 PR テンプレートのチェックリスト

PR 作成時にチェックリストが表示されます。すべて確認してからレビューを依頼しましょう。

- [ ] [Extension guidelines](https://developers.raycast.com/basics/prepare-an-extension-for-store) を読んだ
- [ ] [Publishing documentation](https://developers.raycast.com/basics/publish-an-extension) を読んだ
- [ ] `npm run build` を実行し、ビルド結果を Raycast でテストした
- [ ] `assets` フォルダのファイルが Extension 自体で使われていることを確認した
- [ ] README で使う画像は `metadata` フォルダの外に配置した

### 7.6 自動レビューの指摘例（Greptile）

AI レビューボットからの指摘例 ~~私が過去に受けた指摘~~ です。

| 指摘内容 | 対応方法 |
|---------|---------|
| 日本語テキストが含まれている | Raycast Extension は **US English のみ** |
| 開発用スクリプトが含まれている | `scripts/` 内の開発用ファイルは削除 |
| 未使用の依存がある | `package.json` から不要な依存を削除 |
| eslint 設定が古い | `.eslintrc.js` → `eslint.config.js` 形式に移行 |
| 存在しないファイルを参照している | パスを修正するか、不要なコードを削除 |

### 7.7 レビュアーからの指摘

自動チェックを通過したら、Raycast チームメンバーが実際に Extension をテストしてフィードバックをくれます。よくある ~~私が以前受けた~~ 指摘は以下の通りです。

| 指摘内容 | 対応方法 |
|---------|---------|
| スクリーンショットを追加してほしい | `metadata/` フォルダに配置。[公式ツール](https://developers.raycast.com/basics/prepare-an-extension-for-store#screenshots)で正しいパディングを付ける |
| コメントを英語にしてほしい | Raycast Extension は **US English のみ**。コード内のコメントも英語に |
| `scripts/` フォルダは何？ | 開発用ファイルは削除するか `.gitignore` に追加。本番には含めない |
| `tsconfig.json` がデフォルト形式でない | Raycast テンプレートの設定に合わせる |
| スクリーンキャストを見せてほしい | 動作フローを録画して共有（レビュアーが動作確認しやすくなる） |
| 型定義が重複している | 同じインターフェースが複数ファイルにある場合は共通化 |

レビュアーは実際に Extension をインストールしてテストするので、動作するかだけでなく UX が良いかどうかもチェックされます。準備がやや大変なので心が折れそうになりますが、ここまで来たらレビュー通過まであと少しなので頑張りましょう！


### 7.8 マージと公開

レビューを通過すると Raycast チームがマージします。マージ後、数時間〜1日程度で Store に公開されます。

公開後は Raycast で `Manage Extensions` → 自分の Extension を検索 → `⌘ ⌥ .` でリンクをコピーして共有できます。

---

## 8. 既存 Extension への Contribute

自分の Extension を公開する以外に、既存の人気 Extension に貢献するという選択肢もあります。

Contribute する主なメリットは以下の通りです。

* 既にユーザーが多いので、小さな改善でも多くの人に届く
* コードレビューを通じて Raycast Extension 開発のベストプラクティスを学べる
* OSS 貢献の実績になる

### Contribute の手順

全 Extension は [raycast/extensions](https://github.com/raycast/extensions) リポジトリで管理されています。進行中の PR は [Pull Requests](https://github.com/raycast/extensions/pulls) で確認できます。

Raycast には Fork Extension というアクションがあり、これを使うと Contribute の準備が進みます。

1. Raycast のルート検索（⌘ + Space）で Fork Extension を実行し、対象の Extension を選ぶ

2. GitHub 上に fork が作成される

3. ローカルにコードを取り込み、修正する

4. 変更をコミットして GitHub に push

5. raycast/extensions へ Pull Request（PR）を作成して提出

OSS の定石ですが、まずは typo 修正・文言改善・エラーハンドリング みたいな小さな改善からスタートするのがおすすめです！

---

## 9. まとめ

本記事では、Raycast Extension の開発から Store 公開までの流れをToDo リストを題材に解説しました。

ぜひ自分だけの Extension を作って、日々のワークフローを加速させてください！🚀

---

## 参考

Raycast Developers Docs（公式）

* Create your first extension

  * [https://developers.raycast.com/basics/create-your-first-extension](https://developers.raycast.com/basics/create-your-first-extension)
* Manifest

  * [https://developers.raycast.com/information/manifest](https://developers.raycast.com/information/manifest)
* Storage

  * [https://developers.raycast.com/api-reference/storage](https://developers.raycast.com/api-reference/storage)
* Publish an extension

  * [https://developers.raycast.com/basics/publish-an-extension](https://developers.raycast.com/basics/publish-an-extension)
* Contribute to an extension

  * [https://developers.raycast.com/basics/contribute-to-an-extension](https://developers.raycast.com/basics/contribute-to-an-extension)
