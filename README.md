# School Converter

School Converterは、学校のデータを管理および変換するためのウェブアプリケーションです。Nuxtで構築されています。

## 機能

-   **学校データ管理**: 学校のリストとそれに関連するデータを管理します。
-   **データ変換**: 定義されたしきい値に基づいて学校のデータを変換します。
-   **拡張可能**: YAML設定ファイルを介して新しい学校の種類や学校を簡単に追加できます。

## 使い方

1.  リポジトリをクローンします。
2.  依存関係をインストールします: `npm install`
3.  開発サーバーを起動します: `npm run dev`
4.  ブラウザで `http://localhost:3000` を開いてアプリケーションを表示します。

## データソース

学校のデータは `app/data/schools.yaml` ファイルで管理されます。このファイルには、さまざまな学校の種類、そのしきい値、および各学校の略称とコホートサイズのリストが含まれています。

## セットアップ

依存関係を必ずインストールしてください:

```bash
# npm
npm install

# pnpm
pnpm install

# yarn
yarn install

# bun
bun install
```

## 開発サーバー

`http://localhost:3000` で開発サーバーを起動します:

```bash
# npm
npm run dev

# pnpm
pnpm dev

# yarn
yarn dev

# bun
bun run dev```

## 本番環境

本番用にアプリケーションをビルドします:

```bash
# npm
npm run build

# pnpm
pnpm build

# yarn
yarn build

# bun
bun run build
```

ローカルで本番ビルドをプレビューします:

```bash
# npm
npm run preview

# pnpm
pnpm preview

# yarn
yarn preview

# bun
bun run preview
```

詳細については、[デプロイに関するドキュメンテーション](https://nuxt.com/docs/getting-started/deployment) を確認してください。

## 貢献

貢献を歓迎します！プルリクエストを送信するか、イシューをオープンにしてください。

## ライセンス

このプロジェクトはMITライセンスの下でライセンスされています。