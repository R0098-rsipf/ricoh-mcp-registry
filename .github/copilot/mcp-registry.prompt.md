# MCP Registry 設定ガイド

## 概要

MCP Registry は、Model Context Protocol (MCP) サーバーのディレクトリであり、IDE や Copilot のカタログとして機能します。
Enterprise または Organization 所有者は、MCP Registry URL とアクセス制御ポリシーを構成して、開発者が使用できる MCP サーバーを管理できます。

## MCP Registry の JSON フォーマット

### 基本構造

MCP Registry は、公式スキーマに準拠した以下の構造の JSON 応答を返す必要があります。
参照: https://registry.modelcontextprotocol.io/docs#/operations/list-servers-v0.1

```json
{
  "metadata": {
    "count": 2, // 登録したサーバーの総数
    "nextCursor": "server-name:version" // ページネーション用カーソル（オプション）
  },
  "servers": [
    {
      "server": {
        "$schema": "https://static.modelcontextprotocol.io/schemas/2025-10-17/server.schema.json",
        "name": "example.com/server-name", // 名前空間付きのサーバー名（例: "ai.example/my-server"）
        "version": "1.0.0", // セマンティックバージョニング形式
        "description": "サーバーの説明",
        "repository": {
          "url": "https://github.com/org/repo",
          "source": "github",
          "subfolder": "path/to/server" // オプション: サブフォルダパス
        },
        // 以下のいずれかの配置方法を指定:
        // 1. packages配列: PyPI, npm, OCI等のパッケージレジストリから配布
        "packages": [
          {
            "registryType": "npm", // "npm", "pypi", "oci" など
            "registryBaseUrl": "https://registry.npmjs.org",
            "identifier": "package-name",
            "version": "1.0.0",
            "runtimeHint": "npx", // オプション: 実行時のヒント
            "transport": {
              "type": "stdio" // "stdio" または "sse"
            },
            "environmentVariables": [
              {
                "name": "API_KEY",
                "description": "APIキーの説明",
                "isRequired": true,
                "isSecret": true,
                "default": "default-value", // オプション
                "format": "string" // オプション
              }
            ]
          }
        ],
        // 2. remotes配列: リモートMCPサーバーとして配布
        "remotes": [
          {
            "type": "sse", // "sse" または "streamable-http"
            "url": "https://example.com/mcp",
            "headers": [
              // オプション: 追加HTTPヘッダー
              {
                "name": "Authorization",
                "description": "認証トークン",
                "value": "Bearer ${TOKEN}", // 環境変数参照可能
                "isSecret": true
              }
            ]
          }
        ]
      },
      "_meta": {
        "io.modelcontextprotocol.registry/official": {
          "status": "active", // "active" または "deprecated"
          "publishedAt": "2025-01-01T00:00:00.000000Z", // ISO 8601形式（マイクロ秒まで）
          "updatedAt": "2025-01-01T00:00:00.000000Z", // ISO 8601形式（マイクロ秒まで）
          "isLatest": true // 最新バージョンの場合はtrue
        }
      }
    }
  ]
}
```

### 必須フィールド

#### ルートレベル

- **`metadata`**: レジストリのメタデータ
  - **`count`**: サーバーの総数（数値）
- **`servers`**: サーバーエントリの配列

#### 各サーバーエントリ

- **`server`**: サーバー定義オブジェクト

  - **`$schema`**: スキーマ URL（`https://static.modelcontextprotocol.io/schemas/YYYY-MM-DD/server.schema.json`）
  - **`name`**: 名前空間付きサーバー名（例: `"ai.example/my-server"`）
  - **`version`**: セマンティックバージョン（例: `"1.0.0"`）
  - **`description`**: サーバーの説明文
  - **`repository`**: リポジトリ情報（オプションだが推奨）
    - **`url`**: リポジトリ URL
    - **`source`**: ソース種別（例: `"github"`）
  - **配置方法（以下のいずれか必須）**:
    - **`packages`**: パッケージレジストリからの配布
    - **`remotes`**: リモートサーバーとしての配布

