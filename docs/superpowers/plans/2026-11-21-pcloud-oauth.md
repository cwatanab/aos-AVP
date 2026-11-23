# pCloud OAuth2 対応 実装計画（Android TV / Nova Video Player）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Nova Video Player（Android TV / Leanback UI）に pCloud REST API + OAuth2 認証（`pcloud://` スキーム）をネイティブ統合し、2FA 有効アカウントでも定期メール承認なしでフォルダ参照・動画ストリーミング再生・アップロード（字幕/NFO保存）・ライブラリ登録ができるようにする。

**Architecture:** `FileCoreLibrary` に `webdav` パッケージと同型の `pcloud` パッケージ（OkHttp + `org.json`）を追加し、`MediaLib` のスキーム・プロキシ・スキャナー定義へ登録する。Android TV でのログインは `Video` モジュールに `PcloudAuthActivity`（QR コード表示）＋軽量 `ServerSocket` ベースの `PcloudAuthServer` を新設し、スマホブラウザから OAuth コード認証またはトークン直接入力（`rclone` JSON 対応）で永続トークンを `NetworkCredentialsDatabase` へ保存する。

**Tech Stack:** Java 17, Android SDK (minSdk 23, targetSdk 35, compileSdk 36), OkHttp 5, Android `org.json`, AndroidX Leanback, ZXing Core (`com.google.zxing:core:3.5.3`), JUnit 4, OkHttp MockWebServer, Robolectric.

**Spec:** `docs/superpowers/specs/2026-11-21-pcloud-oauth-design.md`

## Global Constraints

- 対象プラットフォーム: Android TV（Leanback UI）のみ。非 Leanback UI のログイン画面は変更しない。
- Android バージョン要件: `minSdkVersion 23` / `targetSdkVersion 35` / `compileSdk 36`、Java 17（`lint` の `NewApi` は fatal エラー）。
- URI 形式: US は `pcloud://api.pcloud.com/<path>`、EU は `pcloud://eapi.pcloud.com/<path>`。
- 認証情報保存: 既存の `NetworkCredentialsDatabase`（DB スキーマ変更なし）に `uriString = "pcloud://<host>/"`, `username = <email>`, `password = <access_token>`（暗号化）, `domain = ""` で保存する。
- 追加ライブラリ制限: 本番依存に追加してよいのは QR 生成用の `com.google.zxing:core` のみ。HTTP/JSON/ローカルサーバーは既存の OkHttp・Android 標準 `org.json`・Java 標準 `ServerSocket` を使用する。
- セキュリティ: `access_token` および `client_secret` を Git にコミットせず、ログにも一切出力しない。ローカル HTTP サーバーは `PcloudAuthActivity` 表示中のみ起動し終了時に必ず `close()` する。
- 既存プロトコル（SMB / FTP / SFTP / WebDAV / UPnP）の動作と URI バリデーションを壊さない。

## Review Focus

1. **日本語・スペース・`#`・`%`・`?` を含むファイル名/フォルダパス**: pCloud API の `path` パラメータと `pcloud://` URI の相互変換で二重エンコードや切り詰めが起きず、正しく一覧・再生・削除できること（Task 1・Task 2 でテスト）。
2. **`getfilelink` 一時ダウンロード URL の期限切れ（HTTP 403/410）**: 再生中の一時停止や長時間シーク後に一時 URL が失効しても、キャッシュを破棄して 1 回だけ `getfilelink` を再取得しシームレスに再生を継続できること（Task 2 でテスト）。
3. **EU アカウントでのリージョン「自動」判定**: `oauth2_token` や `userinfo` が US (`api.pcloud.com`) で失敗した際、EU (`eapi.pcloud.com`) へ自動フォールバックして正しいホストを確定すること（Task 1・Task 4 でテスト）。
4. **`rclone authorize "pcloud"` 出力のコピペ揺らぎ**: 前後空白・改行・追加フィールド（`token_type`, `expiry`）・`hostname` 欠落があっても正しく抽出し、不正入力時はクラッシュせず HTTP 400 と分かりやすいエラーを返すこと（Task 4 でテスト）。
5. **アップロード（`getOutputStream()`）失敗時の一時ファイルリーク防止**: ネットワークエラーや pCloud API `result != 0` が発生してもキャッシュ内の一時ファイルが `finally` で確実に削除されること（Task 2 でテスト）。

---

## File Structure

