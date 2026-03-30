# DriveMD デプロイメント PRD

## 概要

DriveMD は Google Drive 上の Markdown ファイルを閲覧・編集するための PWA（Progressive Web App）。Safari でホーム画面に追加して使う iOS 向けアプリケーション。

**ホスティング:** GitHub Pages  
**認証:** Google Identity Services（OAuth 2.0）  
**対象ユーザー:** 自分 + 数人の友人（GCP テストユーザーモード）

---

## 成果物

単一の HTML ファイル `index.html`（添付の `drivemd.html`）を GitHub Pages にデプロイする。

---

## タスク一覧

### Phase 1: GitHub Pages デプロイ（Claude Code で自動化）

以下を Claude Code で実行すること。

#### 1-1. リポジトリ作成

```bash
gh repo create drivemd --public --clone
cd drivemd
```

#### 1-2. HTML ファイル配置

- 添付の `drivemd.html` を `index.html` としてリポジトリルートに配置する

```bash
cp drivemd.html index.html
```

#### 1-3. GitHub Pages 有効化

```bash
git add .
git commit -m "Initial deploy: DriveMD"
git push origin main
gh api repos/{owner}/drivemd/pages -X POST -f source.branch=main -f source.path=/
```

#### 1-4. デプロイ URL 確認

```bash
gh api repos/{owner}/drivemd/pages --jq '.html_url'
```

この URL を控えておく（Phase 2 で使用する）。  
形式: `https://{username}.github.io/drivemd/`

---

### Phase 2: GCP OAuth 設定（手動・ブラウザ操作）

> **この Phase は GCP コンソールのブラウザ GUI 操作が必要なため、Claude Code では自動化できない。以下の手順をユーザーに案内すること。**

#### 2-1. GCP コンソールにアクセス

1. https://console.cloud.google.com/ を開く
2. 既存のスクとちプロジェクトを選択する（新規作成は不要）

#### 2-2. Google Drive API を有効化

1. 左メニュー「APIとサービス」→「ライブラリ」
2. 検索バーに `Google Drive API` と入力
3. 「Google Drive API」をクリック
4. 「有効にする」をクリック（既に有効なら「管理」と表示される → スキップ）

#### 2-3. OAuth 同意画面を設定

1. 左メニュー「APIとサービス」→「OAuth 同意画面」
2. 既に設定済みの場合は内容を確認して 2-4 へ進む
3. 未設定の場合:
   - User Type: **外部** を選択 → 「作成」
   - アプリ名: `DriveMD`
   - ユーザーサポートメール: 自分のGmail
   - デベロッパーの連絡先: 自分のGmail
   - 「保存して次へ」

4. スコープ設定画面:
   - 「スコープを追加または削除」をクリック
   - フィルタに `drive` と入力
   - `https://www.googleapis.com/auth/drive` にチェック
   - 「更新」→「保存して次へ」

5. テストユーザー画面:
   - 「+ ADD USERS」をクリック
   - 自分の Gmail アドレスを追加
   - 友人の Gmail アドレスも追加（後からでも可）
   - 「保存して次へ」→「ダッシュボードに戻る」

> **注意:** 公開ステータスは「テスト」のままでよい。テストモードでは登録したユーザー（最大100人）のみ使用可能。Google の審査は不要。

#### 2-4. OAuth クライアント ID を作成

1. 左メニュー「APIとサービス」→「認証情報」
2. 上部の「+ 認証情報を作成」→「OAuth クライアント ID」
3. アプリケーションの種類: **ウェブ アプリケーション**
4. 名前: `DriveMD`（任意）
5. 「承認済みの JavaScript 生成元」に以下を追加:
   - `https://{username}.github.io`
6. 「承認済みのリダイレクト URI」に以下を追加:
   - `https://{username}.github.io/drivemd/`
7. 「作成」をクリック
8. **表示されるクライアント ID をコピーする**（`xxxxx.apps.googleusercontent.com` の形式）

---

### Phase 3: 接続テスト

#### 3-1. ブラウザでアクセス

1. `https://{username}.github.io/drivemd/` を Safari（PC でも可）で開く
2. 画面下部の「Client ID設定」をタップ
3. Phase 2-4 でコピーしたクライアント ID を貼り付け → 「保存」
4. 「Googleアカウントで接続」をタップ
5. Google のログイン画面が表示される → 自分のアカウントでログイン
6. 「このアプリは確認されていません」警告が出る → 「詳細」→「DriveMD（安全でないページ）に移動」
7. アクセス許可を確認 → 「許可」

#### 3-2. 動作確認チェックリスト

- [ ] ログイン後、マイドライブのファイル一覧が表示される
- [ ] フォルダをタップして中に入れる
- [ ] ファイル名でのソートが動作する（名前 ↑ → 名前 ↓ → 日付 ↓ → 日付 ↑）
- [ ] 検索バーでファイル名フィルタが動作する
- [ ] .md ファイルをタップして Markdown プレビューが表示される
- [ ] プレビューが全画面で表示される
- [ ] コードブロックのシンタックスハイライトが機能する
- [ ] 目次パネル（右上ハンバーガー）が開き、見出しジャンプできる
- [ ] ブックマーク追加/削除が動作する
- [ ] ブックマークタブに保存したファイルが表示される
- [ ] 編集ボタン → テキスト編集 → 保存 → プレビューに反映される
- [ ] + ボタンで新規 .md ファイルが作成できる
- [ ] ログアウトが動作する

#### 3-3. iOS PWA としてインストール

1. iPhone/iPad の Safari で `https://{username}.github.io/drivemd/` を開く
2. 共有ボタン（□↑）をタップ
3. 「ホーム画面に追加」をタップ
4. 名前を `DriveMD` にして「追加」
5. ホーム画面のアイコンからアプリとして起動できることを確認

---

### Phase 4: 友人への共有

1. 友人に GitHub Pages の URL を送る: `https://{username}.github.io/drivemd/`
2. GCP コンソール → OAuth 同意画面 → テストユーザー に友人の Gmail を追加
3. 友人に以下を伝える:
   - URL を Safari で開く
   - 「Client ID設定」から Client ID を入力（Client ID を友人に共有する）
   - 「Googleアカウントで接続」でログイン
   - 「確認されていないアプリ」警告は「詳細」→「移動」で進む
   - ホーム画面に追加すれば PWA として使える

---

## 技術仕様

| 項目 | 値 |
|------|-----|
| ホスティング | GitHub Pages（静的 HTML） |
| 認証 | Google Identity Services (GIS) |
| OAuth スコープ | `https://www.googleapis.com/auth/drive` |
| API | Google Drive API v3 |
| Markdown レンダリング | marked.js 12.0.2 |
| シンタックスハイライト | highlight.js 11.9.0 |
| フォント | Montserrat (Google Fonts) |
| ブックマーク保存 | localStorage |
| Client ID 保存 | localStorage |

---

## 注意事項

- GCP 課金: **発生しない**（Drive API の無料枠は日次10億リクエスト、OAuth 設定も無料）
- テストモードの制限: トークンの有効期限が7日間。期限切れ後は再ログインが必要
- OAuth 同意画面が「テスト」のままだと、ログイン時に「確認されていないアプリ」警告が表示される。これは正常動作
- Client ID はアプリ HTML に埋め込まれていないため、初回のみ設定画面で入力が必要