- **`_meta`**: メタデータオブジェクト
  - **`io.modelcontextprotocol.registry/official`**: 公式レジストリメタデータ
    - **`status`**: ステータス（`"active"` または `"deprecated"`）
    - **`publishedAt`**: 公開日時（ISO 8601 形式、マイクロ秒精度）
    - **`updatedAt`**: 更新日時（ISO 8601 形式、マイクロ秒精度）
    - **`isLatest`**: 最新バージョンフラグ（boolean）

### オプションフィールド

- **`metadata.nextCursor`**: ページネーション用の次ページカーソル
- **`server.repository.subfolder`**: モノレポ内のサブフォルダパス
- **`packages[].runtimeHint`**: 実行時のヒント（例: `"npx"`, `"uvx"`）
- **`packages[].environmentVariables`**: 環境変数定義配列
- **`remotes[].headers`**: カスタム HTTP ヘッダー配列

### 配置方法の選択

1. **`packages` を使用する場合**: PyPI、npm、OCI 等のパッケージレジストリから配布

   - `registryType`: `"npm"`, `"pypi"`, `"oci"` など
   - `transport.type`: `"stdio"` または `"sse"`

2. **`remotes` を使用する場合**: HTTP エンドポイント経由でリモートサーバーとして提供
   - `type`: `"sse"` または `"streamable-http"`
   - OAuth 認証等が必要な場合は `headers` でトークンを指定可能

## 公式 API 仕様 - プロパティ詳細

以下は、公式 MCP Registry API (`list-servers-v0.1`) のプロパティ定義です。
参照元: https://registry.modelcontextprotocol.io/docs#/operations/list-servers-v0.1

### レスポンス構造

#### `metadata` (object, required)

ページネーションメタデータ

- **`count`** (integer<int64>, required): 現在のページに含まれるアイテム数
- **`nextCursor`** (string, optional): 次のページを取得するためのページネーションカーソル。次のリクエストの`cursor`クエリパラメータにこの値を使用する

#### `servers` (array[object] or null, required)

サーバーエントリのリスト

各サーバーエントリは以下の構造を持つ:

##### `server` (object, required)

サーバー設定とメタデータ

- **`$schema`** (string<uri>, required): この server.json フォーマットの JSON Schema URI

  - 制約: 1 文字以上
  - 例: `"https://static.modelcontextprotocol.io/schemas/2025-10-17/server.schema.json"`

- **`name`** (string, required): リバース DNS 形式のサーバー名。名前空間とサーバー名を区切る 1 つのスラッシュを含む必要がある

  - 制約: 3-200 文字
  - パターン: `^[a-zA-Z0-9.-]+/[a-zA-Z0-9._-]+$`
  - 例: `"io.github.user/weather"`

- **`version`** (string, required): サーバーのバージョン文字列。セマンティックバージョニングに従うべき

  - 例: `"1.0.2"`

- **`description`** (string, required): サーバー機能の明確で人間が読める説明

  - 制約: 1-100 文字
  - 例: `"MCP server providing weather data and forecasts via OpenWeatherMap API"`

- **`title`** (string, optional): MCP サーバーの人間が読めるタイトルまたは表示名

  - 制約: 1-100 文字
  - 例: `"Weather API"`

- **`websiteUrl`** (string<uri>, optional): サーバーのホームページ、ドキュメント、またはプロジェクトウェブサイトの URL

  - 例: `"https://modelcontextprotocol.io/examples"`

- **`icon`** (string<uri>, optional): サーバーを視覚的に識別するためのアイコン画像 URL

  - 推奨: GitHub organization のアバター画像 URL（例: `"https://avatars.githubusercontent.com/u/124619424?s=200&v=4"`）
  - 形式: PNG, SVG などの画像形式
  - 例: `"https://avatars.githubusercontent.com/u/124619424?s=200&v=4"`

- **`repository`** (object, optional): MCP サーバーソースコードのリポジトリメタデータ

  - **`url`** (string<uri>, required): リポジトリ URL
    - 例: `"https://github.com/modelcontextprotocol/servers"`
  - **`source`** (string, required): ソースの種類
    - 例: `"github"`
  - **`subfolder`** (string, optional): モノレポ内のサブフォルダパス
    - 例: `"src/everything"`
  - **`id`** (string, optional): リポジトリのユニーク ID
    - 例: `"b94b5f7e-c7c6-d760-2c78-a5e9b8a5b8c9"`

