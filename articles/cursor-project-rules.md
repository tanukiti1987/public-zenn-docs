---
title: "Cursor の Project Rules 活用と改善"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['cursor','生成ai']
published: true
publication_name: "globis"
---

## 1. Project Rules とは

Cursor の Project Rules（v0.45~） は、Cursor Chat/Composer での対話において、必要なコンテキストを与えるための機能です

`Cursor > General > Project Rules` から設定できます。

![](/images/cursor-project-rules/project-rules-setting.png)

これまで、Cursor では Rules for AI, .cursorrules といったカスタムルールを用いて、プロンプトのコンテキストを与えていました。

Project Rules は、これらのカスタムルールに加え、より具体的かつ個別にコンテキストを与えることができる機能です。
もちろん、Rules for AI や .cursorrules を併用することも可能です。

### 各ルールの特徴

#### Rules for AI

Cursor のアプリケーション自体に設定するカスタムルールです。
Cursor で開くリポジトリ、ディレクトリに関わらず、常に適用されます。

ルールはテキストで書くことができます。

#### .cursorrules(非推奨)

Cursor で開くリポジトリ、ディレクトリ以下で適用されるカスタムルールです。
必要なときに自分で `.cursorrules` ファイルを作成する必要があります。

ルールはテキストで書くことができます。

