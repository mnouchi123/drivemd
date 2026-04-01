# DriveMD - プロダクト要件定義書 (PRD)

**バージョン:** 1.3
**最終更新:** 2026-03-31
**ステータス:** MVP完了 / 全プラットフォーム動作確認済み・テーマ＆フォントサイズ調整対応

---

## 1. プロダクト概要

DriveMD は Google Drive 上の Markdown ファイルを快適に閲覧・編集するためのシングル HTML アプリ。GitHub Pages にデプロイし、iPad / iPhone の Safari からホーム画面追加して使うことを前提にしている。

解決したい課題:

- Google Drive 上の Markdown を大きな表示領域で読みたい
- ファイル名 / 更新日でソートしたい
- ファイル名検索をしたい
- 最小構成で Markdown の閲覧・編集・新規作成をしたい

---

## 2. 現在の運用前提

- 主利用者: 自分
- 将来的には少人数共有も可能
- 当面は GCP の OAuth 同意画面を `テスト` ステータスで運用
- 配布先は GitHub Pages

現在の公開先:

- リポジトリ: `https://github.com/mnouchi123/drivemd`
- 公開 URL: `https://mnouchi123.github.io/drivemd/`
- GCP Project: `iOS Apps`
- GCP Project ID: `ios-apps-mnouchi123`

---

## 3. 技術スタック

| 項目 | 技術 |
|------|------|
| アーキテクチャ | シングル HTML ファイル（SPA） |
| ホスティング | GitHub Pages |
| 認証 | Google OAuth 2.0 |
| API | Google Drive API v3 |
| Markdown レンダリング | marked.js 12.0.2 |
| シンタックスハイライト | highlight.js 11.9.0 |
| フォント | Montserrat + Hiragino Sans fallback |
| 永続化 | localStorage |
| サーバーサイド | なし |

---

## 4. 認証・認可

### 4-1. 認証方式

現在の実装は、Google OAuth 2.0 のクライアントサイド認証を使い、同一タブのリダイレクトでアクセストークンを受け取る方式を採用している。

- ユーザーが `Googleアカウントで接続` を押す
- Google の認証画面へ同一タブで遷移
- 認証後に `https://mnouchi123.github.io/drivemd/` へ戻る
- URL fragment に含まれる `access_token` をアプリ側で取得
- 取得したアクセストークンで Google Drive API を呼ぶ

補足:

- 初回は `Client ID設定` に OAuth Client ID を手入力する
- `Client ID` は `localStorage` に保存する（入力時に `http://` プレフィックス・末尾スラッシュを自動除去、形式バリデーションあり）
- OAuth state は `localStorage` に保存する（iOS Safari はクロスオリジンリダイレクト後に `sessionStorage` をクリアするため）
- `redirect_uri` は末尾スラッシュを自動付与して GCP 登録 URI と一致させる
- Desktop Chrome / iOS Safari / iOS Chrome すべてで動作確認済み

### 4-2. OAuth スコープ

```text
https://www.googleapis.com/auth/drive
```

Google Drive 全体への読み書きアクセス。閲覧、編集、新規作成に必要。

### 4-3. Client ID 管理

- 初回利用時に設定モーダルから入力
- `localStorage` キー: `drivemd_client_id`
- アプリ本体には埋め込まない
- Web application タイプの OAuth client を使用する

### 4-4. GCP 側の必須設定

- OAuth クライアント種別: `Web application`
- Authorized JavaScript origins: `https://mnouchi123.github.io`
- Authorized redirect URIs: `https://mnouchi123.github.io/drivemd/`
- Test user に利用 Gmail を追加
- Data Access に `https://www.googleapis.com/auth/drive` を追加

---

## 5. 機能仕様

### 5-1. ログイン画面

| 要素 | 仕様 |
|------|------|
| ロゴ | `DriveMD` テキスト |
| サブテキスト | Google Drive の Markdown を閲覧・編集 |
| 主ボタン | `Googleアカウントで接続` |
| 設定導線 | `Client ID設定` |

### 5-2. ファイル一覧画面

- サブフォルダ内でのみ戻るボタン表示
- 現在フォルダ名をヘッダー表示
- リフレッシュボタン
- ユーザーアバター表示
- ファイル名検索
- ソート切替: `名前↑ → 名前↓ → 日付↓ → 日付↑`
- タブ: `ファイル` / `ブックマーク`
- パンくず表示
- FAB で新規 `.md` 作成

### 5-3. Markdown ビューア

- 全画面表示
- 見出し、段落、リンク、リスト、引用、コード、表、画像、水平線に対応
- GFM 有効
- `breaks: true`
- TOC パネルあり
- 編集画面へ遷移可能

### 5-4. エディタ

- テキストエリアで Markdown 編集
- `files.update` による直接保存
- 保存成功後はプレビューを再描画

### 5-5. 新規作成

- `.md` を自動付与
- 初期内容は `# {filename}`
- 作成後はエディタを開く

### 5-6. ブックマーク

- `localStorage` 保存
- ファイル・フォルダ両方をブックマーク可能
- 一覧 / ビューア両方から ON/OFF
- ブックマークタブで一覧表示（フォルダはフォルダアイコン、タップでそのフォルダへ移動）
- ブックマークタブからフォルダを開くと自動的にファイルタブへ切り替わる
- 保存形式: `{id: {name, isFolder}}`（旧形式 `{id: name}` との後方互換あり）