- **`icons`** (array[object] or null, optional): クライアントが UI に表示できるサイズ付きアイコンのセット

  - 各アイコンオブジェクト:
    - **`src`** (string<uri>, required): アイコン画像の URL
      - 例: `"https://example.com/icon.png"`
    - **`mimeType`** (string, required): アイコンの MIME タイプ
      - 例: `"image/png"`
    - **`sizes`** (array[string], optional): アイコンサイズのリスト
    - **`theme`** (string, optional): テーマ指定 (`"light"`, `"dark"`, etc.)

- **`packages`** (array[object] or null, optional): パッケージ設定の配列

  - **`registryType`** (string, required): パッケージレジストリの種類
    - 値: `"npm"`, `"pypi"`, `"oci"` など
    - 例: `"npm"`
  - **`registryBaseUrl`** (string<uri>, optional): レジストリのベース URL
    - 例: `"https://registry.npmjs.org"`
  - **`identifier`** (string, required): パッケージ識別子
    - 例: `"@modelcontextprotocol/server-brave-search"`
  - **`version`** (string, required): パッケージバージョン
    - 例: `"1.0.2"`
  - **`runtimeHint`** (string, optional): 実行時のヒント
    - 例: `"npx"`
  - **`fileSha256`** (string, optional): パッケージファイルの SHA256 ハッシュ
    - 例: `"fe333e598595000ae021bd27117db32ec69af6987f507ba7a63c90638ff633ce"`
  - **`transport`** (object, required): トランスポート設定
    - **`type`** (string, required): トランスポートタイプ (`"stdio"` または `"sse"`)
    - **`url`** (string<uri>, optional): トランスポート URL
    - **`headers`** (array[object], optional): カスタム HTTP ヘッダー配列
  - **`environmentVariables`** (array[object], optional): 環境変数定義の配列
  - **`packageArguments`** (array[object], optional): パッケージ引数の配列
  - **`runtimeArguments`** (array[object], optional): ランタイム引数の配列

- **`remotes`** (array[object] or null, optional): リモート設定の配列

  - **`type`** (string, required): リモートタイプ (`"stdio"`, `"sse"`, `"streamable-http"`)
  - **`url`** (string<uri>, required): リモートエンドポイント URL
    - 例: `"https://api.example.com/mcp"`
  - **`headers`** (array[object], optional): カスタム HTTP ヘッダー配列

- **`_meta`** (object, optional): リバース DNS 名前空間を使用したベンダー固有データの拡張メタデータ

##### `_meta` (object, required)

レジストリ管理メタデータ

- **`io.modelcontextprotocol.registry/official`** (object, optional): 公式 MCP レジストリメタデータ
  - **`status`** (string, required): サーバーのライフサイクルステータス
    - 許可値: `"active"`, `"deprecated"`, `"deleted"`
  - **`publishedAt`** (string<date-time>, required): サーバーがレジストリに初めて公開されたタイムスタンプ
    - 形式: ISO 8601 (RFC3339)
    - 例: `"2019-08-24T14:15:22Z"`
  - **`updatedAt`** (string<date-time>, optional): サーバーエントリが最後に更新されたタイムスタンプ
    - 形式: ISO 8601 (RFC3339)
    - 例: `"2019-08-24T14:15:22Z"`
  - **`isLatest`** (boolean, required): これがサーバーの最新バージョンかどうか

### クエリパラメータ

- **`cursor`** (string, optional): ページネーションカーソル

  - 例: `"server-cursor-123"`

- **`limit`** (integer<int64>, optional): 1 ページあたりのアイテム数

  - 制約: 1-100
  - デフォルト: 30
  - 例: 50

- **`search`** (string, optional): 名前でサーバーを検索 (部分文字列マッチ)

  - 例: `"filesystem"`

- **`updated_since`** (string, optional): 指定されたタイムスタンプ以降に更新されたサーバーをフィルタ (RFC3339 datetime)

  - 例: `"2025-08-07T13:15:04.280Z"`

- **`version`** (string, optional): バージョンでフィルタ (最新版は`'latest'`、または`'1.2.3'`のような正確なバージョン)
  - 例: `"latest"`

## このリポジトリでの運用方法

