# pCloud OAuth2 対応 設計書（Android TV / Nova Video Player）

## 1. 背景と目的（Intent & Why）

現在、pCloud は WebDAV 経由で接続可能だが、pCloud アカウントで二要素認証（2FA）を有効にしている場合、WebDAV 接続を維持するために定期的に承認メールのリンクをクリックし直す必要がある。

本機能では、Nova Video Player（Android TV / Leanback UI）に **pCloud REST API + OAuth2 認証** をネイティブ統合する。pCloud の OAuth2 Bearer トークンはデフォルトで有効期限がなく（ユーザーが明示的に取り消すまで永続）、2FA 有効アカウントでも再認証メールなしで安定したファイル参照・動画再生・ライブラリ登録を実現できる。

## 2. ゴール・非ゴール・アンチゴール

### ゴール
- Android TV（Leanback UI）から pCloud アカウントを OAuth2 で連携し、永続アクセストークンを安全に保存できること。
- TV リモコンでの文字入力を避け、**TV 画面の QR コード → スマホのブラウザ** で 2FA 込みの認証を完結できること。
- pCloud Developer Console の新規アプリ審査が `pending` の状態でもすぐに利用・テストできるよう、スマホ連携画面に **「OAuth コード認証」** と **「アクセストークン直接入力（rclone JSON まるごと貼り付け対応）」** の2モードを備えること。
- 既存の WebDAV 接続と同等の機能を提供すること：
  - フォルダ・ファイル一覧表示
  - 動画ストリーミング再生（HTTP Range によるシーク・レジューム・字幕読み込み・サムネイル生成）
  - ライブラリへのインデックス登録（スクレイパーによるポスター・メタデータ取得、定期再スキャン）
  - 基本的なファイル操作（存在確認・アップロード/書き込み・リネーム・移動・削除・フォルダ作成）
- US リージョン（`api.pcloud.com`）および EU リージョン（`eapi.pcloud.com`）の両方に対応すること。

### 非ゴール
- スマホ・タブレット用 UI（非 Leanback の `BrowserByNetwork` 等）への専用ログイン UI 追加（対象端末は Android TV のみ）。
- pCloud 独自の動画トランスコーディング API（`getvideolink`）や Crypto フォルダ（クライアント側暗号化）のサポート（オリジナルファイルを `getfilelink` 経由でそのままストリーミング再生する）。
- 複数 pCloud アカウントの同時管理（まずは US/EU 各エンドポイントにつき 1 アカウントを `NetworkCredentialsDatabase` で管理。将来的にユーザー識別子をパスに含める拡張は妨げない）。

### アンチゴール
- Android TV 上でリモコンを使ってメールアドレス・パスワード・2FA コード・長いトークンを直接入力させること。
- `client_secret` やユーザーの `access_token` を Git リポジトリにコミットしたり、平文ログに出力したりすること。
- 既存の SMB / FTP / SFTP / WebDAV / UPnP の動作や URI バリデーションを壊すこと。

## 3. 採用アプローチと比較（Approaches）

Android TV での pCloud OAuth2 認証方式として以下の 3 案を比較した（pCloud には OAuth2 Device Authorization Flow が存在しないため、TV 向け工夫が必要）：

| 案 | 概要 | トレードオフ |
|---|---|---|
| **案A（採用）** | `FileCoreLibrary` に `pcloud://` ネイティブ実装 ＋ TV ローカル HTTP サーバー＆QR コードでスマホから OAuth コード認証（＋トークン直接入力タブ併設） | 実装量は中規模だが、TV リモコン入力がゼロ。2FA やスマホのパスワードマネージャがそのまま使え、さらにアプリ審査 `pending` 中でもトークン直接入力で即日利用できる。 |
| 案B | `pcloud://` ネイティブ実装 ＋ TV 上の WebView で直接 OAuth ログイン | TV 単体で完結するが、リモコンでのメール/パスワード/2FA 入力が苦行であり、TV 端末によって WebView のフォーカス制御が不安定。 |
| 案C | `pcloud://` ネイティブ実装 ＋ トークン貼り付けのみ | 実装は最小だが、毎回 PC 等でトークンを取得する手間がかかり、将来の正規 OAuth フローに発展しにくい。 |