### 新規作成
- `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudUtils.java` — シングルトン。認証トークン取得、OkHttpClient 管理、`getfilelink` キャッシュ（TTL 10分・上限64件）、URI↔pCloud パス変換。
- `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudApiClient.java` — pCloud REST API 呼び出しと JSON パース（`userinfo`, `oauth2_token`, `stat`, `listfolder`, `getfilelink`, `createfolder`, `deletefile`, `deletefolderrecursive`, `renamefile`, `renamefolder`, `uploadfile`）。
- `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudFile2.java` — `MetaFile2` 実装。
- `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudRawLister.java` — `RawLister` 実装。
- `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudListingEngine.java` — `ListingEngine` 実装。
- `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudFileEditor.java` — `FileEditor` 実装（Range 読込＋リトライ、一時ファイル経由アップロード、各種ファイル操作）。
- `FileCoreLibrary/test/com/archos/filecorelibrary/pcloud/PcloudApiClientTest.java`
- `FileCoreLibrary/test/com/archos/filecorelibrary/pcloud/PcloudFileEditorTest.java`
- `MediaLib/test/java/com/archos/mediacenter/filecoreextension/PcloudMediaLibTest.java`
- `Video/src/main/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudAuthParser.java` — 生トークン / `rclone` JSON パースとリージョン解決。
- `Video/src/main/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudAuthServer.java` — `ServerSocket` ベースの軽量ローカル HTTP サーバー（2タブ HTML UI、`POST /auth/code`, `POST /auth/token`）。
- `Video/src/main/java/com/archos/mediacenter/video/leanback/network/pcloud/QrCodeGenerator.java` — ZXing Core による QR `Bitmap` 生成。
- `Video/src/main/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudAuthActivity.java` — Leanback TV 認証画面。
- `Video/src/main/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudListingActivity.java` — `ListingActivity` 実装。
- `Video/src/main/java/com/archos/mediacenter/video/leanback/adapter/object/PcloudBrowse.java` — ネットワーク画面の「pCloud を接続」カードオブジェクト。
- `Video/src/main/res/layout/pcloud_auth_activity.xml` — TV 認証画面レイアウト。
- `Video/src/test/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudAuthParserTest.java`
- `Video/src/test/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudAuthServerTest.java`

### 変更
- `FileCoreLibrary/build.gradle` — `test` sourceSet と単体テスト依存追加。
- `FileCoreLibrary/src/com/archos/filecorelibrary/FileEditorFactory.java` — `"pcloud"` 分岐追加。
- `FileCoreLibrary/src/com/archos/filecorelibrary/ListingEngineFactory.java` — `"pcloud"` 分岐追加。
- `FileCoreLibrary/src/com/archos/filecorelibrary/RawListerFactory.java` — `"pcloud"` 分岐追加。
- `FileCoreLibrary/src/com/archos/filecorelibrary/MetaFile2Factory.java` — `"pcloud"` 分岐・デフォルトポート 443 追加。
- `FileCoreLibrary/src/com/archos/filecorelibrary/FileUtils.java` — `isSlowRemote()`, `isNetworkShare()` に `"pcloud"` 追加。
- `MediaLib/src/com/archos/mediacenter/filecoreextension/UriUtils.java` — `networkSharesTypes` に `"pcloud"` 追加。
- `MediaLib/src/com/archos/medialib/SmbProxy.java` — `needToStream()` に `"pcloud"` 追加。
- `MediaLib/src/com/archos/mediaprovider/video/NetworkScannerServiceVideo.java` — `getStorageId()` に `"pcloud://"` 追加。
- `MediaLib/src/com/archos/mediaprovider/video/RemoteStateService.java` — `SELECTION_DISTANT_REMOTE`, `handleDb()` に `"pcloud"` 追加。
- `Video/build.gradle` — `com.google.zxing:core:3.5.3` 追加、`local.properties` の `pcloud.client_id`/`pcloud.client_secret` を `BuildConfig` 注入、単体テスト依存追加。
- `Video/AndroidManifest.xml` — `<data android:scheme="pcloud" />`、Activity 登録。
- `Video/src/main/java/com/archos/mediacenter/video/CustomApplication.java` — `PcloudUtils.getInstance(mContext)` 初期化。
- `Video/src/main/java/com/archos/mediacenter/video/leanback/filebrowsing/ListingActivity.java` — `"pcloud"` → `PcloudListingActivity.class`。
- `Video/src/main/java/com/archos/mediacenter/video/leanback/network/NetworkRootFragment.java` — `PcloudBrowse` カード表示・クリックで `PcloudAuthActivity` 起動。
- `Video/src/main/java/com/archos/mediacenter/video/utils/VideoUtils.java` — `"pcloud"` 用ショートカット/フォルダアイコン分岐追加。
- `Video/src/main/res/values/strings.xml`, `Video/src/main/res/values-ja/strings.xml` — 英語・日本語リソース追加。
- `Video/src/noamazon/assets/logback.xml` — `com.archos.filecorelibrary.pcloud` ロガー設定。

