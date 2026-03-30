# DriveMD Work History - 2026-03-30

## 1. 目的

DriveMD を GitHub Pages に公開し、Google Drive の Markdown 閲覧・編集アプリとして動作確認する。

---

## 2. 実施したこと

### 2-1. ローカル確認

- `20_Projects/DriveMD` フォルダ内を確認
- 主ファイルは以下:
  - `drivemd.html`
  - `DriveMD_APP_PRD.md`
  - `DriveMD_DEPLOY_PRD.md`

### 2-2. GitHub 側

- `gh` CLI をこの PC にインストール
- GitHub 認証を実施
- GitHub ユーザー `mnouchi123` で接続確認
- `drivemd.html` を `index.html` として複製
- `DriveMD` フォルダで Git 初期化
- `mnouchi123/drivemd` リポジトリを作成
- `main` に push
- GitHub Pages を有効化

結果:

- Repository: `https://github.com/mnouchi123/drivemd`
- Pages URL: `https://mnouchi123.github.io/drivemd/`

### 2-3. GCP 側

- `gcloud` ログイン状態を確認
- 新規 GCP プロジェクトを作成
  - Name: `iOS Apps`
  - Project ID: `ios-apps-mnouchi123`
- `drive.googleapis.com` を有効化
- Google Cloud Console 上で OAuth 設定を実施
  - Branding 設定
  - Audience: `External`
  - Test user に `masahiro.nouchi@gmail.com` を追加
  - Data access に `https://www.googleapis.com/auth/drive` を追加
  - OAuth client を `Web application` として作成
  - Authorized JavaScript origin: `https://mnouchi123.github.io`
  - Authorized redirect URI: `https://mnouchi123.github.io/drivemd/`

### 2-4. OAuth クライアント

ローカルに確認できた Web client JSON:

- `client_secret_864457594326-3td80fq5k7trigtl1dkks1jdbu7s34ve.apps.googleusercontent.com.json`
- `client_secret_864457594326-r5u1bknncc16nce6uulp481lms0db201.apps.googleusercontent.com.json`

両方とも `web` クライアントで、以下は一致している:

- `project_id: ios-apps-mnouchi123`
- `redirect_uris: ["https://mnouchi123.github.io/drivemd/"]`
- `javascript_origins: ["https://mnouchi123.github.io"]`

### 2-5. アプリ実装修正

実施したコード変更:

1. 認証状態を見えるようにするトースト追加
2. `invalid_client` などの認証エラー文言を明示化
3. `userinfo` 取得失敗でログイン全体が巻き戻らないように修正
4. OAuth フローを popup ベースから redirect ベースに変更
5. `.gitignore` を追加して `client_secret_*.json` を Git 管理対象外にした

### 2-6. Git 反映

作成したコミット:

- `Initial deploy: DriveMD`
- `Improve OAuth diagnostics`
- `Avoid blocking login on userinfo failure`
- `Use redirect-based OAuth flow`

---

## 3. その時々の症状

### 初期

- Client ID 未設定状態ではログイン不可

### OAuth 設定途中

- `401: invalid_client`
- `アクセスをブロック: 認証エラーです`

### 認証方式変更後

- Desktop Chrome: Google の認証画面までは到達
- 未確認アプリ警告を経由して許可可能
- ただし一覧遷移の最終確認は未完

### iOS 側

- iOS Safari: `401: invalid_client`
- iOS Chrome: `401: invalid_client`
- popup 方式では `認証画面が閉じられました...`
- redirect 方式へ切り替え後も iOS で未解決

---

## 4. 現在の結論

### 成功していること

- GitHub Pages 公開
- GCP プロジェクト作成
- Drive API 有効化
- OAuth 同意画面の作成
- Test user 設定
- Drive scope 設定
- Web OAuth client 作成
- DriveMD 本体の公開

### 未解決のこと

- iOS Safari / iOS Chrome で `401: invalid_client`
- Desktop Chrome で認証後に一覧画面へ確実に進むかの最終確認

---

## 5. 次回の着手候補

優先順:

1. iOS で使っている Client ID が最新の Web client と一致しているか再確認
2. iOS の Web サイトデータを削除して再テスト
3. Desktop Chrome での一覧遷移可否を再確認
4. 必要なら OAuth 実装をさらに簡素な redirect flow に整理
5. それでも iOS が不安定なら backend 付き code flow を検討

---

## 6. 補足メモ

- `client_secret_*.json` はローカル保管のみ。GitHub に push しない
- `Client secret` は現行の静的フロント構成では使っていない
- このアプリに必要なのは `Client ID` の文字列のみ
- iOS 用の OAuth client ではなく `Web application` が正解
- `DriveMD_DEPLOY_PRD.md` は履歴資料として残し、今後の主資料は `DriveMD_APP_PRD.md` とこの `Work History` に寄せる