**決定理由**: ユーザーは Android TV のみでの利用を想定しており、リモコン入力の煩わしさを排除しつつ、現在 pCloud アプリ申請が `pending` である状況でも即座に使えるようにするため、**案A（トークン直接入力フォールバック併設）** を採用する。

## 4. アーキテクチャ設計

### 4.1 URI 設計と認証情報保存

- **URI スキーム**: `pcloud://`
- **URI 形式**:
  - US リージョン: `pcloud://api.pcloud.com/<path>`（ルートは `pcloud://api.pcloud.com/`）
  - EU リージョン: `pcloud://eapi.pcloud.com/<path>`（ルートは `pcloud://eapi.pcloud.com/`）
- **既存仕組みとの親和性**:
  - ホスト部に実際の pCloud API ホスト名（`api.pcloud.com` / `eapi.pcloud.com`）を入れることで、`UriUtils.isValidHost()` および `UriUtils.isIndexable()`（`/` が 3 つ以上）を既存ロジックのまま満たせる。
  - 認証情報は既存の `NetworkCredentialsDatabase`（SQLite + 暗号化）に保存する（DB スキーマ変更不要）：
    - `uriString`（キー）: `pcloud://api.pcloud.com/` または `pcloud://eapi.pcloud.com/`
    - `username`: pCloud アカウントのメールアドレス（`userinfo` API から取得）
    - `password`: OAuth2 `access_token`（既存の `encrypt()` で暗号化保存）
    - `domain`: 空文字 `""`

### 4.2 FileCoreLibrary 層（`com.archos.filecorelibrary.pcloud`）

`webdav` パッケージの設計に倣い、以下のクラスを新設する（既存の `OkHttpClient` と Android 標準 `org.json` を使用し、追加の HTTP/JSON 依存ライブラリは増やさない）：

1. **`PcloudUtils`**:
   - シングルトン（`WebdavUtils` 同様、`CustomApplication` で初期化）。
   - `NetworkCredentialsDatabase` から URI に対応する `access_token` を取得し、`Authorization: Bearer <token>` ヘッダーを付与する `OkHttpClient` をキャッシュ・管理する。
   - `getfilelink` で取得した一時ダウンロード URL の軽量キャッシュ（TTL: 例 10 分、最大件数制限あり）を保持し、動画シーク時に毎回 API を叩くオーバーヘッドを防ぐ。
   - pCloud API のエラーコード（`result != 0`）を解析し、`1000`（Log in required）や `2000`（Log in failed）の場合は `AuthenticationException` をスローする。
   - API リクエストおよびエラーログには `access_token` や `client_secret` を絶対に出力しない。

2. **`PcloudFile2` (`extends MetaFile2`)**:
   - pCloud JSON メタデータ（`name`, `isfolder`, `size`, `modified`, `fileid`/`folderid`）を保持。
   - `fromUri(Uri uri)` は `GET https://<host>/stat?path=<encodedPath>` を呼び出してインスタンス化。
   - `getRawListerInstance()` → `new PcloudRawLister(getUri())`
   - `getFileEditorInstance(Context)` → `new PcloudFileEditor(getUri())`

3. **`PcloudRawLister` (`extends RawLister`)**:
   - `GET https://<host>/listfolder?path=<encodedPath>&nofiles=0` を実行。
   - レスポンスの `metadata.contents` を走査し、`ArrayList<MetaFile2>` として返す。
   - パス結合時は日本語・スペース・記号を含むファイル名を正しく URI エンコードする。

4. **`PcloudListingEngine` (`extends ListingEngine`)**:
   - `WebdavListingEngine` と同様に非同期スレッドで `PcloudRawLister` を実行し、ソート・フィルタリング・キャンセル・認証エラー通知（`ErrorEnum.ERROR_AUTHENTICATION`）を行う。