---

### Task 1: FileCoreLibrary pCloud API クライアント基盤と認証・パス変換

**Files:**
- Modify: `FileCoreLibrary/build.gradle`
- Create: `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudUtils.java`
- Create: `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudApiClient.java`
- Test: `FileCoreLibrary/test/com/archos/filecorelibrary/pcloud/PcloudApiClientTest.java`

**Interfaces:**
- Consumes: `com.archos.filecorelibrary.AuthenticationException`, `com.archos.filecorelibrary.samba.NetworkCredentialsDatabase`, `okhttp3.OkHttpClient`.
- Produces:
  - `PcloudUtils`:
    - `public static final String HOST_US = "api.pcloud.com";`
    - `public static final String HOST_EU = "eapi.pcloud.com";`
    - `public static PcloudUtils getInstance(Context context)`
    - `public static PcloudUtils peekInstance()`
    - `public Context getContext()`
    - `public String getAccessToken(Uri uri) throws AuthenticationException`
    - `public OkHttpClient getApiClient()`
    - `public OkHttpClient getDownloadClient()`
    - `public String getCachedFileLink(String cacheKey)` / `public void putCachedFileLink(String cacheKey, String url)` / `public void invalidateFileLink(String cacheKey)`
    - `public static String extractApiPath(Uri uri)` — `pcloud://api.pcloud.com/Movies/日本語%20#1.mkv` → `/Movies/日本語 #1.mkv`（空なら `/`、ルート以外の末尾 `/` は除去）
    - `public static Uri buildChildUri(Uri parentUri, String childName)` — `Uri.Builder.appendPath(childName)` で安全に構築
    - `public static Uri buildRootUri(String host)` — `pcloud://<host>/`
  - `PcloudApiClient`:
    - `public static class PcloudEntry { public final String name; public final boolean isFolder; public final long size; public final long modifiedMs; public final long id; }`
    - `public static class UserInfo { public final String email; public final String host; }`
    - `public static class TokenResult { public final String accessToken; public final String host; }`
    - `public PcloudApiClient(OkHttpClient httpClient)` / `public PcloudApiClient(OkHttpClient httpClient, String baseUrlOverride)`（テスト用）
    - `public UserInfo fetchUserInfo(String host, String accessToken) throws IOException, AuthenticationException`
    - `public UserInfo fetchUserInfoAutoRegion(String preferredHost, String accessToken) throws IOException, AuthenticationException`（`preferredHost` が null/空なら US → EU 順に試行）
    - `public TokenResult exchangeCodeForToken(String preferredHost, String clientId, String clientSecret, String code) throws IOException, AuthenticationException`（US → 失敗時 EU 自動フォールバック、`locationid == 2` なら `HOST_EU`）
    - `public PcloudEntry stat(String host, String accessToken, String apiPath) throws IOException, AuthenticationException`
    - `public List<PcloudEntry> listFolder(String host, String accessToken, String apiPath) throws IOException, AuthenticationException`
    - `public String getFileLink(String host, String accessToken, String apiPath) throws IOException, AuthenticationException`
    - `public boolean createFolder(String host, String accessToken, String apiPath) throws IOException, AuthenticationException`
    - `public void delete(String host, String accessToken, String apiPath, boolean isFolder) throws IOException, AuthenticationException`
    - `public boolean renameOrMove(String host, String accessToken, String fromPath, String toPath, boolean isFolder) throws IOException, AuthenticationException`
    - `public void uploadFile(String host, String accessToken, String parentPath, String fileName, File tempFile) throws IOException, AuthenticationException`

- [ ] **Step 1: `FileCoreLibrary/build.gradle` に JVM 単体テスト設定を追加**
  - `sourceSets { test { java.srcDirs = ['test'] } }`、`testOptions { unitTests.returnDefaultValues = true }`、および `testImplementation` に `junit:junit:4.13.2`, `org.json:json:20240303`, `com.squareup.okhttp3:mockwebserver:5.3.2`, `org.robolectric:robolectric:4.16.1` を追加する。

