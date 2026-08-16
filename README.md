# GitHub Drive

GitHub リポジトリをストレージにした、OneDrive 風のファイル共有 Web アプリです。
サーバーは使わず、静的ファイルのみで動作します（GitHub Pages でホスト可能）。すべての通信は GitHub の API (`https://api.github.com`) 経由の HTTPS（443番ポート）です。

配布元: [github.com/basel5freedom/github-drive-app](https://github.com/basel5freedom/github-drive-app)（自分用にする手順は下記「3. このアプリをホストする」参照）

## できること

- フォルダ作成 / アップロード / ダウンロード / 削除
- フォルダ階層のナビゲーション（パンくずリスト）
- ドラッグ＆ドロップでのアップロード

## アーキテクチャ

- フロントエンドのみ（HTML / CSS / 素の JavaScript、ビルド不要）
- GitHub の [Contents API](https://docs.github.com/ja/rest/repos/contents) を直接ブラウザから呼び出してファイルを読み書き
- ファイルは指定したリポジトリへの git コミットとして保存されます
- 認証には Personal Access Token (PAT) を使用し、**ブラウザの localStorage にのみ**保存します（サーバーには一切送信されません）

### リポジトリ構成の推奨

アプリを配信するリポジトリと、ファイルを保存するリポジトリは **分けることを推奨**します。

- `github-drive-app`（このリポジトリ）: GitHub Pages で静的サイトとして公開
- `my-drive-storage`（別リポジトリ）: アップロードしたファイルの保存先

同じリポジトリを兼用することも可能ですが、アップロードのたびに Pages の自動デプロイが走ったり、アプリのソース履歴とファイル履歴が混在して肥大化するため、分離をおすすめします。

## セットアップ手順

### 1. ファイル保存用リポジトリを作成する

GitHub で新しいリポジトリ（例: `my-drive-storage`）を作成します。

- 公開範囲: 個人利用なら **Private** を強く推奨（Public だと誰でも中身が見えます）
- 「Add a README file」にチェックを入れて作成すると、`main` ブランチが最初から存在するので安心です

### 2. Personal Access Token (PAT) を発行する

[GitHub Settings > Developer settings > Fine-grained tokens](https://github.com/settings/personal-access-tokens/new) から発行します。

- **Repository access**: 「Only select repositories」→ 手順1で作ったリポジトリのみを選択
- **Permissions**: 「Contents」を **Read and write** に設定（他は不要）
- **Expiration**: 必要に応じて短めの期限を設定し、期限が切れたら再発行してください

⚠️ classic トークン（`ghp_...`）も動作しますが、権限がアカウント全体に及ぶため fine-grained トークンを強く推奨します。

### 3. このアプリをホストする

#### 方法A: テンプレートから自分のリポジトリを作る（推奨・一番簡単）

1. [github.com/basel5freedom/github-drive-app](https://github.com/basel5freedom/github-drive-app) を開く
2. 緑色の **Use this template** ボタン →「Create a new repository」を選択
3. リポジトリ名を入力し、公開範囲（Public / Private どちらでもOK。GitHub Pages は無料プランでは Public リポジトリのみ利用可能な点に注意）を選んで **Create repository**
4. 作成されたリポジトリの Settings > Pages を開き、Source を「Deploy from a branch」、Branch を `main` / `(root)` にして Save
5. 1〜2分後、`https://<あなたのユーザー名>.github.io/<リポジトリ名>/` でアクセスできるようになります

「Fork」ではなく「Use this template」を使うことで、元リポジトリと履歴・PRのつながりを持たない独立したコピーになります。

#### 方法B: ファイルを手動でアップロードする

このフォルダ一式（`index.html`, `assets/`）を任意の GitHub リポジトリにアップロードし、Settings > Pages で GitHub Pages を有効化する方法でも構いません。手順は方法Aの手順4以降と同じです。

#### ローカルで試す場合

任意の静的サーバーで配信してください（`type="module"` を使っているため `file://` では動作しません）。例:

```
npx serve .
```

### 4. アプリ側で接続設定をする

1. 公開した URL にブラウザでアクセス
2. 右上の ⚙️ をクリック
3. 手順2で発行した PAT、手順1のオーナー名・リポジトリ名・ブランチ名（`main` など）を入力して「接続して保存」

以降はブラウザにトークンが保存され、次回アクセス時も自動的に接続されます。

## パスワード保護(簡易・任意)

URL を知っているだけの通りすがりの人がアプリ画面を開けてしまうのを防ぐため、簡易的なパスワード画面を追加できます。

警告: これは正規の認証ではありません。サーバーを持たない静的サイトのため、パスワード判定のコードは全てブラウザ側(誰でも見える場所)で動きます。開発者ツールで回避することも技術的には可能な、あくまで「抑止」レベルの鍵です。実際のファイルの安全性は PAT(対象リポジトリのみ・Contents 権限のみに絞ったもの)によって守られています。

設定手順:

1. 公開後のアプリ URL(または `npx serve .` で立てたローカルURL)にブラウザでアクセスし、F12 で開発者ツールの Console タブを開く
2. 以下を実行し、使いたいパスワードのハッシュ値を生成する(`your-password` を実際のパスワードに置き換える)

   ```js
   crypto.subtle.digest("SHA-256", new TextEncoder().encode("your-password"))
     .then(buf => console.log(Array.from(new Uint8Array(buf)).map(b => b.toString(16).padStart(2, "0")).join("")))
   ```

3. 出力された64文字の文字列をコピーし、`assets/config.js` の `ACCESS_PASSWORD_HASH` に貼り付ける

   ```js
   export const ACCESS_PASSWORD_HASH = "ここに貼り付けたハッシュ値";
   ```

4. アプリを配信しているリポジトリの `assets/config.js` を更新後の内容で上書き(Commit changes)

以降、初回アクセス時(ブラウザタブを閉じるまで有効)にパスワード入力画面が表示されます。`ACCESS_PASSWORD_HASH` を空文字 `""` に戻せば無効化されます。平文のパスワードは絶対にコミットしないでください(ハッシュ値のみを保存する設計です)。

## セキュリティ上の注意

- PAT はブラウザの localStorage に平文で保存されます。共有 PC や信頼できない端末では使用しないでください。
- 必ず対象リポジトリのみ・Contents 権限のみに絞った fine-grained PAT を使ってください。
- ファイル保存用リポジトリは Private を推奨します。Public リポジトリを使うと、このアプリを介さずとも GitHub 上で誰でもファイルを閲覧できてしまいます。
- このアプリの URL（GitHub Pages）自体は誰でも開けますが、有効な PAT を持たない限りファイル一覧の取得や操作はできません。パスワード画面はこれに加えた簡易的な抑止策です。

## 制限事項

- 1ファイルあたり約 90MB まで（GitHub Contents API の実用上の上限に基づく安全マージン）。100MB を超えるファイルは GitHub 側で拒否されます。
- GitHub には空フォルダの概念がないため、新規フォルダ作成時に内部的に `.gitkeep` という空ファイルを置いています（一覧には表示されません）。
- アップロード・削除のたびに GitHub 上へ1コミットが作成されます。