5. **`PcloudFileEditor` (`extends FileEditor`)**:
   - `getInputStream()` / `getInputStream(long from)`:
     - `PcloudUtils` のキャッシュまたは `GET https://<host>/getfilelink?path=<encodedPath>` からダウンロード URL（`https://<hosts[0]><path>`）を取得。
     - `OkHttpClient`（認証ヘッダーなし、pCloud ダウンロードリンクは署名済み一時 URL）で `GET` し、`from >= 0` の場合は `Range: bytes=<from>-` ヘッダーを付与。
     - 401/403/410 等の期限切れ応答だった場合はリンクキャッシュを破棄して 1 回だけ `getfilelink` を再取得・リトライする。
     - `Content-Length`（または `Content-Range` の総サイズ）から `mLength` を設定して `InputStream` を返す。
   - `getOutputStream()`:
     - ダウンロードした字幕（`.srt` 等）や NFO / ポスター書き出し、ファイルコピーに対応するため実装する。
     - Android TV のメモリ制限を考慮し、書き込まれたデータをアプリキャッシュ配下の一時ファイルへ書き出し、`close()` 時に `POST https://<host>/uploadfile?path=<encodedParentPath>&filename=<encodedFileName>&nopartial=1` へ `OkHttp` の `MultipartBody`（ファイルストリーム送信）でアップロードする（同名ファイルは上書き、送信後に一時ファイルを確実に削除、`result != 0` は `IOException`）。
   - `exists()`: `stat?path=...` を呼び、`result == 0` なら `true`、`2005`/`2009`（not found）なら `false`。
   - `mkdir()`: `createfolder?path=...`
   - `delete()`: ファイルなら `deletefile?path=...`、フォルダなら `deletefolderrecursive?path=...`
   - `move(Uri)` / `rename(String)`: `renamefile` / `renamefolder`

6. **既存 Factory / Utils への登録**:
   - `FileEditorFactory`, `ListingEngineFactory`, `RawListerFactory`, `MetaFile2Factory`（`defaultPortForProtocol("pcloud")` → `443`）
   - `FileUtils` のリモートスキーム判定に `"pcloud"` を追加

### 4.3 MediaLib 層

- `UriUtils`: `networkSharesTypes`（および `sImplementedByFileCore` / `sIndexableSchemes`）に `"pcloud"` を追加。
- `SmbProxy`: `needToStream("pcloud")` を `true` に追加（`StreamOverHttp` 経由で `PcloudFileEditor` からシーク対応ストリーミング再生）。
- `NetworkScannerServiceVideo`: `getStorageId()` に `"pcloud://"` を追加（インデックス番号 `10 + LIGHT_INDEX_STORAGE_ID_OFFSET`）。
- `RemoteStateService`: `DATA LIKE 'pcloud://%'` およびリモートサーバー存在チェックスキップ対象に `"pcloud"` を追加。

### 4.4 Video 層（Android TV Leanback UI ＋ ローカル認証サーバー）

1. **導線（`NetworkRootFragment`）**:
   - ネットワーク画面の「ネットワークショートカット」行に「pCloud を接続（OAuth）」アイテムを追加（または既存の「ネットワークサーバーを参照」と並べて配置）。
   - クリックすると `PcloudAuthActivity` を起動する。
   - 認証完了後は `ShortcutDb` に pCloud ルート（`pcloud://<host>/`）をショートカット登録し、そのまま `PcloudListingActivity`（`WebdavListingActivity` と同様に `NetworkListingFragment` を表示）へ遷移する。

2. **TV 認証画面（`PcloudAuthActivity`）**:
   - Leanback 向けレイアウト：左側に QR コード（`com.google.zxing:core` で `Bitmap` 生成）、右側にローカル URL（`http://<TV_LAN_IP>:<PORT>/`）、接続手順、現在のステータス（「スマホからの入力を待っています…」「認証確認中…」「接続完了！」）を表示。
   - 「戻る」ボタンまたは画面終了時にローカル HTTP サーバーを確実に停止する。
   - Wi-Fi/Ethernet から TV の IPv4 アドレスを取得する（取得できない場合はネットワーク未接続の案内を表示）。

