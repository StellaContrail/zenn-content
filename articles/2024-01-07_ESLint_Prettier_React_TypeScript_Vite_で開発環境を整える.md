---
title: "ESLint + Prettier + React + TypeScript + Vite で開発環境を整える"
emoji: "🔷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript","React","ESLint","prettier","vite"]
published: true
---

# 目的
Vite を使うことで、ESLint + Prettier を有効にした TypeScript + React プロジェクトを迅速に作成することができる。今回はプロジェクトの作成から、ESLint + Prettier を VSCode 上で使えるようになるまでの手順を説明する。

# 前提条件
以下のバージョンを想定して、環境構築の手順を説明する。
| パッケージ名 | バージョン |
| ---- | ---- |
| npm | 10.2.3 |
| node | 21.2.0 |
| vite | 5.0.8 |

# 環境構築の手順
Vite はビルドツールの一種であり、プロジェクト作成の時点で幾つかのパッケージが同梱されている。ESLint も同梱されているパッケージ群に含まれるが、Prettier と組み合わせる場合は多少工夫が必要である。


## Vite プロジェクトの作成
まずは以下のコマンドを入力し、画面の指示に従ってプロジェクトの設定を行う。
```bash:bash
npm create vite@latest
```

"Project name"（プロジェクト名）は自由に指定し、"Framework" には React を、"Variant" には TypeScript を指定する。

| 質問項目 | 回答内容 |
| ---- | ---- |
| Project name | 自由記述 |
| Select a framework | React |
| Select a variant | TypeScript |

次に、プロジェクトのディレクトリに移動し、依存パッケージをインストールする。
```bash:bash
cd vite-project
npm install
```

プロジェクトの作成が完了した。

