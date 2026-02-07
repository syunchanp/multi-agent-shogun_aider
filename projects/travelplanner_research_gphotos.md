# Google Photos API 制約調査報告

> **調査日**: 2026-02-07
> **調査者**: 足軽2
> **task_id**: subtask_001b

## 概要

Google Photos APIは2025年3月31日に大幅な変更が実施され、従来のLibrary APIのスコープが大幅に制限された。本調査では、現在（2026年2月）時点でのAPI制約と設計への影響をまとめる。

---

## 1. Google Photos API の機能と制限

### 1.1 現在利用可能なAPI

| API名 | 用途 | 主な制限 |
|-------|------|----------|
| **Photos Picker API** | ユーザーがライブラリから写真を選択 | ユーザー操作が必要、セッションベース |
| **Photos Library API** | アプリが作成したコンテンツの管理 | **アプリがアップロードした写真のみ**アクセス可 |

### 1.2 2025年3月31日の変更点（重要）

以下のスコープは**削除済み**であり、使用すると `403 PERMISSION_DENIED` エラーが返る：

| 削除されたスコープ | 影響 |
|-------------------|------|
| `photoslibrary.readonly` | ユーザーの全ライブラリへの読み取り不可 |
| `photoslibrary.sharing` | 共有アルバム機能の使用不可 |
| `photoslibrary` | フルアクセス権限の使用不可 |

**設計への影響**:
- ユーザーの既存写真（旅行写真など）を自動取得することは**不可能**
- ユーザーに写真を選択させる場合は**Picker API**を使用する必要あり

### 1.3 アクセス可能なデータ

| データ種別 | Library API | Picker API |
|-----------|-------------|------------|
| アプリがアップロードした写真 | ✅ | ✅ |
| ユーザーの既存写真 | ❌ | ✅（ユーザー選択時のみ） |
| アルバム（アプリ作成） | ✅ | - |
| アルバム（ユーザー作成） | ❌ | ✅（ユーザー選択時のみ） |
| EXIFメタデータ | ✅（制限あり） | ✅（制限あり） |
| GPS位置情報 | ✅（アプリ作成分のみ） | ✅（選択分のみ） |

---

## 2. API利用の制約

### 2.1 レート制限（クォータ）

#### Photos Picker API
| 制限項目 | 制限値 |
|---------|--------|
| APIリクエスト | 100,000 リクエスト/分/プロジェクト |
| メディアバイトアクセス | 1,000,000 リクエスト/分/プロジェクト |

#### Photos Library API
| 制限項目 | 制限値 |
|---------|--------|
| APIリクエスト | **10,000 リクエスト/日/プロジェクト** |
| メディアバイトアクセス | 75,000 リクエスト/日/プロジェクト |

**エラー処理**: クォータ超過時は `429 Too Many Requests` が返る。指数バックオフでリトライを実装すること。

**設計への影響**:
- Library APIの日次10,000リクエスト制限は厳しい。大量の写真を扱う場合は設計上の考慮が必要
- Picker APIは分単位制限のため、より柔軟

### 2.2 利用規約上の制約

#### 許可される用途
- 個人的な画像・動画の保存、編集、印刷、エクスポート、共有
- Google Photosのコア機能を**強化**するアプリケーション

#### 禁止される用途
| 禁止事項 | 詳細 |
|---------|------|
| 競合製品の開発 | Google Photosの代替となるギャラリーアプリ禁止 |
| 大規模ストレージ利用 | CDN代替やエンタープライズストレージとしての利用禁止 |
| 顔クラスタリング機能 | 顔認識機能の開発禁止 |
| データの販売 | ユーザーデータの第三者への販売禁止 |
| 広告目的の利用 | ユーザーデータを広告に使用禁止 |
| 一括処理 | 明示的許可なしのバルク処理禁止 |

### 2.3 商用利用の可否

**商用利用は条件付きで可能**:
- 承認されたユースケース内であれば可
- ユーザーの明示的同意が必須
- データ保護要件の厳守が必要

**禁止事項**:
- ユーザーデータの販売
- 広告目的でのデータ利用
- 信用判定へのデータ利用
- 音楽付きメモリーの商用利用

---

## 3. 認証・認可

### 3.1 OAuth 2.0の実装要件

| 要件 | 詳細 |
|------|------|
| 認証方式 | OAuth 2.0 **必須** |
| サービスアカウント | **非対応**（ユーザー認証のみ） |
| アクセストークン | 短期間有効、リフレッシュトークン要 |
| OAuth検証 | 公開アプリは**Google審査必須** |