このリポジトリでは MCP Registry の管理を行っています。

### Registry ファイルの管理

- **ファイルパス**: `docs/registry.json`
- **公開方法**: GitHub Pages にて自動公開
- **更新方法**: ユーザーから MCP サーバーの追加・更新・削除の指示があった場合は、`docs/registry.json` を直接編集してください

### 更新時の注意点

1. **JSON フォーマットに従って正確に記述する**

   - 公式スキーマ（`https://static.modelcontextprotocol.io/schemas/YYYY-MM-DD/server.schema.json`）に準拠すること

2. **`metadata.count` を更新する**

   - レジストリ内の全サーバーバージョンの総数に合わせる（最新版のみでなく全バージョンを含む）

3. **サーバー名は名前空間付きで記述する**

   - 形式: `"組織ドメイン/サーバー名"` （例: `"ai.example/my-server"`, `"com.github/awesome-server"`）
   - 一意性を保つため、ドメインベースの命名規則を推奨

4. **必須フィールドを必ず含める**

   - `server.$schema`: スキーマ URL
   - `server.name`: 名前空間付きサーバー名
   - `server.version`: セマンティックバージョン
   - `server.description`: サーバーの説明
   - `server.packages` または `server.remotes`: 配置方法（いずれか必須）
   - `_meta.io.modelcontextprotocol.registry/official`: 公式メタデータ

5. **日時は ISO 8601 形式（マイクロ秒精度）で記述する**

   - 形式: `"YYYY-MM-DDTHH:mm:ss.ffffffZ"`
   - 例: `"2025-11-27T10:30:45.123456Z"`
   - `publishedAt`: 新規追加時の日付
   - `updatedAt`: 更新時の日付

6. **リポジトリ情報を可能な限り含める**

   - `repository.url`: GitHub リポジトリ URL
   - `repository.source`: `"github"` など
   - `repository.subfolder`: モノレポの場合はサブフォルダパス

7. **アイコンを設定する（推奨）**

   - `server.icon`: サーバーのアイコン画像 URL を指定
   - 推奨: GitHub organization のアバター画像 URL を使用
     - 形式: `"https://avatars.githubusercontent.com/u/{org_id}?s=200&v=4"`
   - Organization ID の確認方法:
     - GitHub organization ページの URL から取得
     - または GitHub API を使用: `https://api.github.com/orgs/{org_name}`
   - 例: Model Context Protocol の場合
     - `"icon": "https://avatars.githubusercontent.com/u/124619424?s=200&v=4"`

8. **配置方法（packages または remotes）を適切に選択する**

   - **packages**: PyPI、npm、OCI 等のパッケージレジストリから配布する場合
     - `registryType`, `identifier`, `version`, `transport` を指定
     - 環境変数が必要な場合は `environmentVariables` を定義
   - **remotes**: HTTP エンドポイント経由でサービスとして提供する場合
     - `type` (`"sse"` または `"streamable-http"`) と `url` を指定
     - 認証が必要な場合は `headers` を定義

9. **バージョン管理の注意点**

   - 同じサーバーの複数バージョンを登録可能
   - 最新バージョンのみ `_meta.io.modelcontextprotocol.registry/official.isLatest` を `true` に設定
   - 古いバージョンは `false` に設定

10. **`docs/mcp-registry.md` の更新**

- `docs/registry.json` を更新した後、必ず `docs/mcp-registry.md` のサーバー一覧テーブルも更新する
- テーブルフォーマット: `| 名前 | 説明 |`
- 名前カラムには `server.repository.url` へのリンクを設定: `[server.name](repository.url)`
- 説明カラムには `server.description` の内容をそのまま記載
- 最新バージョンのみをテーブルに含める（`isLatest: true` のもの）

## セキュリティサーベイの実行

MCP Registry の更新作業が完了した後、追加された MCP サーバーのセキュリティ評価を実施してください。

**実行手順**:

1. `.github/prompts/mcp-security-servey.prompt.md` に記載された指示に従う
2. 追加された MCP サーバー名を入力として使用
3. セキュリティサーベイを実施し、PR Template を埋めた markdown file を repository root に生成し、保存する

詳細な指示については、`.github/prompts/mcp-security-servey.prompt.md` を参照してください。