ただし、[Cursor Rules for AI](https://docs.cursor.com/context/rules-for-ai#cursorrules) に記載されている通り、.cursorrules は将来的に廃止予定だそうです。
新たに .cursorrules を作成するよりは、Project Rules を使ったほうがよいでしょう。

#### Project Rules

Cursor で開くリポジトリ、ディレクトリ以下で適用されるカスタムルールであることは `.cursorrules` と同じですが、
以下の点で異なります。

- ルールはテキストおよび、ファイルのシンボルが使用できます
  - シンボルが使えるので、プロジェクト内のファイルを正確に指定することができます
- ルールが発動する条件（globs）を設定できます
  - ファイルの種類、ファイルの場所、ファイルの名前などを指定できます

ルール自体は `.cursor/rules/` 以下に保存されます。 `.mdc` という拡張子です。
定義したルールを使うかどうかは、Chat や Composer にて **自動で** 判断されます。

## 2. Project Rules の設定方法

Project Rules は `Cursor > General > Project Rules > Add new rule` もしくは `⌘(ctrl) + Shift + P > File: New Cursor Rule` から追加できます。

![](/images/cursor-project-rules/add-new-rule.png)

ファイル名を決めると Cursor 上で以下のようにファイルが見えるようになります。

![](/images/cursor-project-rules/initial-rule-view.png)

Description には、ルールの説明を自分がわかりやすいように記述します。
Globs は前述した通り、ファイルの種類、ファイルの場所、ファイルの名前などを指定できます。
メインのエリアには、ルールの内容を記述します。

とはいえ、ルールには何を書くべきか迷うケースもあると思いますので、私が現在設定しているルールを次のセクションでご紹介します。

## 3. Project Rules の活用例

### ファイルを変更したら、コマンドを実行してほしいときのルール

普段は docker で開発しているので、`bundle exec rspec ~~` が使えないということで、このルールを足しています。
spec ファイルに手をいれた場合には agent により、 RSpec を指示なしでしてほしい。というときに活用できます。

file name: `rspec-execution.mdc`
description: `RSpec 実行のルール`
globs: `**/*_spec.rb`
rule:
```markdown
## RSpec を実行する場合

---
docker compose exec app bundle exec rspec {{実行したい RSpec のファイルパス}}
---

また、RSpec が失敗し、失敗した行番号がわかる場合には以下のコマンドを実行することで限定的に RSpec を走らせることができる

---
docker compose exec app bundle exec rspec {{実行したい RSpec のファイルパス}}:{{失敗したRSpec の行番号}}
---
```

### ディレクトリ構成やコーディングルールを言語化し、コンテキストを理解しやすくするルール

file name: `frontend-coding-rule.mdc`
description: `フロントエンドにおけるベストプラクティス`
globs: `packages/**/*.*`
rule:
```markdown
以下は、フロントエンドの開発において、機能ごとに整理されたコンポーネントやロジックを管理するためのベストプラクティスを反映しています。

### 1. **ディレクトリ構造の基本方針**
- **機能ごとの分割**: 各機能は独立したディレクトリに分割され、その中にコンポーネント、フック、操作（mutation/query）、バリデーション、ルーティングなどを含む。
- **責務の分離**: UI層（表示）とロジック層（データ取得・処理）を明確に分離し、コンポーネントをシンプルに保つ。
- **共通化可能な処理の抽出**: 複数の機能で利用される共通処理は、`shared` ディレクトリに配置し、再利用性を高める。

### 2. **ディレクトリ構成の詳細**
#### 2.1. **機能ディレクトリ**
- **`features/foo/{NewFeature}`**: 管理者向けの{NewFeature}機能を提供するディレクトリ。
  - **`components`**: 機能固有のUIコンポーネントを配置。
    - **`Header`**: ヘッダー関連のコンポーネント（例: `Header.tsx`, `HeaderBreadcrumb.tsx`）。
    - **`Modal`**: モーダル関連のコンポーネント（例: `ArchiveNewFeatureModal.tsx`）。
  - **`domain`**: ドメインロジックを配置。
    - **`*.ts`**: バリデーションやビジネスロジックを実装。
    - **`*.spec.ts`**: バリデーションやロジックのテストを実装。

#### 2.2. **共通ディレクトリ**
- **`shared`**: 複数の機能で利用される共通コンポーネントやロジックを配置。
  - **`components`**: 共通UIコンポーネント（例: `Table.tsx`, `Alert.tsx`）。
  - **`hooks`**: 共通フック（例: `useSelectedItemIds.ts`）。
  - **`utils`**: ユーティリティ関数（例: `breadcrumb.ts`, `path.ts`）。

#### 2.3. **テストディレクトリ**

### 3. **命名規則**
- **ディレクトリ名**: 機能や役割が明確に分かる命名（例: `NewFeatureListItemPage`, `Header`）。
- **ファイル名**: 処理や役割が分かる命名（例: `useTitleForm.ts`, `NewFeatureListItemsTable.tsx`）。
- **コンポーネント名**: パーツ名を先頭に配置（例: `HeaderBreadcrumb`, `NewFeatureDropdown`）。

### 4. **テストと品質管理**
- **テストの範囲**: 主要なシナリオを網羅し、ハッピーパスを中心にテストを実施。
- **バリデーション**: zodのスキーマを別ファイルで管理し、必ずspecを作成。
- **Storybook**: コンポーネントのストーリーを作成し、UIの動作を確認。

### 5. **ルーティング**
- **ルーティングファイル**: `routes` ディレクトリに配置し、React Routerを使用してルーティングを管理。
- **ルートコンポーネント**: `index.tsx` をエントリーポイントとして使用し、モジュールのカプセル化を実現。

### 6. **GraphQLクエリ/ミューテーション**
- **操作の分離**: `operations` ディレクトリにMutationとQueryを分けて配置。
- **型定義**: GraphQLの型定義は `@generated/api` からインポートし、型安全性を確保。

### 7. **UIコンポーネントの構造**
- **Atomic Design**: コンポーネントを `atoms`, `molecules`, `organisms`, `templates` に分割し、再利用性を高める。
- **表示とロジックの分離**: `*.ui.tsx` と `*.container.tsx` に分離し、責務を明確化。

### 8. **エラーハンドリング**
- **エラーメッセージ**: ユーザーに表示するエラーメッセージは、`constants` ディレクトリに定義。
- **エラーハンドリング**: GraphQLのエラーハンドリングは、`operations` ディレクトリ内で実施。

### 9. **状態管理**
- **ローカルステート**: コンポーネント内での状態管理は `useState` や `useReducer` を使用。
- **グローバルステート**: 必要に応じてContextやRecoilを使用。

### 10. **ドキュメント**
- **README**: 各ディレクトリにREADMEを配置し、機能や使用方法を記載。
- **コメント**: コード内に適切なコメントを記載し、可読性を向上。

この構成は、機能ごとに整理されたコードベースを維持し、開発者が効率的に作業を進めるためのガイドラインとして活用できます。
```

### エージェントによる PullRequest を自動作成するときのルール

file name: `create-pullrequest.mdc`
description: `pull request作成`
globs: `なし`
rule:
```markdown
## pull request作成手順

### 必須前提条件
- Issue番号の確認
  - Issueのリンクが提供されていない場合は、必ずユーザーに「関連するIssueのリンクはありますか？」と確認する
  - Issueが存在しない場合は、その旨をPRの説明に明記する

### 差分の確認
- {{マージ先ブランチ}}は特に指示がなければ main とする
- `git diff origin/{{マージ先ブランチ}}...HEAD | cat` でマージ先ブランチとの差分を確認

### descriptionに記載するリンクの準備
- Issueのリンクを確認（必須前提条件で確認済みであること）

### Pull Request 作成とブラウザでの表示
- 以下のコマンドでpull requestを作成し、自動的にブラウザで開く
- PRタイトルおよびPRテンプレートはマージ先との差分をもとに適切な内容にする
- 指示がない限りDraftでpull requestを作成
- `{{PRテンプレートを1行に変換}}`の部分はPRテンプレートの内容を`\n`で改行表現した1行の文字列
- 各セクションを明確に区分
- 必要な情報を漏れなく記載

---
git push origin HEAD && \
echo -e "{{PRテンプレートを1行に変換}}" | \
gh pr create --draft --title "{{PRタイトル}}" --body-file - && \
gh pr view --web
---

#### PRテンプレート

@PULL_REQUEST_TEMPLATE.md からテンプレート内容を取得すること
```

## 4. 作成した Rule を改善方法

上記の例を眺めて、この内容を自分で書くのは流石に難しい。自分のプロジェクトケースではどう書くかわからない方もいるでしょう。
このセクションでは、各プロジェクトでのルールの作り方や改善方法を紹介します。

### カスタムルールのベースの作り方

とりわけ、先の例に示した「ディレクトリ構成やコーディングルールを言語化し、コンテキストを理解しやすくするルール」は人の手で書くのは骨が折れます。

そんなときは、 Cursor の Chat でルールを作成する相談を投げて、たたき台を作ってもらうのがよいでしょう。

```
Cursor の Project Rule を作りたいです。
ルールの内容は、このリポジトリにおけるフロントエンドのコーディングルールです。

{{フロントエンドコードが置いてあるディレクトリをシンボルで指定}} 以下を分析し、ルールを作成してください。
```

得られた出力から必要な部分を追記、不要な部分を削除して、自分のプロジェクトに合ったルールを作成することをオススメします。

### カスタムルールの改善の仕方

上記で作成したルールを動かしてみて、意図通りにルールが動いていない事象に遭遇することも多いと思います。
そんなときも、Chat で相談がオススメです。

```
{{mdcファイルへのシンボル}} のルールについてわかりにくいところや、改善したほうがよいポイントはありますか？
```

こう聞いてあげることで、ルールの改善点や、より具体的なルールの追加を提案してくれるので、ルールをブラッシュアップしていくことが可能です。

## 5. ちょっとした Tips

Project Rules を追加していっても、ルールが呼ばれるときと、呼ばれないときがあり、ルールが効いているのか、効いていないか。わからないときがあります。

そんなときには、ルールの先頭で無意味な言葉を叫ばせる。という方法があります。

[Memory and Confidence Checks 🧠](https://docs.cline.bot/improving-your-prompting-skills/prompting#memory-and-confidence-checks)

ルールの先頭に

```
まず、このファイルを参照したら、「YAAAARRRR!」と叫んでください。
```

と書くだけです。 Chat/Composer でAIから YAAAARRRR! という言葉が出力された場合には、ルールのコンテキストが効いていることを意味します 👍

## 6. まとめ

今回は Project Rules について、記事を書かせていただきました。
Cursor は素の状態で使うよりも、Rules for AI, .cursorrules, Project Rules を組み合わせて使いつつ、プロジェクトの状態に沿ったルールを組んであげることで開発効率は爆発的に上がります。

ぜひ、この記事を読んだら、一つ Project Rule を作ってみてください。