- [ ] **Step 2: 失敗するテスト `PcloudApiClientTest.java` を作成**
  - 以下のテストケースを記述する（`@RunWith(RobolectricTestRunner.class)` で `android.net.Uri` を実動作させ、`MockWebServer` で実 HTTP 通信を検証）：
    - `testExtractApiPathAndBuildChildUri_specialChars()`: 日本語・スペース・`#`・`%`・`?` を含むファイル名で `buildChildUri` → `extractApiPath` が往復一致すること。
    - `testListFolder_parsesEntriesAndSendsBearerHeader()`: `listfolder` の JSON からフォルダ・ファイル・サイズ・更新日時（RFC1123 パース）が正しく取得され、`Authorization: Bearer ...` が送られること。
    - `testAuthError1000And2000_throwsAuthenticationException()`: `{"result":1000}` および `{"result":2000}` で `AuthenticationException` がスローされること。
    - `testGetFileLink_constructsHttpsUrl()`: `hosts[0]` + `path` から `https://...` が返ること。
    - `testUploadFile_sendsMultipartWithParentAndFilename()`: `uploadfile` が `path`・`filename`・`nopartial=1` 付き multipart POST で送信され、`result != 0` 時に `IOException` になること。
    - `testAutoRegionFallback_usFailsThenEuSucceeds()`: US 側が `result: 2000` を返し EU 側が成功 JSON を返すとき、EU ホストで成功すること。
    - `testFileLinkCache_respectsTtlAndInvalidation()`: キャッシュ登録・取得・`invalidateFileLink` が動作すること。

- [ ] **Step 3: テストが失敗することを確認**
  - Run: `.\gradlew.bat :FileCoreLibrary:testDebugUnitTest --tests "com.archos.filecorelibrary.pcloud.PcloudApiClientTest"`
  - Expected: FAIL（`PcloudApiClient` / `PcloudUtils` 未定義によるコンパイルエラー）

- [ ] **Step 4: `PcloudUtils.java` と `PcloudApiClient.java` を実装**
  - `PcloudUtils`: シングルトン、`LinkedHashMap` ベースの 64 件 LRU + 10 分 TTL キャッシュ、`HttpUrl.Builder` を使った安全な URL 組み立て（`access_token` はヘッダーに載せ、クエリパラメータに出さず、ログ出力禁止）。
  - `PcloudApiClient`: 各 REST API 呼び出し、`result` フィールド検証（`1000`/`2000` → `AuthenticationException`、`2005`/`2009` → `FileNotFoundException`、その他非 0 → `IOException`）、`modified` 日時の `ZonedDateTime` / `SimpleDateFormat`（RFC 1123, `Locale.US`）パース。

- [ ] **Step 5: テストがパスすることを確認**
  - Run: `.\gradlew.bat :FileCoreLibrary:testDebugUnitTest --tests "com.archos.filecorelibrary.pcloud.PcloudApiClientTest"`
  - Expected: PASS

- [ ] **Step 6: Commit**
  - `FileCoreLibrary` サブモジュール内で `git add build.gradle src/com/archos/filecorelibrary/pcloud test/com/archos/filecorelibrary/pcloud && git commit -m "feat(pcloud): add PcloudApiClient and PcloudUtils with unit tests"`

---

### Task 2: FileCoreLibrary ファイルシステム実装と Factory 登録

**Files:**
- Create: `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudFile2.java`
- Create: `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudRawLister.java`
- Create: `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudListingEngine.java`
- Create: `FileCoreLibrary/src/com/archos/filecorelibrary/pcloud/PcloudFileEditor.java`
- Modify: `FileCoreLibrary/src/com/archos/filecorelibrary/FileEditorFactory.java`
- Modify: `FileCoreLibrary/src/com/archos/filecorelibrary/ListingEngineFactory.java`
- Modify: `FileCoreLibrary/src/com/archos/filecorelibrary/RawListerFactory.java`
- Modify: `FileCoreLibrary/src/com/archos/filecorelibrary/MetaFile2Factory.java`
- Modify: `FileCoreLibrary/src/com/archos/filecorelibrary/FileUtils.java`
- Test: `FileCoreLibrary/test/com/archos/filecorelibrary/pcloud/PcloudFileEditorTest.java`

**Interfaces:**
- Consumes: Task 1 の `PcloudUtils`, `PcloudApiClient`。
- Produces:
  - `PcloudFile2`: `public PcloudFile2(PcloudApiClient.PcloudEntry entry, Uri uri)`, `public static MetaFile2 fromUri(Uri uri) throws Exception`
  - `PcloudRawLister`: `public PcloudRawLister(Uri uri)`
  - `PcloudListingEngine`: `public PcloudListingEngine(Context context, Uri uri)`
  - `PcloudFileEditor`: `public PcloudFileEditor(Uri uri)`（`getInputStream()`, `getInputStream(long from)`, `getOutputStream()`, `length()`, `exists()`, `mkdir()`, `delete()`, `move(Uri)`, `rename(String)`）
  - 各 Factory が `"pcloud"` スキームに対応、`MetaFile2Factory.defaultPortForProtocol("pcloud") == 443`、`FileUtils.isNetworkShare()` / `isSlowRemote()` が `"pcloud"` で `true`。

