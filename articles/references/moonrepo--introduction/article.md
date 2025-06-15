---
title: '【moonrepo】モノレポツールの新星 "moon" repo とは？'
emoji: "🌕"
type: "tech"
topics: ["monorepo", "moonrepo", "tool"]
published: false
---

## はじめに

みなさんは「moonrepo（ムーンレポ）」というツールをご存じでしょうか？

moonrepo は Rust 製のモノレポ管理ツールで、[State of JavaScript 2024](https://2024.stateofjs.com/en-US/libraries/monorepo_tools/#monorepo_tools_ratios) の Monorepo Tools 部門にも名前が挙がっており、じわじわと注目度が高まっているツールです。

最近のモノレポ管理ツールとしては Turborepo や Nx が人気ですが、moonrepo はそれらと比べてよりシンプルな設計思想を持ち、かつ強力な自動化機能を備えているツールといった印象です。

私自身も１年前から個人開発で moonrepo を使っており、Turborepo や Nx で感じていた「使いにくさ」や「痒いところに手が届かない」という課題が解消され、非常に使い勝手の良さを実感しています。

本記事では、そんな moonrepo の魅力を紹介しつつ、どういった人におすすめなのか、目玉機能の紹介、ざっくりした使い方などを伝えられればと思います！

https://moonrepo.dev/docs/install

<br />

## こんな人におすすめ

moonrepo は以下のような方に特におすすめです！🚀

1. Javascript / Typescript に限定されないモノレポ環境を作りたい
2. タスクと依存関係を１つのファイルで定義したい
3. 変数や継承を用いてタスクや依存関係の定義をラクしたい
4. 豊富なオプションと高度な自動化を備えつつオプトインで利用したい

<br />

## moonrepo の全体像

では、さっそく[なぜおすすめなのか](#なぜおすすめなのか)について詳しく解説したいところですが、まずは全体像をイメージしていただくために、moonrepo を導入した際のプロジェクト構造とサンプルコードをご紹介したいと思います。

### プロジェクト構造の例

以下が moonrepo を導入した際の簡単なプロジェクト構造の例です。

```sh:
.
├── .moon/
│   ├── workspace.yml # プロジェクトの設定
│   ├── tasks.yml     # タスクの共通設定
│   └── toolchain.yml # ツールチェーンの設定
│
├── apps
│   ├── client # JS/TS のプロジェクトの場合
│   │   ├── …
│   │   ├── package.json
│   │   └── moon.yml
│   └── server # Rust のプロジェクトの場合
│   │   ├── …
│   │   ├── Cargo.toml
│   │   └── moon.yml
│   └── cms # PHP のプロジェクトの場合
│       ├── …
│       ├── composer.json
│       └── moon.yml
│
├── packages
│   ├── ml # Python のプロジェクトの場合
│   │   ├── …
│   │   ├── pyproject.toml
│   │   └── moon.yml
│   └── tools # 深いディレクトリ構造も可能
│       ├── logger # Ruby のプロジェクトの場合
│       │   ├── …
│       │   ├── *.gemspec
│       │   ├── Gemfile
│       │   └── moon.yml
│       └── monitoring # Go のプロジェクトの場合
│           ├── …
│           ├── go.mod
│           └── moon.yml
│
├── (各言語のパッケージ管理ファイル ※任意)
└── moon.yml
```

このように、各プロジェクトのディレクトリに `moon.yml` を配置する形になります。

今回の例はスタンダードなモノレポ構造を模倣してみましたが、moonrepo に明確なルールはなく `packages/tools` のように、ディレクトリ構造を１段階深くしてみたりなど、独自のディレクトリ構成も可能です。
‎

### サンプルコード

#### `.moon` ディレクトリ

このディレクトリ配下には moonrepo を活用する際のコアとなる設定ファイルを配置します。

:::details 設定の例

**`.moon/workspace.yml` ... プロジェクトの構成・設定を定義するファイル**

```yml:.moon/workspace.yml
# プロジェクトの構成
projects:
  sources:
    root: '.' # ルートディレクトリは root という名前で参照できるように登録
  globs:
    - '*/**/moon.yml' # moon.yml があるディレクトリをプロジェクトとして認識

# Git などの VCS の設定
vcs:
  defaultBranch: 'main'
```

> ドキュメント： https://moonrepo.dev/docs/config/workspace

\
**`.moon/tasks.yml` ... 共通タスクや全体的な設定を定義するファイル**

```yml:.moon/tasks.yml
# タスクのデフォルト設定
taskOptions:
  outputStyle: 'stream'
  runInCI: false

# 共通のタスク定義
tasks:
  check:
    command: "prettier --check ."
    options:
      runInCI: true
```

> ドキュメント： https://moonrepo.dev/docs/config/tasks

\
**`.moon/toolchain.yml` ... 使用するツールの設定を定義するファイル**

```yml:.moon/toolchain.yml
node:
  version: '16.13'
  rootPackageOnly: true # 依存関係はルートでしか管理しない
  packageManager: 'bun'
  bun:
    version: '1.0.0'

rust:
  version: '1.69.0'
  syncToolchainConfig: true # rust-toolchain.toml を自動で更新

python:
  version: '3.11'
  rootVenvOnly: true # 依存関係はルートでしか管理しない
  packageManager: 'uv'
  uv:
    version: '0.5.26'

typescript:
  syncProjectReferences: true # tsconfig.json の references を自動で更新
```

> ドキュメント： https://moonrepo.dev/docs/config/toolchain

:::

#### `**/*/moon.yml` ファイル

各プロジェクトのディレクトリに配置する設定ファイルです。

:::details 設定の例

```yml:apps/client/moon.yml
id: 'apps/client'

# 依存しているプロジェクトを定義
dependsOn:
  - { id: 'packages/ml' } # packages/ml を依存プロジェクトとして登録
  - { id: 'packages/tools/logger', scope: 'peer' } # ペア依存関係として登録
  - { id: 'packages/tools/monitoring', scope: 'development' } # 開発用の依存関係として登録

# タスクの定義
tasks:
  build:
    command: 'vite build'
    inputs: # 入力ファイルのパターンを指定
      - 'src/**/*'
      - 'tsconfig.json'
      - 'vite.config.ts'
    outputs: # 出力ファイルのパターンを指定
      - 'dist'
```

:::

#### スクリプト実行例

:::details 例

```sh
# 全ての環境で check タスクを実行
moon :check

# 特定の環境で check タスクを実行（`<project>:<task>` の形式）
moon apps/client:check

# ディレクトリを移動して run でタスクを実行する方法
cd apps/client
moon run check
```

:::

## なぜおすすめなのか

では、前のセクションで moonrepo の雰囲気がなんとなくつかめたと思いますので、いよいよおすすめポイントについて詳しく見ていきましょう。

### 1. Javascript / Typescript に限定されないモノレポ環境を作りたい

moonrepo のモノレポ管理は `package.json` など言語固有のエコシステムに依存するファイルではなく、YAML ファイルで定義するのみで実現されているため、基本的にあらゆるプログラミング言語に対応可能です。
加えて、この汎用的な基盤の上に特定の言語・ツールチェーンに対する強力なサポートを重ねることで、各言語のエコシステムや特性を最大限に活用できる仕組みを備えています。

この特定言語・ツールチェーンへのサポート状況は Tier で分類されています。

- **Tier 0**： タスクランナーと依存関係の定義をサポート
- **Tier 1**： 言語を設定として指定でき、専用の機能を提供
- **Tier 2**： 言語の構成ファイルなどを解析し、依存関係やタスクを自動推測
- **Tier 3**： 依存関係の自動インストールやバージョン管理など、高度なツールチェーン統合

言語の対応状況や各 Tier の詳細な機能については、公式ドキュメントをご確認ください。

> https://moonrepo.dev/docs#supported-languages

2025年3月時点では、JS/TS、Rust、Python、Go、Ruby、PHP、Bash が Tier 1 までのサポートが提供されており、非常に多くの言語で専用の機能を使用することができます。
‎

### 2. タスクと依存関係を１つのファイルで定義したい

moonrepo はタスクと依存関係を１つのファイルで定義することができます。

以下は簡単な例です。

```yml:moon.yml
tasks:
  build:
    command: 'echo "Hello build"'

  preview:
    command: 'echo "Hello preview"'
    deps: ["build"]
```

`command` には実行する任意のコマンドを、`deps` には依存するタスク名を記述しています。\
とってもスマートですね...！✨

\
では、実際に `preview` タスクを実行してみるとしましょう。

```sh
moon run preview
```

上記を実行すると、以下のように出力されます。

```sh
Hello build   # deps に指定された build タスクが先に実行される
Hello preview # その後 preview タスクが実行される
```

`preview` よ呼び出しただけですが、しっかりと `build` コマンドの後に `preview` コマンドが実行されていることがわかると思います。

\
少し Turborepo と比較してみましょう。
これと同じことを Turborepo で定義すると、以下のようになります。

:::details Turborepo の場合

**`command` に相当する箇所は `package.json` に記述する**

```json:package.json
{
  "scripts": {
    "build": "echo 'Hello build'",
    "preview": "echo 'Hello preview'"
  }
}
```

\
**`deps` に相当する箇所は `turbo.json` に記述する**

```json:turbo.json
{
  "tasks": {
    "preview": {
      "dependsOn": ["build"]
    }
  }
}
```

:::

\
比較してみると moonrepo がとてもシンプルに見えるのではないでしょうか？
もちろん、感じ方や考え方は色々ありますので一概にどちらの書き方が優れているとは言えませんが、関連する設定が１つのファイルにまとまっているほうが直感的で管理しやすいと感じる方には、moonrepo はかなり魅力的に映ると思います！
‎

### 3. タスク・依存関係の定義を変数的に使い回してラクしたい人

moonrepo では、タスクの 入力（inputs）や出力（outputs）の対象を適切に指定することで、変更されたファイルのみを検知し増分キャッシュが適用され、ビルドやタスクの実行が最適化されます。しかし、同じファイルパターンを何度も記述するのは手間がかかりますよね。そこで便利なのが fileGroups 機能です。

```yml:moon.yml
fileGroups:
  sources:
    - 'src/**/*'
    - 'types/**/*'

tasks:
  build:
    command: 'vite build'
    inputs:
      - '@group(sources)'

  test:
    command: 'vite test'
    inputs:
      - '@group(sources)'
```

このように fileGroups を使用することで、共通のファイルパターンを一元管理できます。さらに、この fileGroups は `.moon/tasks.yml` に定義することで、各プロジェクトの `moon.yml` でも `@group(sources)` のように参照でき、プロジェクト間で共有することができます。

:::details: もっとプログラムチックに書きたい！

最近、moonrepo は [Pkl configuration](https://moonrepo.dev/docs/guides/pkl-config) をサポートするようになりました。

[Pkl](https://pkl-lang.org) は Apple が開発したプログラム可能な設定形式で、この形式を使用すると、プログラミング的な完全な変数や if 文、ループなどの制御構造まで実装できるようになります。

詳しい使い方やコードサンプルについては、公式ドキュメントをご参照ください。

https://moonrepo.dev/docs/guides/pkl-config

:::

### 5. 多機能だけど必要最低限の機能だけ選択して使いたい人

えー、この moonrepo さん、ものレポ管理以外にもたくさんの機能を持っています。
一度、大きなカテゴリーでまとめてみましょう。

| 機能                 | 説明                                                             | 類似ツール                   |
| -------------------- | ---------------------------------------------------------------- | ---------------------------- |
| タスクランナー       | タスクと依存関係を一元管理                                       | go-task, justfile, makefile  |
| パッケージ管理 ※1    | 依存パッケージの自動インストールと同期                           | mise, nvm, nodebrew          |
| Git(VCS) Hooks       | コミットやプッシュなどのアクションを検知し、自動でタスク実行する | lefthook, husky, lint-staged |
| テンプレート生成機能 | テンプレートを生成する機能を提供                                 | Twig, Blade                  |

※1 ... proto をインストールする必要あり

ここまでの紹介だけでも、かなり様々な機能をカバーしていることがわかると思います。（紹介しきれていない機能もたくさんあります）

ですが、多機能なツールはその分依存することが多く、不要なものを使うのを渋ってしまう気持ちもわかります。

ですが、moonrepo は 基本的にオプトイン のスタイルで、使いたい機能だけ ピンポイントで有効化できるのが特徴です。
何も設定がないと、タスクランナーと依存関係のみを管理するシンプルなツールになります。

そう、moonrepo を使うと。
しかも、moonrepo のモノレポ管理以外の機能は基本的にオプトインとなっているため、たとえば、まだ信頼度が高い husky を使いたい、といった場合も特に設定せず husky を導入することができます。

## moonrepo のその他の魅力

上で挙げた以外にも、moonrepo には「便利だな」と思うポイントがいろいろあります。ここでは大きく 「自動化」と「CI の簡略化」 に分けてご紹介します。

### 豊富なオプションと高度な自動化を備えつつオプトインで利用したい

moonrepo は非常に多くのオプションと高度な自動化を提供していますが、それらは基本オプトインで利用できる設計になっています。

#### オプトイン設計のメリット

moonrepo のオプトイン設計には以下のようなメリットがあります：

- **段階的な導入が可能**：最初はシンプルな設定から始めて、必要に応じて機能を追加できる
- **学習コストの低減**：必要な機能だけを学べば良いため、初期の学習負荷が少ない
- **既存ツールとの共存**：既存のツールチェーンを壊すことなく、部分的に導入できる
- **チーム間の合意形成が容易**：機能を一つずつ導入することで、チーム内での合意を取りやすい

#### 主要なオプションの具体例

moonrepo は大きく分けて、以下の設定が可能です：

- **workspace**: モノレポの設定をまとめておくファイル
- **toolchain**: ツールチェーンの設定をまとめておくファイル
- **task**: タスクの設定をまとめておくファイル

特にタスクのオプションが豊富で、2025/03/19 現在だけでも以下のような多様な設定が可能です：

```yaml
tasks:
  build:
    command: "vite build"
    # 以下のオプションは全てオプトイン
    options:
      cache: true # キャッシュを有効化
      persistent: false # タスクを継続的に実行しない
      runDepsInParallel: true # 依存タスクを並列実行
      runInCI: true # CI環境でも実行
      retryCount: 3 # 失敗時に再試行する回数
      env: # 環境変数の設定
        NODE_ENV: "production"
```

これらのオプションは、必要に応じて段階的に導入できます。例えば、最初はシンプルなタスク定義から始めて、徐々にキャッシュや依存関係の最適化を追加していくといった使い方が可能です。

#### 自動化機能の具体例

moonrepo は高度な自動化機能も備えています。例えば：

- **TypeScript Project References の自動設定例**：

  ```yaml
  # .moon/toolchain.yml
  typescript:
    syncProjectReferences: true # この設定だけで自動化される
  ```

- **package.json の依存関係の同期例**：

  ```yaml
  # moon.yml
  dependsOn:
    - { id: "packages/shared", scope: "production" }
  # 上記の設定が自動的に package.json に反映される
  ```

- **依存パッケージの自動インストール**：タスク実行時に必要なパッケージを自動でインストール
- **コードオーナーの自動同期**：CODEOWNERS ファイルを自動生成・更新

#### 段階的な導入シナリオ

moonrepo は段階的に導入できるのが特徴です。以下は導入シナリオの例です：

**ステップ1: 基本的なタスクランナーとして**

```yaml
tasks:
  build: "vite build"
  test: "jest"
```

**ステップ2: 依存関係の管理を追加**

```yaml
dependsOn:
  - { id: "packages/shared" }
tasks:
  build:
    command: "vite build"
    deps: ["^build"] # 依存プロジェクトのbuildタスクを先に実行
```

**ステップ3: キャッシュと最適化を追加**

```yaml
tasks:
  build:
    command: "vite build"
    options:
      cache: true
    inputs:
      - "src/**/*"
    outputs:
      - "dist"
```

このように、豊富なオプションと高度な自動化機能を必要に応じて選択的に活用できる点が、moonrepo の大きな魅力の一つとなっています。

### CI がむちゃくちゃ簡略化できる

moonrepo は タスク依存関係の管理 や 差分ビルド の仕組みがしっかりしているので、CI/CD パイプラインの設定がスッキリ します。
具体的には、「このタスクが終わってからあのタスクを実行する」 といったフローが簡単に書けますし、差分検出（変更のあったプロジェクトだけビルド or テスト）を自動でやってくれるため、ビルド時間の短縮や無駄なリソース消費の削減に効果的です。
私の場合、GitHub Actions 上でも「moon コマンド一発」でビルド〜テスト〜デプロイまでを並列・直列にサクッと書けるので、YAML がかなりコンパクトになりました。

## 現状の課題と解決策

- **クラウドキャッシュが未実装**
  現時点ではローカルキャッシュのみのため、大規模プロジェクトの効率化には工夫が必要です。
  今後のバージョンアップで対応が予定されているようなので、Changelog をチェックしましょう。

- **ドキュメントが少ない**
  まだ新しいツールなので情報が少ないのが実情です。
  公式ドキュメントや GitHub の Issue、コミュニティなどを積極的に活用すると良いでしょう。

## まとめ

日本語の情報が少ないぶん、最初は戸惑う点もあるかもしれませんが、オプトインで徐々に導入できる のが moonrepo の良いところ。
「とりあえずタスクランナーとして試してみる」→「CI を連携してみる」→「ツールチェーン管理やコードオーナー同期を活用」……とステップを踏めば、きっと快適なモノレポライフが待っています！

もしこの記事を読んで「ちょっと面白そうかも」と思ったら、ぜひ一度 moonrepo を試してみてください。
「mono」の typo じゃありません。「moon」 です！ それだけは覚えて帰ってもらえたら嬉しいです（笑）

それでは、快適なモノレポライフを楽しんでくださいね。ありがとうございました！
