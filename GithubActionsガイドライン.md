# GitHub Actions ワークフロー設定ガイドライン

## 概要

GitHub Actions ワークフローの品質・セキュリティ・メンテナンス性を統一するための設定ガイドライン。

## 1. パーミッション設定

### Organization / リポジトリレベルの設定

#### 方針

`Read repository contents and packages permissions` に設定する。

#### 理由

`Read and write permissions` にしてパーミッションブロックが設定されていないワークフローがある場合、不用意に広範囲な権限が付与されセキュリティリスクが発生するため。
ワークフローでも権限を設定する方針だが、設定漏れなどが発生しても問題ないように多段的に防衛する。

### 個別ワークフローのパーミッション設定

#### 方針

トップレベルの `permissions` ブロックで最小限のパーミッションを設定する。

#### 理由

セキュリティリスクを最小限に抑えるため。
Organization / リポジトリレベルでも設定するが、設定漏れなどが発生しても問題ないように多段的に防衛する。

#### 実装方法

```yaml
permissions:
  contents: read
```

## 2. ステップ名

### 方針

各ステップには日本語でわかりやすい名前を必ず付けること。

### 理由

視認性を向上し、失敗時や調査時などに一目でどこでエラーが発生しているか分かるようにするため。

### 実装方法

```yaml
- name: リポジトリのチェックアウト
  uses: actions/checkout@v4
```

## 3. pnpm/action-setup

### pnpm バージョン

#### 方針

`package.json` の `packageManager` フィールドのバージョンを使用すること。

#### 理由

ワークフローと開発環境の pnpm バージョンを統一し、バージョン違いによる動作が異ならないようにするため。また設定を DRY に保ち、設定の乖離が起こらないようにするため。

#### 実装方法

```yaml
- name: pnpm のセットアップ
  uses: pnpm/action-setup@v4
  # version は省略（package.json の packageManager を使用）
```

## 4. actions/setup-node

### Node.js バージョン

#### 方針

`.node-version` ファイルを参照すること。

#### 理由

ワークフローと開発環境の Node.js バージョンを統一し、バージョン違いによる動作が異ならないようにするため。また設定を DRY に保ち、設定の乖離が起こらないようにするため。

#### 実装方法

```yaml
- name: Node.js のセットアップ
  uses: actions/setup-node@v4
  with:
    node-version-file: .node-version
```

## 5. 重複実行防止

### 方針

重複実行を防ぐため、原則的に `concurrency` を設定し、設定しない場合はワークフローファイルにその理由をコメントで残すこと。

### 理由

直ぐに Push し直して古いコードでのワークフローが不要になった場合に最後まで実行されるなどの課金時間の無駄遣いを防止するため。

### 実装方法

```yaml
concurrency:
  group: <workflow-name>-${{ github.ref }}
  cancel-in-progress: true
```

## 6. タイムアウト設定

### 方針

各ジョブには最低でも 10 分のタイムアウトを設定すること。

### 理由

何らかの理由でジョブが終わらない場合などに課金時間の無駄遣いを防止するため。

### 実装方法

```yaml
jobs:
  build:
    timeout-minutes: 10
```

## 7. 不要なワークフローのスキップ

### 方針

関係のあるファイルが変更された場合にのみ実行されるようにする。
例： web アプリのテストは apps/web と apps/web が依存している packages/database が変更された場合に実行する

### 理由

- GitHub Actions は実行時間で課金されるため、不必要な実行を削減するため
- 無関係なワークフロー実行によるマージ遅延を防ぎ、開発効率を低下させないため

### 実装方法

ファイル変更の検出方法は、ジョブがマージの必須条件に指定されているかどうかで使い分ける。
turborepo を使用している場合は、後述の `--affected` を使用する。

#### マージの必須条件に指定されていないジョブの場合

GitHub Actions の標準機能 `paths-ignore` を使用してパスを指定する。

```yaml
on:
  push:
    branches: [develop, main]
    paths-ignore:
      - "docs/**"
      - "**/*.md"
  pull_request:
    branches: ["**"]
    paths-ignore:
      - "docs/**"
      - "**/*.md"
```

#### マージの必須条件に指定されているジョブの場合

`paths-ignore` に設定したパスではワークフロー自体が起動しないため、マージ必須のジョブが pass とならず、マージできなくなる。

そのため、`dorny/paths-filter` を使用してステップレベルの制御を行う。

```yaml
jobs:
  need-ci:
    runs-on: ubuntu-latest
    steps:
      - name: チェックアウト
        uses: actions/checkout@v4

      - name: 変更ファイルの検出
        uses: dorny/paths-filter@v2
        id: filter
        with:
          predicate-quantifier: every # フィルターの条件を AND 条件にするために必要。 default は same で or 条件
          filters: |
            web:
              - 'apps/web/**'
              - '!**/*.md'

      - name: web のテスト実行
        if: ${{ steps.filter.outputs.web == 'true' }}
        run: test
```

#### turborepo を使用している場合

`turbo run <task> --affected` を使用する。
変更されたパッケージと、それに依存しているパッケージのタスクのみが実行される。

パスの指定が不要なため、ワークフローと turborepo の依存グラフでパスを二重管理せずに済み、パッケージの追加や依存関係の変更にも自動で追従する。

`paths-ignore` や `dorny/paths-filter` と異なりワークフロー自体は常に起動するため、マージの必須条件に指定するジョブでもそのまま使用できる。

```yaml
- name: テストの実行
  run: pnpm turbo run test --affected
  env:
    # push では既定の比較先（main）と HEAD が一致し差分ゼロになるため push 前のコミットを基準にする
    TURBO_SCM_BASE: ${{ github.event_name == 'push' && github.event.before || format('origin/{0}', github.base_ref) }}
```

変更されたパッケージの判定には全コミット履歴が必要なため、チェックアウト時に `fetch-depth: 0` を指定する。
判定はファイル名の差分しか参照しないため、`filter: blob:none` でファイルの中身を遅延取得にして取得量を抑える。

```yaml
- name: リポジトリのチェックアウト
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
    filter: blob:none
```

なお `pnpm install` までは常に実行されるため、ドキュメントのみの変更でも数分の実行時間が発生する。
これを避けたい場合は `paths-ignore` / `dorny/paths-filter` と併用する。

## 8. サードパーティ Actions の SHA ピン留め

### 方針

サードパーティ Actions の指定には、`@v4` などのタグやブランチではなく、コミット SHA 形式での指定を行うこと。

### 理由

`@v4` などのタグ方式やブランチ指定の場合、対象の Action のリポジトリが悪意のあるコードに置き換えられた際に、その悪意のあるコードを実行してしまうリスクがある。
コミット SHA の場合は immutable なため、参照先の変更による影響を減らせる。

### 実装方法

#### ローカルでの追加・更新時

[suzuki-shunsuke/pinact](https://github.com/suzuki-shunsuke/pinact) を使用して SHA 形式にピン留めする。

```yaml
- name: リポジトリのチェックアウト
  uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

#### Renovate による自動更新

`renovate.json` に `pinGitHubActionDigests` を追加し、アップデート時に自動的に SHA 形式へ変換されるようにする。

```json
{
  "extends": ["config:recommended", "helpers:pinGitHubActionDigests"]
}
```