- [ ] **Step 1: 失敗するテスト `PcloudFileEditorTest.java` を作成**
  - 以下のテストケースを記述：
    - `testGetInputStreamWithRange_sendsRangeHeaderAndSetsLength()`: `getInputStream(1024)` が `Range: bytes=1024-` を送り、`Content-Length` から `length()` を返すこと。
    - `testGetInputStream_expiredLink403_invalidatesCacheAndRetriesOnce()`: 1回目のダウンロードが HTTP 403/410 のときキャッシュを破棄して `getfilelink` を再取得し、2回目で成功すること。
    - `testGetOutputStream_uploadsViaTempFileAndDeletesTempOnSuccess()`: `write()` → `close()` で `uploadfile` が呼ばれ、完了後に一時ファイルが削除されていること。
    - `testGetOutputStream_deletesTempFileOnUploadFailure()`: `uploadfile` が `result != 0` で `IOException` を投げた場合でも、一時ファイルが確実に削除されていること。
    - `testFactoriesAndFileUtils_supportPcloudScheme()`: `FileEditorFactory`, `ListingEngineFactory`, `RawListerFactory`, `MetaFile2Factory.defaultPortForProtocol("pcloud")`, `FileUtils.isNetworkShare`, `FileUtils.isSlowRemote` がすべて `"pcloud"` を正しく扱うこと。

- [ ] **Step 2: テストが失敗することを確認**
  - Run: `.\gradlew.bat :FileCoreLibrary:testDebugUnitTest --tests "com.archos.filecorelibrary.pcloud.PcloudFileEditorTest"`
  - Expected: FAIL（クラス未定義・Factory 未対応）

- [ ] **Step 3: `PcloudFile2`, `PcloudRawLister`, `PcloudListingEngine`, `PcloudFileEditor` を実装し、Factory と `FileUtils` に登録**
  - `PcloudFileEditor.getInputStream(long from)`: キャッシュまたは `getFileLink` でダウンロード URL を取得 → `downloadClient` で `GET`（`from >= 0` なら `Range: bytes=<from>-`）→ 401/403/410 ならキャッシュ破棄して 1 回だけリトライ → `Content-Length` を `mLength` へ保存して `resp.body().byteStream()` を返す。
  - `PcloudFileEditor.getOutputStream()`: `Context.getCacheDir()`（取得不能時は `java.io.tmpdir`）に `File.createTempFile("pcloud_upload_", ".tmp", cacheDir)` を作成し、ラッパー `OutputStream` を返す。`close()` 内でストリームを閉じた後、`try { apiClient.uploadFile(...); } finally { tempFile.delete(); }` を実行する。
  - `PcloudListingEngine`: `WebdavListingEngine` 準拠に加え、`AuthenticationException` キャッチ時に `mListener.onListingFatalError(e, ErrorEnum.ERROR_AUTHENTICATION)` を通知する。
  - `FileEditorFactory`, `ListingEngineFactory`, `RawListerFactory`, `MetaFile2Factory`, `FileUtils` に `"pcloud"` 分岐を追加。

- [ ] **Step 4: テストがパスすることを確認**
  - Run: `.\gradlew.bat :FileCoreLibrary:testDebugUnitTest`
  - Expected: PASS（Task 1 + Task 2 全テスト合格）

- [ ] **Step 5: Commit**
  - `FileCoreLibrary` サブモジュール内でコミット: `"feat(pcloud): implement FileCoreLibrary filesystem classes and register factories"`

---

### Task 3: MediaLib 統合（スキーム・プロキシ・スキャナー・リモート状態）

**Files:**
- Modify: `MediaLib/src/com/archos/mediacenter/filecoreextension/UriUtils.java`
- Modify: `MediaLib/src/com/archos/medialib/SmbProxy.java`
- Modify: `MediaLib/src/com/archos/mediaprovider/video/NetworkScannerServiceVideo.java`
- Modify: `MediaLib/src/com/archos/mediaprovider/video/RemoteStateService.java`
- Test: `MediaLib/test/java/com/archos/mediacenter/filecoreextension/PcloudMediaLibTest.java`

**Interfaces:**
- Consumes: Task 2 の `FileCoreLibrary`（`"pcloud"` スキーム対応済み）。
- Produces:
  - `UriUtils.isImplementedByFileCore(pcloudUri) == true`
  - `UriUtils.isIndexable(Uri.parse("pcloud://api.pcloud.com/Movies")) == true`
  - `UriUtils.isCompatibleWithRemoteDB(pcloudUri) == true`
  - `SmbProxy.needToStream("pcloud") == true`
  - `NetworkScannerServiceVideo`: `"pcloud://"` に対する非ゼロの storage ID（インデックス 10）。
  - `RemoteStateService`: `pcloud://%` を遠隔リモートとして扱い、接続ありならアクティブ維持。

