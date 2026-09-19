---
title: "ブラウザ拡張機能を 3 ストアに公開する CLI wepub (Chrome Web Store API v2 対応)"
emoji: "📦"
type: "tech"
topics: ["chromeextension", "firefox", "githubactions", "rust", "cli"]
published: true
---

ブラウザ拡張機能を公開するための Rust 製 CLI である [wepub](https://github.com/iorate/wepub) を紹介します。自作の拡張機能 [uBlacklist](https://github.com/iorate/ublacklist) の公開のためだけに細々と開発していましたが、なぜか他にも利用者がいるようなのと、[Chrome Web Store API (V1) が 2026 年 10 月 15 日に EOL になり](https://developer.chrome.com/docs/webstore/api/v1)、移行先を探している人がいるかもしれないので、宣伝記事を書くことにしました。

:::message
以降の本文は Claude Fable 5.1 が書きました。一字一句、人間がチェックしています。
:::

## はじめに

[Chrome Web Store API v1](https://developer.chrome.com/docs/webstore/api/v1) は 2026 年 10 月 15 日に終了し、以降は v2 への移行が必要になります。v2 では公開先のアイテムを Publisher ID と Item ID の組で指定するなど、v1 とは API の形が変わっています。

この記事では、その Chrome Web Store API v2 に対応した拡張機能公開用 CLI である [wepub](https://github.com/iorate/wepub) を紹介します。筆者が開発している拡張機能 [uBlacklist](https://github.com/iorate/ublacklist) のリリースワークフローで実際に使っているものです。設計方針、環境変数による設定、GitHub Actions からの使い方の順に説明します。

## wepub とは

wepub は、ブラウザ拡張機能のパッケージ (zip) を各ストアにアップロードして公開する CLI です。Rust で書かれた単一バイナリで、実行時に Node.js などのランタイムを必要としません。

対応しているストアと API のバージョンは次のとおりです。

- Chrome Web Store (API v2)
- Firefox Add-ons (API v5)
- Edge Add-ons (API v1.1)

3 ストアそれぞれにサブコマンドがあり、パッケージのパスを位置引数で渡します。

```sh
wepub chrome ./my-extension.zip
wepub firefox ./my-addon.zip --channel listed
wepub edge ./my-addon.zip
```

なお、wepub が扱えるのは既存アイテムの更新だけです。新しい拡張機能の初回登録は、各ストアの Web UI から行う必要があります。これは各ストアの API 自体の制約でもあります。

## 設計方針

### 1 ストア = 1 サブコマンド

ストアごとに必要な ID や認証情報は大きく異なります。wepub では、それらをひとつのコマンドに詰め込むのではなく、ストアごとのサブコマンドに分けています。各サブコマンドのフラグはそのストアに必要なものだけで構成されており、ヘルプを読めば何を用意すればよいかが分かるようになっています。

ID と認証情報以外のフラグは次のとおりです。Chrome Web Store 向けには、承認後すぐに公開するか公開を保留するかの選択、段階的ロールアウトの初期割合、レビューのスキップを試みるかどうかを指定できます。

```
--publish-type       default | staged
--deploy-percentage  0-100
--skip-review        true | false
```

Firefox Add-ons 向けには、公開チャンネル (listed / unlisted) の指定が必須で、そのほかに対応アプリケーション、審査担当者向けのメモ、リリースノートとその言語、ソースアーカイブの添付を指定できます。

```
--channel             listed | unlisted
--compatibility       firefox,android
--approval-notes      TEXT
--approval-notes-file PATH
--release-notes       TEXT
--release-notes-file  PATH
--release-notes-lang  LANG
--source              PATH
```

Edge Add-ons 向けには、認定審査のためのメモを指定できます。

```
--notes       TEXT
--notes-file  PATH
```

メモ類はコマンドラインで直接渡すほか、ファイルからも読み込めます。ファイル名にハイフン 1 文字を指定すると標準入力から読みます。CI で生成したリリースノートをそのまま流し込む用途を想定しています。

### アップロードから公開までを 1 コマンドで完結させる

各ストアの API は、パッケージをアップロードした後にサーバー側で処理が完了するのを待ち、それから公開や提出の操作を行う、という複数ステップの構成になっています。wepub はこの一連の流れをひとつのコマンドで実行します。処理待ちが必要なストアではステータスをポーリングし、一定時間内に完了しなければタイムアウトとして失敗します。

途中のステップで失敗すれば非ゼロの終了コードで終了し、エラーの内容を標準エラー出力に書き出します。既定ではどのステップを実行中かが INFO レベルで表示され、フラグで詳細表示や抑制に切り替えられます。

```
wepub -v chrome ./my-extension.zip
wepub -q chrome ./my-extension.zip
```

### 認証情報の扱い

認証情報はフラグでも環境変数でも渡せますが、CI ではシークレットを環境変数として渡すのが自然です。次の節で説明する `WEPUB_*` 環境変数がそのためのものです。

シークレットの漏洩を防ぐため、ヘルプ出力に環境変数の値を表示しないようにしているほか、認証ヘッダやトークン交換のリクエスト・レスポンス本文はログに出さない方針で実装しています。詳細ログを有効にしたまま CI で実行しても、シークレットがログに残らないようにするためです。

### 単一バイナリとして配布する

wepub は GitHub Releases でプラットフォームごとのビルド済みバイナリを配布しています。後述する GitHub Actions 用の setup アクションのほか、シェルスクリプト、PowerShell、Homebrew、npm、Cargo からインストールできます。

```sh
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/iorate/wepub/releases/latest/download/wepub-installer.sh | sh
```

```sh
brew install iorate/tap/wepub
```

```sh
npm install -g @iorate/wepub
```

```sh
cargo install wepub
```

npm パッケージも用意しているので、Node.js ベースのプロジェクトからも同じ導線で導入できます。

### ライブラリとしても使える

CLI の中核は `wepub-core` というクレートに分離されており、crates.io で公開しています。ストアごとに `publish` というビルダー形式のエントリポイントがあり、非同期ランタイムに依存しないので、Rust で独自のリリースツールを組む場合にも利用できます。

## `WEPUB_*` 環境変数

ID と認証情報は、フラグと環境変数のどちらでも指定できます。フラグと環境変数の対応は次のとおりです。

Chrome Web Store:

```
--publisher-id   WEPUB_CHROME_PUBLISHER_ID
--item-id        WEPUB_CHROME_ITEM_ID
--client-id      WEPUB_CHROME_CLIENT_ID
--client-secret  WEPUB_CHROME_CLIENT_SECRET
--refresh-token  WEPUB_CHROME_REFRESH_TOKEN
--access-token   WEPUB_CHROME_ACCESS_TOKEN
```

Firefox Add-ons:

```
--addon-id    WEPUB_FIREFOX_ADDON_ID
--api-key     WEPUB_FIREFOX_API_KEY
--api-secret  WEPUB_FIREFOX_API_SECRET
```

Edge Add-ons:

```
--product-id  WEPUB_EDGE_PRODUCT_ID
--client-id   WEPUB_EDGE_CLIENT_ID
--api-key     WEPUB_EDGE_API_KEY
```

Chrome Web Store の認証には 2 つのモードがあります。ひとつは OAuth のクライアント ID、クライアントシークレット、リフレッシュトークンの 3 つを渡す方法で、公式ドキュメントの [Use the Chrome Web Store API](https://developer.chrome.com/docs/webstore/using-api) の手順で取得できます。もうひとつは、[サービスアカウント](https://developer.chrome.com/docs/webstore/service-accounts) などで事前に取得したアクセストークンをそのまま渡す方法です。2 つのモードは排他で、同時に指定するとエラーになります。

Firefox Add-ons の API キーと API シークレットは [API Credentials Management Page](https://addons.mozilla.org/developers/addon/api/key/) から取得します。Edge Add-ons のクライアント ID と API キーは [Partner Center](https://partner.microsoft.com/dashboard/microsoftedge/public/login) の Publish API ページから取得し、Product ID は同じく Partner Center の Extension overview ページに表示される GUID です。

wepub は起動時にカレントディレクトリの `.env` ファイルを読み込むので、ローカルで試すときは環境変数をそこに置いておけます。シェルの環境変数がすでに設定されている場合はそちらが優先されます。

```
WEPUB_CHROME_PUBLISHER_ID=...
WEPUB_CHROME_ITEM_ID=...
WEPUB_CHROME_CLIENT_ID=...
WEPUB_CHROME_CLIENT_SECRET=...
WEPUB_CHROME_REFRESH_TOKEN=...
```

なお、プロキシ経由で通信する場合は次の環境変数を参照します。OS のプロキシ設定は読みません。

```
http_proxy
https_proxy
no_proxy
```

## GitHub Actions からの使い方

### setup アクション

wepub のリポジトリには、ランナーに wepub をインストールする composite アクションが含まれています。

```yaml
- name: Install wepub
  uses: iorate/wepub/setup@v1
```

このアクションは、ランナーの OS とアーキテクチャに合ったビルド済みバイナリを GitHub Releases からダウンロードし、GitHub の artifact attestation でビルドの出所を検証してから、ツールキャッシュに展開してパスを通します。検証には既定で GitHub Actions の自動トークンを使うので、追加の設定は不要です。

uBlacklist ではサプライチェーン対策として、タグではなくコミット SHA でアクションを固定しています。

```yaml
- name: Install wepub
  uses: iorate/wepub/setup@cc561a13c1b2a0eb32e0e56aadeef79a289d4acc # v1.0.5
```

### ワークフローの構成

uBlacklist の [publish.yml](https://github.com/iorate/ublacklist/blob/0ca455399f97d5d2c978b993dd00b36c23d070e8/.github/workflows/publish.yml) は、次のような構成になっています。

1. build ジョブでストアごとのパッケージとメモ類を生成し、artifact としてアップロードする
2. ストアごとのジョブが artifact をダウンロードし、wepub で公開する
3. 3 ストアすべての公開が終わったら GitHub Release を作る

ストアごとのジョブには GitHub の environment を割り当て、シークレットをそのジョブだけに渡しています。ID のような秘匿不要な値は variables に、認証情報は secrets に置き、いずれも `WEPUB_*` 環境変数として wepub に渡します。

以下は、この構成の公開部分を抜き出したものです。

```yaml
jobs:
  chrome:
    needs: build
    environment: chrome
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v8
        with:
          name: release
          path: dist/release

      - name: Install wepub
        uses: iorate/wepub/setup@v1

      - name: Publish to Chrome Web Store
        run: wepub chrome dist/release/chrome/package.zip
        env:
          WEPUB_CHROME_PUBLISHER_ID: ${{ vars.CHROME_PUBLISHER_ID }}
          WEPUB_CHROME_ITEM_ID: ${{ vars.CHROME_ITEM_ID }}
          WEPUB_CHROME_CLIENT_ID: ${{ secrets.CHROME_CLIENT_ID }}
          WEPUB_CHROME_CLIENT_SECRET: ${{ secrets.CHROME_CLIENT_SECRET }}
          WEPUB_CHROME_REFRESH_TOKEN: ${{ secrets.CHROME_REFRESH_TOKEN }}

  edge:
    needs: build
    environment: edge
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v8
        with:
          name: release
          path: dist/release

      - name: Install wepub
        uses: iorate/wepub/setup@v1

      - name: Publish to Edge Add-ons
        run: wepub edge dist/release/edge/package.zip --notes-file dist/release/edge/notes-for-certification.txt
        env:
          WEPUB_EDGE_PRODUCT_ID: ${{ vars.EDGE_PRODUCT_ID }}
          WEPUB_EDGE_CLIENT_ID: ${{ secrets.EDGE_CLIENT_ID }}
          WEPUB_EDGE_API_KEY: ${{ secrets.EDGE_API_KEY }}

  firefox:
    needs: build
    environment: firefox
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v8
        with:
          name: release
          path: dist/release

      - name: Install wepub
        uses: iorate/wepub/setup@v1

      - name: Publish to Firefox Add-ons
        run: |
          wepub firefox dist/release/firefox/package.zip \
            --channel listed \
            --compatibility firefox,android \
            --approval-notes-file dist/release/firefox/approval-notes.txt \
            --release-notes-file dist/release/firefox/release-notes.md \
            --source dist/release/firefox/source.zip
        env:
          WEPUB_FIREFOX_ADDON_ID: ${{ vars.FIREFOX_ADDON_ID }}
          WEPUB_FIREFOX_API_KEY: ${{ secrets.FIREFOX_API_KEY }}
          WEPUB_FIREFOX_API_SECRET: ${{ secrets.FIREFOX_API_SECRET }}
```

Firefox Add-ons にはソースアーカイブを添付しています。uBlacklist ではビルドに必要な環境変数を `.env` として同梱したものを build ジョブとは別に生成し、審査担当者が同じ手順でビルドを再現できるようにしています。Edge Add-ons 向けのメモは、リリースページの URL をテンプレートに埋め込んで生成したファイルを渡しています。

### ほかのプロジェクトでの採用例

wepub は uBlacklist 以外のプロジェクトでも使われています。

[hanydd/BilibiliSponsorBlock](https://github.com/hanydd/BilibiliSponsorBlock) の [publish-stores.yml](https://github.com/hanydd/BilibiliSponsorBlock/blob/9f316ee195afbd3f4c36b08c7fbf0c57f78874de/.github/workflows/publish-stores.yml) は、手動実行時にストアごとの公開可否を boolean 入力で選べるようにしたワークフローです。公開ステップの前に必要な `WEPUB_*` 環境変数がすべて設定されているかを確認するステップを置き、Edge 向けの認定メモにはコミットメッセージを流し込んでいます。

[pt-plugins/PT-depiler](https://github.com/pt-plugins/PT-depiler) の [action_publish.yml](https://github.com/pt-plugins/PT-depiler/blob/dcab0cf779aa364013f88b15305c7cb1285b26a8/.github/workflows/action_publish.yml) は、スケジュール実行で定期的にリリースするワークフローです。GitHub Release の作成後に 3 ストアへ順に公開し、Firefox Add-ons に添付するソースアーカイブにはそのコミットの GitHub アーカイブを使っています。

## v1 系ツールから移る人向け

Chrome Web Store API v1 向けのツールから移る場合、wepub で新たに必要になるのは Publisher ID です。v2 ではアイテムを Publisher ID と Item ID の組で指定するため、これまで使っていた 32 文字の Item ID に加えて、UUID 形式の Publisher ID を用意します。

OAuth のクライアント ID、クライアントシークレット、リフレッシュトークンによる認証は v2 でも引き続き使えます。取得手順は公式ドキュメントの [Use the Chrome Web Store API](https://developer.chrome.com/docs/webstore/using-api) を参照してください。サービスアカウントで運用している場合は、事前に取得したアクセストークンを渡すモードを使います。

## おわりに

wepub は、3 ストアへの公開を同じ流儀で扱えることと、CI で安全に動かせることを重視して作っています。使い方の詳細やフラグの一覧は [README](https://github.com/iorate/wepub#readme) にまとめてあります。不具合や要望があれば、GitHub の Issue でお知らせください。