## ESLint の導入
### ESLint のインストール
Vite は ESLint をリンターに採用しており、プロジェクトを作成した時点で ESLint による静的解析が有効になっている。プラグインは `@typescript-eslint/eslint-plugin` が既存で設定されており、`recommended` や `strict` などが選択肢として存在する。
> Ref. [Configurations | typescript-eslint](https://typescript-eslint.io/linting/configs#recommended-configurations)

:::message alert
警告
eslint-config-airbnb-typescript はルールが厳しすぎるために、Vite で作成されたプロジェクトではエラーが多発してしまう。このため、以降は @typescript-eslint/eslint-plugin の使用を前提において説明する。
:::

プロジェクト作成時において、React 関連のルールは [react-hooks/recommended](https://www.npmjs.com/package/eslint-plugin-react-hooks) のみが有効になっている。より具体的なルールを追加したい場合は、以下のように [jsx-eslint/eslint-plugin-react](https://github.com/jsx-eslint/eslint-plugin-react) を追加する。

依存パッケージ `eslint-plugin-react` のインストール
```bash:bash
npm install eslint eslint-plugin-react --save-dev
```

ESLint の extends 設定に `react/recommended` と `react/jsx-runtime` を追加する。eslint-plugin-react に React の対象バージョンを教えてあげるため、`settings.react.version: "detect"` も追加する。

```javascript:.eslintrc.cjs
module.exports = {
  // ...
  // 省略
  // ...
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react/recommended", // <--- 追記
    "plugin:react/jsx-runtime", // <--- 追記
    "plugin:react-hooks/recommended",
  ],
  ignorePatterns: ["dist", ".eslintrc.cjs", "vite.config.js"], // <--- 追記
  // ...
  // 追記
  settings: {
    react: {
      version: "detect",
    },
  },
  // ...
};
```

これで React 関連の推奨ルールが有効化された。eslint-plugin-react で指定できるルールの詳細は、公式リポジトリの "List of supported rules" を参照する。

> Ref. [jsx-eslint/eslint-plugin-react: React-specific linting rules for ESLint](https://github.com/jsx-eslint/eslint-plugin-react#list-of-supported-rules)

### ESLint による命名規則チェックの有効化
標準では命名規則に関する静的解析が設定されていないため、ESLint の設定ファイル `.eslintrc.cjs` に `@typescript-eslint/naming-convention` から始まるルールを追加する。

命名規則には [TypeScript Deep Dive](https://typescript-jp.gitbook.io/deep-dive/styleguide) のものをベースとし、React に適応する形式に微修正したものを使用する。

```javascript: .eslintrc.cjs
module.exports = {
  // ...
  // 省略
  // ...
  rules: {
    "react-refresh/only-export-components": [
      "warn",
      { allowConstantExport: true },
    ],
    "@typescript-eslint/naming-convention": [
      "error",
      {
        selector: "variable",
        format: ["camelCase"],
      },
      {
        selector: "function",
        format: ["camelCase", "PascalCase"],
      },
      {
        selector: "parameter",
        format: ["camelCase"],
      },
      {
        selector: "class",
        format: ["PascalCase"],
      },
      {
        selector: "method",
        format: ["camelCase"],
      },
      {
        selector: "property",
        format: ["camelCase"],
      },
      {
        selector: "interface",
        format: ["PascalCase"],
      },
      {
        selector: "typeAlias",
        format: ["PascalCase"],
      },
      {
        selector: "typeParameter",
        format: ["camelCase"],
      },
      {
        selector: "enum",
        format: ["PascalCase"],
      },
      {
        selector: "enumMember",
        format: ["UPPER_CASE"],
      },
    ],
  },
};
```

Prettier の設定後に再び ESLint の設定ファイルを修正することになるが、一旦は ESLint の導入が完了した。

### ESLint の動作確認
動作確認のために、`src/App.tsx` を以下のように書き換える。

```react:src/App.tsx
function App() {
  let Name: string = "Alice";

  return (
    <>
      <h1>Hello, World!</h1>
    </>
  );
}

export default App;

```

ファイルを保存したうえで、ルートディレクトリで以下のコマンドを実行する。もしくは、`npm run lint` を実行することでも同じコマンドが実行できる。
```bash:bash
eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0
```

出力例
```bash:bash
> vite-project@0.0.0 lint
> eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0


/home/dev/vite-project/src/App.tsx
  2:7  error  Variable name `Name` must match one of the following formats: camelCase  @typescript-eslint/naming-convention
  2:7  error  'Name' is assigned a value but never used                                @typescript-eslint/no-unused-vars
  2:7  error  'Name' is never reassigned. Use 'const' instead                          prefer-const

✖ 3 problems (3 errors, 0 warnings)
  1 error and 0 warnings potentially fixable with the `--fix` option.
```

出力例のように、`@typescript-eslint` や `prefer-const` などの ESLint に起因するエラーが発生していれば、ESLint が正常に動作していることの確認は完了である。

## Prettierの導入
### Prettier のインストール
まずは Prettier のパッケージをインストールする。
```bash:bash
npm install --save-dev --save-exact prettier
```

次に Prettier の設定ファイルを作成する。
```bash:bash
node --eval "fs.writeFileSync('.prettierrc','{}\n')"
```

`.prettierignore` ファイルをプロジェクトのルートディレクトリに作成し、フォーマット整形の対象外となるファイルを指定する。
```python:.prettierignore
# Logs
logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
lerna-debug.log*

# Artifacts
dist
dist-ssr
*.local

# Editor directories and files
.vscode/*
!.vscode/extensions.json
.idea
.DS_Store
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?

# JSON (Optional):
*.json

# Docs (Optional):
*.md
```

### ESLint と Prettier における競合の解決
ESLint と Prettier を併用するのであれば、ESLint のフォーマット機能と Prettier のフォーマット機能が競合しないように設定してあげる必要がある。

ESLint のフォーマット機能を無効にするプラグインが Prettier から提供されているため、そのプラグイン [prettier/eslint-config-prettier](https://github.com/prettier/eslint-config-prettier) を導入することで競合を回避する。

まずは prettier/eslint-config-prettier をインストールする。
```bash:bash
npm install --save-dev eslint-config-prettier
```

次に、ESLint の設定ファイル `.eslintrc.cjs` に `prettier` の使用を「**最後に**」追記する。各extendsが適用された後に上書きをしてあげたいので、必ず順番として最後に記載する。
```javascript:.eslintrc.cjs
module.exports = {
  // ...
  // 省略
  // ...
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react-hooks/recommended",
    "prettier", // <--- 追記
  ],
  // ...
  },
};
```

:::message
情報
ESLint と Prettier のバージョンによっては、eslint-config-prettier による競合の解決が間に合っていない可能性がある。競合が起きているのかを確認するには次のコマンドを実行する。

```bash:bash
npx eslint-config-prettier src/**/*.tsx
```

競合の解決方法などは eslint-config-prettier の GitHub リポジトリを参照する。

> Ref. [prettier/eslint-config-prettier: Turns off all rules that are unnecessary or might conflict with Prettier.](https://github.com/prettier/eslint-config-prettier?tab=readme-ov-file#cli-helper-tool)
:::

以上で、Prettier の導入が完了した。

### Prettier の動作確認
動作確認のために、`src/App.tsx` を以下のように書き換える。

```react:src/App.tsx
function App() {
    return (
    <>
      <h1>Hello, World!
      </h1>
    </>



    
  )
}

                                 export default App
```

ファイルを保存したうえで、ルートディレクトリで以下のコマンドを実行する。
```bash:bash
npx prettier . --write
```

出力例
```bash:bash
.eslintrc.cjs 28ms (unchanged)
.prettierrc 8ms (unchanged)
index.html 15ms (unchanged)
src/App.css 16ms (unchanged)
src/App.tsx 84ms
src/index.css 4ms (unchanged)
src/main.tsx 4ms (unchanged)
src/vite-env.d.ts 1ms (unchanged)
vite.config.ts 2ms (unchanged)
```
```react:src/App.tsx
function App() {
  return (
    <>
      <h1>Hello, World!</h1>
    </>
  );
}

export default App;

```

出力例のように `src/App.tsx` が整形されていたら、Prettier が正常に動作していることの確認は完了した。

### Prettier の詳細設定
Prettier の既定フォーマットはバージョン毎に変化しているため、ルールを固定したい場合や、カスタマイズしたい場合は `.prettierrc` でルール設定を行う。

```json:.prettierrc
{
  "trailingComma": "es5",
  "tabWidth": 2,
  "semi": true,
  "singleQuote": false
}
```

**trailingComma**：配列や JSON などにおいて、末尾のコンマを付与する規則。
**tabWidth**：インデントに使うスペースの数。
**semi**：セミコロンを付与するか否か。
**singleQuote**：ダブルクォーテーションの代わりに、シングルクォーテーションを使うか否か。

より包括的なオプションは [Prettier の Options ページ](https://prettier.io/docs/en/options) を参照する。

## オプション：VSCode拡張機能の整備
VSCode 上で ESLint の静的解析結果や、保存時の Prettier によるフォーマットを有効にするためには、拡張機能をインストールする必要がある。

まず、VSCode の拡張機能をインストールする。ESLint に対しては [dbaeumer.vscode-eslint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)、Prettier に対しては [esbenp.prettier-vscode](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode) をインストールする。

ESLint による静的解析は、インストールした時点で有効化されるため、特に設定は必要ない。

Prettier によるフォーマットは、以下の設定を追加することで有効化される。
```json:.vscode/settings.json
{
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.formatOnSave": true
}
```

以降、ファイルの保存時にフォーマットを自動的に整形することが出来る。

デフォルトでは、フォーマット整形はファイル単位で行われるため、git diff に大量の変化を出したくない場合は以下の設定も追加する。

```json:.vscode/settings.json (抜粋)
"editor.formatOnSaveMode": "modificationsIfAvailable"
```

## 参考文献
- ESLint で使えるルール集

https://eslint.org/docs/latest/rules/

- `@typescript-eslint/eslint-plugin` で追加できるルール集

https://typescript-eslint.io/rules/

- Prettier で使えるオプション集

https://prettier.io/docs/en/options

- `prettier/eslint-config-prettier` が既定で無効化にしているが、特定の条件下では有効に出来るルール集

https://github.com/prettier/eslint-config-prettier#special-rules

- tsconfig の書き方や詳細なトラブルシューティングが参考になる

https://chaika.hatenablog.com/entry/2021/07/10/083000

