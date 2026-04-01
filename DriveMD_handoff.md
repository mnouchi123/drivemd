# DriveMD 認証改善 — VSCode + Codex 引き継ぎコンテキスト

作成日: 2026-03-31

---

## 1. 背景・やりたいこと

DriveMD（GitHub Pages上のシングルHTMLアプリ）の認証を改善したい。

**現状の問題:**
- Google OAuth 2.0 の Implicit Flow を使用
- アクセストークンのみ取得（リフレッシュトークンなし）
- アプリ再読み込みのたびに再認証が必要
- GCP の OAuth 同意画面がテストモードのため、**7日ごとに必ず再認証が必要**

**ゴール:**
- アプリを開いたらログインボタンを押さずに、すぐファイル一覧が表示される
- 認証を意識しない運用にする

---

## 2. 結論として採用するアーキテクチャ

**Cloud Run プロキシ経由でサービスアカウント認証に切り替える。**

```
DriveMD（GitHub Pages / index.html）
    ↓ Drive API リクエスト（Bearer トークン不要）
Cloud Run プロキシ（新規追加・既存プロジェクトに相乗り）
    ↓ サービスアカウントで認証
Google Drive API v3
```

---

## 3. 既存環境（スクとち）

| 項目               | 内容                                                        |
| ------------------ | ----------------------------------------------------------- |
| GCP プロジェクト   | `ios-apps-mnouchi123`                                       |
| 既存サービス       | スクとち（不動産ポータルスクレイパー）が Cloud Run で稼働中 |
| 月額費用           | 約1,000円（Cloud Run リクエスト課金）                       |
| 運用               | VS Code + Codex が担当                                      |
| サービスアカウント | スクとち用が既存（流用可能か要確認）                        |

**DriveMD の追加コストはほぼゼロ**（個人利用レベルのリクエスト数は無料枠内）

---

## 4. DriveMD 現状スペック

| 項目             | 内容                                     |
| ---------------- | ---------------------------------------- |
| リポジトリ       | https://github.com/mnouchi123/drivemd    |
| 公開URL          | https://mnouchi123.github.io/drivemd/    |
| GCP プロジェクト | `ios-apps-mnouchi123`（スクとちと同じ）  |
| アーキテクチャ   | シングル HTML（index.html）SPA           |
| 認証             | Google OAuth 2.0 Implicit Flow（現在）   |
| Drive API        | Google Drive API v3 直接呼び出し（現在） |
| 利用者           | オーナー1名のみ（Masahiro）              |

---

## 5. 必要な作業

### Step 1: サービスアカウントの確認・準備
- 既存のスクとち用サービスアカウントが流用できるか確認
- できなければ `ios-apps-mnouchi123` に新規作成
- サービスアカウントに **Google Drive API** のスコープを付与
- DriveMD で使いたい Drive フォルダをサービスアカウントのメールアドレスに共有（閲覧者または編集者）

### Step 2: Cloud Run プロキシの実装・デプロイ
既存スクとちプロジェクトに相乗りして、以下のエンドポイントを持つ薄いプロキシを追加：

```
GET  /drive/files          → Drive API ファイル一覧
GET  /drive/files/:id      → ファイル内容取得
PATCH /drive/files/:id     → ファイル更新
POST /drive/files          → ファイル新規作成
GET  /drive/userinfo       → ユーザー情報（不要になる可能性あり）
```

- サービスアカウントキーは Cloud Run の**環境変数**に格納（コードに埋め込まない）
- CORS ヘッダーに `https://mnouchi123.github.io` を許可
- 認証はシンプルに「決まったOriginからのリクエストのみ許可」で十分（個人利用のため）

### Step 3: index.html の修正
- `APP.token` と Bearer 認証ヘッダーを使っている箇所をすべてプロキシURLへの fetch に置き換え
- `gFetch()` / `gFetchText()` 関数のベースURLをプロキシに変更するだけでほぼ対応可能
- OAuth 関連のコード（`startRedirectAuth`, `handleRedirectAuthResult`, `initGIS` など）は削除
- ログイン画面は不要になるため、起動時に直接ファイル一覧を表示

---

## 6. index.html の現在の API 呼び出し箇所（修正対象）

```javascript
// ユーザー情報
fetch('https://www.googleapis.com/oauth2/v2/userinfo', { headers: { Authorization: 'Bearer ' + APP.token } })

// ファイル一覧
gFetch(`https://www.googleapis.com/drive/v3/files?q=...`)

// ファイル内容
gFetchText(`https://www.googleapis.com/drive/v3/files/${id}?alt=media`)

// ファイル更新
fetch(`https://www.googleapis.com/upload/drive/v3/files/${id}?uploadType=multipart`, { method: 'PATCH', ... })

// ファイル作成
fetch('https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart', { method: 'POST', ... })
```

これらを `https://<プロキシURL>/drive/...` に置き換えるだけ。

---

## 7. セキュリティ方針（個人利用のため簡易で可）

- プロキシは `Origin: https://mnouchi123.github.io` のリクエストのみ受け付ける
- サービスアカウントキーは Cloud Run 環境変数に格納
- GitHub リポジトリにキーは一切コミットしない

---

## 8. 確認事項（Codex への質問）

1. スクとちで使っているサービスアカウントのメールアドレスは何か？
2. 既存の Cloud Run サービスにプロキシを相乗りさせるか、別サービスとして立てるか？
3. プロキシの言語は既存スクとちに合わせる（Python？Node.js？）

---

## 9. 参考ファイル

- `index.html`（DriveMD本体）: 添付または https://github.com/mnouchi123/drivemd
- `DriveMD_APP_PRD.md` v1.3: Google Drive の DriveMD フォルダ内に格納済み