**重要**: サービスアカウントが使えないため、サーバーサイドでの自動処理には制限がある。ユーザーのインタラクティブなログインが必要。

### 3.2 現在有効なスコープ

#### Picker API用
```
https://www.googleapis.com/auth/photospicker.mediaitems.readonly
```
- セッションの作成・取得・削除
- セッション内のメディアアイテム一覧取得

#### Library API用（現在有効なもの）
```
https://www.googleapis.com/auth/photoslibrary.appendonly
```
- アップロード、メディアアイテム作成（書き込み専用）
- アルバム作成

```
https://www.googleapis.com/auth/photoslibrary.readonly.appcreateddata
```
- アプリ作成コンテンツの読み取り

```
https://www.googleapis.com/auth/photoslibrary.edit.appcreateddata
```
- アプリ作成コンテンツの編集

### 3.3 OAuth検証プロセス

1. Google API Consoleでプロジェクト作成
2. Photos APIを有効化
3. OAuth同意画面の設定
4. **Google審査を受ける**（公開アプリの場合）
   - 審査前は「未確認アプリ」警告が表示される
   - 審査には数週間かかる可能性あり

---

## 4. 代替手段

### 4.1 Picker APIへの移行（推奨）

ユーザーの既存写真にアクセスする必要がある場合：

```
旧方式（2025年3月以前）:
Library API + photoslibrary.readonly → ユーザーの全写真にアクセス

新方式（2025年4月以降）:
Picker API → ユーザーが選択した写真のみアクセス
```

**Picker APIの特徴**:
- ユーザーがGoogle提供のUIで写真を選択
- セッションベースのアクセス（一時的）
- より高いクォータ制限

### 4.2 ユーザー手動アップロード

API制約を回避する最も確実な方法：
- ユーザーにファイルを直接アップロードさせる
- 自前のストレージ（S3、GCS等）に保存
- Google Photosへの依存を排除

### 4.3 Google Drive API との併用

| 比較項目 | Google Photos API | Google Drive API |
|---------|-------------------|------------------|
| 既存ファイルへのアクセス | ❌（Picker経由のみ） | ✅ |
| サービスアカウント | ❌ | ✅ |
| EXIF/GPS取得 | 制限あり | ファイルとして取得可能 |
| 写真管理UI | Google Photos UI | なし |

**注意**: Google Driveに保存された写真とGoogle Photosは別管理のため、ユーザーがDriveに写真を保存している場合のみ有効。

### 4.4 Partner Program

大規模アプリケーション向け：
- クォータ制限の緩和が可能
- 審査・契約が必要
- 一般的な個人開発では利用困難

---

## 5. 設計への影響まとめ

### 5.1 TravelPlannerへの影響

| 機能要件 | 実現可否 | 備考 |
|---------|---------|------|
| ユーザーの旅行写真を自動取得 | ❌ | API制限により不可 |
| ユーザーに写真を選択させる | ✅ | Picker API使用 |
| 写真のGPS情報取得 | ✅ | 選択された写真のみ |
| 写真のEXIF取得 | ✅ | 制限あり |
| アプリで撮影した写真の管理 | ✅ | Library API使用 |
| 自動でアルバム作成 | ✅ | アプリ作成アルバムのみ |

### 5.2 推奨アーキテクチャ

```
ユーザー操作
    │
    ▼
Picker API（写真選択UI）
    │
    ▼
選択された写真のメタデータ・URL取得
    │
    ▼
アプリ側でGPS情報を抽出
    │
    ▼
地図上にプロット
```

### 5.3 制約事項チェックリスト

- [ ] OAuth 2.0実装必須
- [ ] サービスアカウント使用不可（ユーザー認証必須）
- [ ] ユーザーの既存写真への自動アクセス不可
- [ ] Library API日次10,000リクエスト制限に注意
- [ ] 公開前にGoogle OAuth審査必要
- [ ] データの販売・広告利用禁止

---

## 参考資料

- [API limits and quotas | Google Photos APIs](https://developers.google.com/photos/overview/api-limits-quotas)
- [Authorization scopes | Google Photos APIs](https://developers.google.com/photos/overview/authorization)
- [Photos API User Data and Developer Policy](https://developers.google.com/photos/support/api-policy)
- [Updates to the Google Photos APIs](https://developers.google.com/photos/support/updates)
- [Get started with the Picker API](https://developers.google.com/photos/picker/guides/get-started-picker)
- [Google Photos API Deprecation - memoryKPR](https://memorykpr.com/blog/google-photos-api-deprecation-what-it-means-for-third-party-apps-and-how-to-prepare/)