### 5-7. テーマ切替

- ダーク（デフォルト）/ ライト（白ベース・ノート風）の2テーマ
- ファイル一覧・ビューアのヘッダーに ☀/🌙 ボタン
- ダーク時は太陽アイコン、ライト時は月アイコンを表示
- 設定は `localStorage`（`drivemd_theme`）に保存し、次回起動時も維持

ライトテーマ配色:

| 要素 | 値 |
|------|-----|
| 背景 | `#fafaf7`（メイン）/ `#ffffff`（サーフェス） |
| テキスト | `#2a2420`（プライマリ）/ `#6b6560`（セカンダリ） |
| アクセント | `#7c6f5b`（ダークと共通） |
| ボーダー | `#ddd8d0` |

### 5-8. フォントサイズ調整

- ビューアヘッダーに `A-` / `A+` ボタン
- 1px 単位で変更（範囲: 11px〜22px）
- 設定は `localStorage`（`drivemd_fontsize`）に保存し、次回起動時も維持
- `--font-size` CSS 変数で全体に反映

### 5-9. エラーハンドリング

- `認証エラー: Client ID または OAuth 設定を確認してください`
- `認証エラー: state が一致しません`
- `認証エラー: access token を受け取れませんでした`
- Client ID 形式不正時: `xxx.apps.googleusercontent.com の形式で入力してください`

---

## 6. UI / デザイン仕様

### 6-1. デザイン

| 要素 | 値 |
|------|-----|
| テーマ | ダーク / ライト切替可能 |
| 背景色（ダーク） | `#0f0f14`, `#1a1a24`, `#22222e` |
| テキスト色（ダーク） | `#e0d6c8`, `#9a9aae`, `#6a6a7e` |
| 背景色（ライト） | `#fafaf7`, `#ffffff`, `#f0ede8` |
| テキスト色（ライト） | `#2a2420`, `#6b6560`, `#9b9590` |
| アクセント色 | `#7c6f5b`（共通） |
| 角丸 | 12px / 8px |
| フォント | Montserrat / Hiragino Sans |
| フォントサイズ | `--font-size` 変数（デフォルト 15px、11〜22px） |

### 6-2. PWA メタ

- `apple-mobile-web-app-capable: yes`
- `apple-mobile-web-app-status-bar-style: black-translucent`
- `apple-mobile-web-app-title: DriveMD`
- `apple-touch-icon` は base64 SVG

### 6-3. Safe Area

- `safe-area-inset-top`
- `safe-area-inset-bottom`
- FAB とホームバーの重なり回避済み

---

## 7. API 仕様

| 用途 | エンドポイント | メソッド |
|------|---------------|----------|
| ユーザー情報取得 | `googleapis.com/oauth2/v2/userinfo` | GET |
| ファイル一覧取得 | `googleapis.com/drive/v3/files` | GET |
| ファイル内容取得 | `googleapis.com/drive/v3/files/{id}?alt=media` | GET |
| ファイル更新 | `googleapis.com/upload/drive/v3/files/{id}?uploadType=multipart` | PATCH |
| ファイル作成 | `googleapis.com/upload/drive/v3/files?uploadType=multipart` | POST |

一覧取得パラメータ:

```text
q: '{folderId}' in parents and trashed = false
fields: files(id,name,mimeType,modifiedTime,size,parents),nextPageToken
pageSize: 200
orderBy: folder,name
```

---

## 8. 動作確認状況

### 8-1. 確認済み

- GitHub Pages 公開
- GCP Project 作成
- Google Drive API 有効化
- OAuth 同意画面作成
- Web application の OAuth client 作成
- Desktop Chrome で OAuth 認証・Drive 一覧表示
- iOS Safari での OAuth 認証・Drive 一覧表示
- iOS Chrome での OAuth 認証・Drive 一覧表示
- iOS Safari ホーム画面追加・フルスクリーン起動
- ファイル・フォルダのブックマーク
- ライト / ダーク テーマ切替
- フォントサイズ A- / A+ 調整

### 8-2. 未完了

- なし（MVP 完了）

---

## 9. 既知の問題

| 問題 | 状態 |
|------|------|
| iOS Safari で `401: invalid_client` | 解決済み（Client ID の入力形式誤り・sessionStorage → localStorage 切替） |
| iOS Chrome で `401: invalid_client` | 解決済み |
| `userinfo` 取得失敗時のログイン巻き戻り | 対応済み |

---

## 10. 既知の制限事項

| 制限 | 詳細 |
|------|------|
| オフライン非対応 | Service Worker 未実装 |
| セッション永続化なし | 再読み込みや再起動で再認証が必要 |
| テストモード警告 | 未確認アプリ警告が出る |
| テストモードのトークン期限 | 7日で再ログイン |
| Markdown 以外のプレビュー | 未対応 |
| 画像相対パス | 未対応 |
| 同時編集 | 最後の保存が勝つ |

---

## 11. 今後の方針

優先順:

1. Service Worker とオフラインキャッシュ対応
2. OAuth 同意画面を本番公開（テストモード解除・7日トークン期限の解消）
3. Client ID の埋め込みまたは設定簡略化
4. 必要なら認証を backend 付き Authorization Code + PKCE flow に移行