- [ ] **Step 1: 失敗するテスト `PcloudMediaLibTest.java` を作成**
  - `@RunWith(RobolectricTestRunner.class)` で `UriUtils`（`isImplementedByFileCore`, `isIndexable`, `isCompatibleWithRemoteDB`, `isValidStringUri` for US/EU）および `SmbProxy.needToStream("pcloud")` を検証する。

- [ ] **Step 2: テストが失敗することを確認**
  - Run: `.\gradlew.bat :MediaLib:testDebugUnitTest --tests "com.archos.mediacenter.filecoreextension.PcloudMediaLibTest"`
  - Expected: FAIL（`"pcloud"` 未登録のため `false`）

- [ ] **Step 3: `UriUtils`, `SmbProxy`, `NetworkScannerServiceVideo`, `RemoteStateService` を更新**
  - 注意: `UriUtils.networkSharesTypes` の末尾（`"upnp"` の後ろ）に `"pcloud"` を追加することで、既存の `NetworkServerCredentialsDialog` の Spinner position（0=ftp〜7=webdavs）のインデックスを一切ずらさないこと。
  - `SmbProxy.needToStream()` に `"pcloud".equalsIgnoreCase(scheme)` を追加。
  - `NetworkScannerServiceVideo.getStorageId()` に `else if (path.startsWith("pcloud://")) return getStorageId(10 + ArchosMediaCommon.LIGHT_INDEX_STORAGE_ID_OFFSET);` を追加。
  - `RemoteStateService` の `SELECTION_DISTANT_REMOTE` に `MediaColumns.DATA + " LIKE 'pcloud://%'"`、`handleDb()` の `if(...)` に `server.startsWith("pcloud")` を追加。

- [ ] **Step 4: テストがパスすることを確認**
  - Run: `.\gradlew.bat :MediaLib:testDebugUnitTest --tests "com.archos.mediacenter.filecoreextension.PcloudMediaLibTest"`
  - Expected: PASS

- [ ] **Step 5: Commit**
  - `MediaLib` サブモジュール内でコミット: `"feat(pcloud): register pcloud scheme in UriUtils, SmbProxy, and scanners"`

---

### Task 4: Video ローカル認証 HTTP サーバーとトークン/JSON パーサー

**Files:**
- Modify: `Video/build.gradle`（単体テスト依存 `junit`, `org.json`, `robolectric` 追加）
- Create: `Video/src/main/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudAuthParser.java`
- Create: `Video/src/main/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudAuthServer.java`
- Test: `Video/src/test/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudAuthParserTest.java`
- Test: `Video/src/test/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudAuthServerTest.java`

**Interfaces:**
- Consumes: Task 1 の `PcloudApiClient`, `PcloudUtils`。
- Produces:
  - `PcloudAuthParser`:
    - `public static class ParsedToken { public final String accessToken; public final String preferredHost; }`（`preferredHost` は null 可）
    - `public static ParsedToken parseTokenInput(String rawInput, String regionSelection) throws IllegalArgumentException`（`regionSelection` は `"auto"`, `"us"`, `"eu"`。`rawInput` が `{...}` なら JSON として `access_token` と任意の `hostname` を抽出、そうでなければトリムして生トークンとして扱う）
    - `public static String resolveRegionHost(String regionSelection)`（`"us"` → `HOST_US`, `"eu"` → `HOST_EU`, その他 → `null`）
  - `PcloudAuthServer`:
    - `public interface Listener { void onAuthSuccess(String accessToken, String host, String email); void onAuthError(String message); }`
    - `public interface AuthHandler { PcloudApiClient.UserInfo verifyToken(String accessToken, String preferredHost) throws IOException, AuthenticationException; PcloudApiClient.UserInfo exchangeCode(String clientId, String clientSecret, String code, String preferredHost) throws IOException, AuthenticationException; }`
    - `public PcloudAuthServer(String defaultClientId, String defaultClientSecret, AuthHandler handler, Listener listener)`
    - `public int start() throws IOException`（ポート `8765` を試し使用中なら `0` で起動、実バインドポートを返す）
    - `public void stop()`（`ServerSocket` とワーカースレッドを確実に停止）

