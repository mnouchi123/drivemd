# DriveMD Work History - 2026-03-31

## 1. 目的

2026-03-30 時点で未解決だった iOS Safari / iOS Chrome の認証問題を解消し、
全プラットフォームで DriveMD を使えるようにする。
あわせてフォルダのブックマーク機能を追加する。

---

## 2. 実施したこと

### 2-1. iOS 認証問題の原因特定と修正

#### 原因1: `sessionStorage` がリダイレクト後にクリアされる

iOS Safari（および WebKit ベースの iOS Chrome）は、クロスオリジンリダイレクト後に
`sessionStorage` をクリアする仕様がある。

OAuth の CSRF 対策として使っていた `state` を `sessionStorage` に保存していたため、
Google から戻ってきたときに state が読めず「state が一致しません」エラーになっていた。

**対応:** `sessionStorage` → `localStorage` に変更。

#### 原因2: `redirect_uri` の末尾スラッシュ不一致

`redirect_uri` を `window.location.origin + window.location.pathname` で動的生成していたが、
iOS でのアクセス URL によっては末尾スラッシュがなく、GCP 登録 URI と不一致になる可能性があった。

**対応:** `getRedirectUri()` 関数を追加し、末尾スラッシュを常に付与するよう正規化。

#### 原因3: Client ID が URL 形式で保存されていた（根本原因）

iOS で Client ID を手入力した際に `http://864457594326-xxx.apps.googleusercontent.com/` と
URL 形式で入力されていた（`http://` プレフィックスと末尾 `/` が余分についていた）。
これが Google に送られ `401: invalid_client` を引き起こしていた。

デバッグトーストを追加して `client_id` と `redirect_uri` の実際の値を確認し、特定した。

**対応:**
- `sanitizeClientId()` 関数を追加し、保存時に `http://` と末尾 `/` を自動除去
- `.apps.googleusercontent.com` で終わるかバリデーション
- 不正形式の場合はエラートーストを表示

#### 原因4: 有効期限チェックが常に失敗していた

state に 5分の有効期限を設けていたが、`drivemd_oauth_state_exp` キーが
iOS で正しく読み取れず `exp = 0` となり、`Date.now() > 0` が常に true になっていた。

**対応:** state の一致チェックだけで十分セキュアなため、有効期限チェックを削除。

### 2-2. フォルダブックマーク機能追加

- ファイル一覧でフォルダにもブックマークボタン（★）を表示
- ブックマーク保存形式を `{id: name}` から `{id: {name, isFolder}}` に変更
  - 旧形式との後方互換あり（string の場合はファイルとして扱う）
- ブックマーク一覧でフォルダはフォルダアイコンで表示
- フォルダブックマークをタップすると `enterFolder()` でそのフォルダへ移動
- `enterFolder()` がブックマークタブから呼ばれた場合、自動的にファイルタブへ切り替わるよう修正
  （修正前は `activeTab === 'bookmarks'` のままで `renderFileList` がブックマーク一覧を再描画していた）

### 2-3. ホーム画面への追加

iOS Safari から「共有 → ホーム画面に追加」でインストール。
`apple-mobile-web-app-capable` 設定により、フルスクリーン PWA として起動することを確認。

---

## 3. デバッグ手順の記録

### Client ID / redirect_uri の確認方法

`startRedirectAuth()` にトーストを追加して実際の送信値を画面に表示した。

```javascript
showToast('redirect_uri: ' + redirectUri, 6000);
showToast('client_id: ' + APP.clientId, 8000);
```

### state 一致状況の確認方法

`handleRedirectAuthResult()` にトーストを追加して state 値を比較表示した。

```javascript
showToast('DBG state=' + state.slice(0,6) + ' expected=' + (expectedState ? expectedState.slice(0,6) : 'null'), 6000);
```

結果: `state=ibjq2j expected=ibjq2j`（一致）→ 有効期限チェックが原因と特定。

---

## 4. コミット一覧

| コミット | 内容 |
|----------|------|
| `2ae1dad` | Fix iOS OAuth: use localStorage for state and normalize redirect_uri |
| `e34a277` | Add debug toast to diagnose iOS invalid_client (temporary) |
| `620a361` | Improve debug: show full client_id and redirect_uri separately |
| `eada417` | Fix invalid_client: sanitize client_id input and validate format |
| `f7d671f` | Debug state mismatch: show state values on return from Google |
| `aee54dc` | Fix state mismatch: remove expiry check that was always failing on iOS |
| `ba413e5` | Add folder bookmark support |
| `5ccec42` | Fix folder bookmark navigation: switch to files tab on enterFolder |

---

## 5. 結果

| 項目 | 状態 |
|------|------|
| Desktop Chrome 認証・一覧表示 | 完了 |
| iOS Safari 認証・一覧表示 | 完了 |
| iOS Chrome 認証・一覧表示 | 完了 |
| iOS Safari ホーム画面追加・PWA 起動 | 完了 |
| フォルダブックマーク | 完了 |

MVP としてすべてのプラットフォームで動作確認済み。

---

## 6. 追加実装: テーマ切替・フォントサイズ調整（同日）

### 背景
- 画面がダークテーマ固定で文字が見にくい
- フォントサイズを状況に応じて変えたい

### 実装内容

**テーマ切替（ダーク↔ライト）**
- CSS カスタムプロパティ（`--bg`, `--text-primary` など）を一元管理していたため、`body.light` クラスで変数をまるごと上書きする方式で実装
- ライトテーマは白ベース・ノート帳風（`#fafaf7` 背景、`#2a2420` テキスト）
- ファイル一覧・ビューア両方のヘッダーに ☀/🌙 ボタンを追加
- ダーク時は太陽アイコン、ライト時は月アイコンに切り替わる
- `localStorage`（`drivemd_theme`）に保存、起動時に `initTheme()` で復元

**フォントサイズ調整**
- `:root` に `--font-size: 15px` 変数を追加し `body { font-size: var(--font-size) }` で全体に適用
- ビューアヘッダーに `A-` / `A+` ボタンを追加
- `changeFontSize(delta)` が `getComputedStyle` で現在値を読み、±1px ずつ変更（11〜22px の範囲）
- `localStorage`（`drivemd_fontsize`）に保存、起動時に復元

**ホーム画面の再登録不要**
- GitHub Pages は URL が変わらないため、既存のホーム画面アイコンから最新版が読み込まれる
- キャッシュが残る場合は強制リロードまたは Safari のデータ消去で対応

### コミット
- `8b7233e` Add light theme toggle and font size adjustment

---

## 7. 次回の着手候補

1. Service Worker によるオフライン対応
2. OAuth 同意画面の本番公開（テストモード解除）
3. Client ID の自動設定（埋め込みまたは共有設定）