3. **ローカル HTTP サーバー（`PcloudAuthServer`）**:
   - Java 標準の `ServerSocket` による軽量シングルスレッド／スレッドプール HTTP サーバー（新規 HTTP サーバーライブラリ不要）。
   - ポートは空きポート（例: `8765` を試し、使用中なら `0` で動的割り当て）を使用。
   - `GET /`: スマホ向けレスポンシブ HTML ページを返す（外部 CDN 不要のインライン HTML/CSS/JS）。
     - **タブ 1: OAuth コード認証（推奨・アプリ審査通過後）**
       - `client_id` / `client_secret` 入力欄（`local.properties` の `pcloud.client_id` / `pcloud.client_secret` がビルド時に `BuildConfig` へ入っていれば自動入力・折りたたみ可能）。
       - 「pCloud でログイン（別タブで開く）」ボタン：
         `https://my.pcloud.com/oauth2/authorize?client_id=<id>&response_type=code`（`redirect_uri` 省略 → pCloud が画面に認証コードを表示）
       - 表示された認証コード（`code`）の貼り付け欄＋リージョン選択（自動 / US / EU）＋「送信」ボタン。
     - **タブ 2: アクセストークン直接入力（審査 `pending` 中でも利用可能）**
       - `access_token`（または `rclone authorize "pcloud"` が出力する JSON 文字列まるごと）の貼り付け欄＋リージョン選択（自動 / US / EU）＋「送信」ボタン。
   - `POST /auth/code`:
     - TV 側で `oauth2_token` API を実行して `access_token` を取得（リージョン「自動」時は US `api.pcloud.com` → 失敗時に EU `eapi.pcloud.com` を試行）。
     - 取得したトークンで `userinfo` API を検証し、成功したら認証情報を保存して成功 HTML を返し、TV 側コールバックを呼ぶ。
   - `POST /auth/token`:
     - 入力が JSON（例: `{"access_token":"...","hostname":"eapi.pcloud.com"}`）なら `access_token` と `hostname` を自動抽出。単体文字列ならそのままトークンとして使用。
     - リージョンが未指定/自動なら `api.pcloud.com/userinfo` → `eapi.pcloud.com/userinfo` の順に検証して正しいホストを自動確定。
     - 成功したら認証情報を保存して成功 HTML を返し、TV 側コールバックを呼ぶ。

4. **その他 `Video` 統合箇所**:
   - `AndroidManifest.xml`: `pcloud` スキームの `<data android:scheme="pcloud" />` 追加、`PcloudListingActivity`・`PcloudAuthActivity` の登録。
   - `ListingActivity.getActivityForUri()`: `"pcloud"` → `PcloudListingActivity.class`
   - `CustomApplication`: `PcloudUtils.getInstance(mContext)` の初期化。
   - `VideoUtils`: `pcloud` 用のフォルダ/ショートカットアイコン（既存 `folder_shortcut_webdav` 等の流用または専用アイコン）。
   - `Video/build.gradle`:
     - `com.google.zxing:core` の追加（QR コード生成用）。
     - `local.properties` から任意で `pcloud.client_id` / `pcloud.client_secret` を読み `BuildConfig.PCLOUD_CLIENT_ID` / `BuildConfig.PCLOUD_CLIENT_SECRET` に注入（未設定時は空文字 `""`）。
   - 文字列リソース（`values/strings.xml`, `values-ja/strings.xml`）の追加。

## 5. エラーハンドリング・セキュリティ考慮

- **トークン失効（エラー `1000`/`2000`）**: `ListingEngine` が `ERROR_AUTHENTICATION` を返し、再認証を促す（`PcloudAuthActivity` を再度開いて上書き保存すれば復旧）。
- **ローカル HTTP サーバーの安全性**: 認証画面（`PcloudAuthActivity`）が開いている間だけ起動し、認証成功または画面クローズ時に即座に `ServerSocket.close()` する。
- **機密情報の保護**: `access_token` と `client_secret` はログに出力せず、`NetworkCredentialsDatabase` の暗号化保存に委ねる。`local.properties` は `.gitignore` 対象のためリポジトリに漏れない。
- **日本語・特殊文字ファイル名**: pCloud API の `path` パラメータは UTF-8 URL エンコードして送信し、`Uri` 生成時は `Uri.Builder.appendPath()` を用いて二重エンコード・未エンコードを防ぐ。

## 6. 実装者に委ねる事項（Left to the Builder）

- `PcloudAuthActivity` の具体的な XML レイアウト調整（Leanback テーマ・余白・文字サイズ）。
- `PcloudAuthServer` の HTML/CSS スタイリング（スマホで見やすいシンプルなダーク/ライトフォーム）。
- pCloud 用アイコンを既存の WebDAV アイコン流用にするか、専用ベクタードロワブルを作るかの細部。
- 単体テストの構成（pCloud JSON レスポンスパース、`rclone` JSON 抽出、URI/パス変換、エラーコード→`AuthenticationException` 変換など）。