- [ ] **Step 1: 失敗するテスト `PcloudAuthParserTest.java` と `PcloudAuthServerTest.java` を作成**
  - `PcloudAuthParserTest`:
    - `testRawTokenWithWhitespace_trimmed()`
    - `testRcloneJsonWithHostname_extractsTokenAndEuHost()`
    - `testRcloneJsonWithoutHostname_usesRegionSelection()`
    - `testInvalidJsonOrEmpty_throwsIllegalArgumentException()`
  - `PcloudAuthServerTest`:
    - `testGetRoot_returnsHtmlWithBothTabs()`: `GET /` が 200 OK で OAuth コード認証タブとトークン直接入力タブの HTML を返すこと。
    - `testPostToken_rcloneJson_invokesListenerAndReturns200()`: `POST /auth/token` に rclone JSON を送り、モック `AuthHandler` 経由で `Listener.onAuthSuccess` が呼ばれること。
    - `testPostToken_invalidInput_returns400()`
    - `testPostCode_validCode_invokesListenerAndReturns200()`: `POST /auth/code` で `exchangeCode` が呼ばれ成功すること。
    - `testStop_closesSocketCleanly()`

- [ ] **Step 2: テストが失敗することを確認**
  - Run: `.\gradlew.bat :Video:testNoamazonDebugUnitTest --tests "com.archos.mediacenter.video.leanback.network.pcloud.*"`
  - Expected: FAIL（クラス未定義）

- [ ] **Step 3: `PcloudAuthParser.java` と `PcloudAuthServer.java` を実装**
  - `PcloudAuthServer`:
    - `ServerSocket` でリクエストを受け、`Content-Length` を読み取り `application/x-www-form-urlencoded` を UTF-8 デコードする。
    - `GET /` はスマホ向けレスポンシブ HTML（日本語・英語どちらでも直感的に使えるシンプルなフォーム）。タブ1は `client_id`/`client_secret`（`defaultClientId`/`defaultClientSecret` があればプリセットし、`client_secret` の生値は HTML に埋め込まず「設定済み」フラグで扱う）＋「pCloud でログイン」リンク（`https://my.pcloud.com/oauth2/authorize?client_id=...&response_type=code`）＋ `code` 入力欄＋リージョン選択（自動/US/EU）。タブ2は `access_token` または `rclone authorize "pcloud"` JSON 貼り付け欄＋リージョン選択（自動/US/EU）。
    - ログにはトークン・シークレット・認証コードを絶対に出力しない。

- [ ] **Step 4: テストがパスすることを確認**
  - Run: `.\gradlew.bat :Video:testNoamazonDebugUnitTest --tests "com.archos.mediacenter.video.leanback.network.pcloud.*"`
  - Expected: PASS

- [ ] **Step 5: Commit**
  - `Video` サブモジュール内でコミット: `"feat(pcloud): add PcloudAuthParser and local PcloudAuthServer with tests"`

---

### Task 5: Video Android TV Leanback UI とアプリ全体統合

**Files:**
- Modify: `Video/build.gradle`
- Create: `Video/src/main/java/com/archos/mediacenter/video/leanback/network/pcloud/QrCodeGenerator.java`
- Create: `Video/src/main/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudAuthActivity.java`
- Create: `Video/src/main/java/com/archos/mediacenter/video/leanback/network/pcloud/PcloudListingActivity.java`
- Create: `Video/src/main/java/com/archos/mediacenter/video/leanback/adapter/object/PcloudBrowse.java`
- Create: `Video/src/main/res/layout/pcloud_auth_activity.xml`
- Modify: `Video/AndroidManifest.xml`
- Modify: `Video/src/main/java/com/archos/mediacenter/video/CustomApplication.java`
- Modify: `Video/src/main/java/com/archos/mediacenter/video/leanback/filebrowsing/ListingActivity.java`
- Modify: `Video/src/main/java/com/archos/mediacenter/video/leanback/network/NetworkRootFragment.java`
- Modify: `Video/src/main/java/com/archos/mediacenter/video/utils/VideoUtils.java`
- Modify: `Video/src/main/res/values/strings.xml`
- Modify: `Video/src/main/res/values-ja/strings.xml`
- Modify: `Video/src/noamazon/assets/logback.xml`

**Interfaces:**
- Consumes: Task 1–4 の `PcloudUtils`, `PcloudApiClient`, `PcloudAuthServer`, `PcloudAuthParser`。
- Produces: Android TV のネットワーク画面から pCloud を QR 認証で接続し、ショートカット登録および `PcloudListingActivity` でフォルダ参照・再生・インデックス登録ができる完全なアプリ統合。

- [ ] **Step 1: `Video/build.gradle` に ZXing Core と `local.properties` 読み込みを追加**
  - `implementation 'com.google.zxing:core:3.5.3'` を追加。
  - ルートまたは `Video` の `local.properties` から `pcloud.client_id` / `pcloud.client_secret` を読み取り（未定義時は `""`）、`buildConfigField "String", "PCLOUD_CLIENT_ID", ...` と `PCLOUD_CLIENT_SECRET` を追加する。

- [ ] **Step 2: `QrCodeGenerator.java`, `PcloudListingActivity.java`, `PcloudBrowse.java` を実装**
  - `QrCodeGenerator.generate(String content, int sizePx)`: ZXing `QRCodeWriter` で白背景・黒モジュールの `Bitmap` を生成。
  - `PcloudListingActivity`: `WebdavListingActivity` と同等に `NetworkListingFragment` を返す。
  - `PcloudBrowse extends Shortcut`: `NetworkBrowse` と同型、カードタイトル「pCloud (OAuth)」。

- [ ] **Step 3: `pcloud_auth_activity.xml` と `PcloudAuthActivity.java` を実装**
  - Leanback TV 向け横並びレイアウト（左: QR コード `ImageView`、右: タイトル・URL テキスト・手順説明・ステータス表示・キャンセルボタン）。
  - `onStart()` / `onCreate()` で LAN IPv4 アドレスを取得し `PcloudAuthServer` を起動して QR コード表示（IP 未取得時は Wi-Fi/有線接続を促す表示）。
  - `onAuthSuccess(accessToken, host, email)` を受けたら:
    1. `String rootUri = "pcloud://" + host + "/";`
    2. `NetworkCredentialsDatabase.getInstance().saveCredential(new Credential(email, accessToken, rootUri, "", false));`
    3. `ShortcutDb.STATIC.insertShortcut(this, Uri.parse(rootUri), "pCloud (" + email + ")");`（未登録時）
    4. `PcloudListingActivity` へ `EXTRA_ROOT_URI = Uri.parse(rootUri)`, `EXTRA_ROOT_NAME = "pCloud"` で遷移し、`setResult(NetworkRootFragment.RESULT_CODE_SHORTCUTS_MODIFIED)` して終了。
  - `onDestroy()` で `PcloudAuthServer.stop()` を確実に呼ぶ。

- [ ] **Step 4: `NetworkRootFragment`, `ListingActivity`, `CustomApplication`, `VideoUtils`, `AndroidManifest.xml`, 文字列リソース, `logback.xml` を統合**
  - `NetworkRootFragment.NetworkShortcutsLoaderTask`: `NetworkBrowse` の次に `PcloudBrowse` を追加。クリック時に `PcloudAuthActivity` を `startActivityForResult(..., REQUEST_CODE_DETAILS)` で起動。
  - `ListingActivity.getActivityForUri()`: `"pcloud"` → `PcloudListingActivity.class`。
  - `CustomApplication.onCreate()`: バックグラウンド初期化スレッドで `PcloudUtils.getInstance(mContext)` を呼ぶ。
  - `VideoUtils`: `"pcloud"` のアイコン分岐を追加。
  - `AndroidManifest.xml`: `webdav` / `webdavs` と同じ全 intent-filter に `<data android:scheme="pcloud" />` を追加し、`PcloudListingActivity` と `PcloudAuthActivity` を登録。
  - `strings.xml`（英語）と `values-ja/strings.xml`（日本語）に UI 文言を追加。

- [ ] **Step 5: ビルドとユニットテストを実行して確認**
  - Run: `.\gradlew.bat :Video:compileNoamazonDebugJavaWithJavac :Video:testNoamazonDebugUnitTest`
  - Expected: PASS

- [ ] **Step 6: Commit**
  - `Video` サブモジュール内でコミット: `"feat(pcloud): add Android TV OAuth QR flow, PcloudListingActivity, and manifest integration"`

---

### Task 6: 全体ビルド・Lint (`NewApi`)・全ユニットテスト検証

**Files:**
- Verify only（修正が必要な場合のみ該当ファイル）

- [ ] **Step 1: 全モジュールのユニットテストと APK ビルド＋Lint を実行**
  - Run: `.\gradlew.bat :FileCoreLibrary:testDebugUnitTest :MediaLib:testDebugUnitTest :Video:testNoamazonDebugUnitTest :Video:assembleNoamazonDebug`
  - Expected: すべてのテストが PASS し、`lintNoamazonDebug`（`NewApi` fatal チェック含む）および `assembleNoamazonDebug` が BUILD SUCCESSFUL で完了すること。

- [ ] **Step 2: 機密情報リークチェック**
  - Run: `git -C FileCoreLibrary diff; git -C MediaLib diff; git -C Video diff`（`access_token` / `client_secret` のログ出力やハードコードが無いことを確認）。
