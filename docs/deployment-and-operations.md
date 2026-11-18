# 配置・運用・バックアップ設計

## 1. ローカル環境

- Docker Compose: `php:8.2-apache`相当のWeb/API、MySQL 8。
- ホストNode.js: Vite開発サーバ。`/api`はApacheコンテナへproxyする。
- ComposerはPHPコンテナ内で実行し、`composer.lock`をGit管理する。
- DB、storage、backupは名前付きvolumeまたはbind mountへ保存する。

## 2. 本番ディレクトリ

```text
/home/{account}/
├── secureshare/
│   ├── app/                 # backend本体・vendor
│   ├── config/              # .env（600）
│   ├── var/
│   │   ├── storage/         # 保存ファイル（700）
│   │   ├── trash/           # 削除隔離（700）
│   │   ├── sessions/        # PHP session（700）
│   │   ├── logs/            # 構造化ログ（700）
│   │   ├── maintenance      # 存在時は状態変更を503にする
│   │   └── backups/         # バックアップ（700）
│   └── releases/
└── www/
    ├── index.html           # React build
    ├── assets/
    ├── api/index.php        # backend publicへの入口
    └── .htaccess            # HTTPS・API・SPA rewrite
```

`www`配下に`.env`、vendor、SQL、storage、backupを置かない。PHPが必要とする実パスは環境変数で絶対パス指定する。

## 3. 環境変数

必須:

```text
APP_ENV=production
APP_URL=https://example.jp
APP_TIMEZONE=UTC
APP_ENCRYPTION_KEY=<base64 32 bytes>
APP_HMAC_KEY=<base64 32 bytes>
DB_HOST=<mysql host>
DB_PORT=3306
DB_DATABASE=<database>
DB_USERNAME=<least privilege user>
DB_PASSWORD=<secret>
STORAGE_PATH=/home/{account}/secureshare/var/storage
TRASH_PATH=/home/{account}/secureshare/var/trash
SESSION_PATH=/home/{account}/secureshare/var/sessions
LOG_PATH=/home/{account}/secureshare/var/logs
BACKUP_PATH=/home/{account}/secureshare/var/backups
TRUSTED_PROXIES=
```

起動時に必須値、鍵長、公開領域外のstorage、書込権限を診断し、不正ならHTTP処理を開始しない。

## 4. デプロイ手順

1. ローカル/CIでPHPUnit、フロントエンドtest、build、OpenAPI検証を完了する。
2. フロントエンドをbuildし、リリースディレクトリへアプリと依存を転送する。
3. `.env`を本番で作成し、権限600にする。秘密を転送物へ含めない。
4. メンテナンスファイルを作成し、状態変更APIを503にする。
5. `bin/migrate`を実行し、適用済みversionと結果を確認する。
6. `www`の入口とassetsを新リリースへ原子的に切り替える。
7. health、ログイン、アップロード、共有resolve、downloadのsmoke testを行う。
8. メンテナンスを解除し、旧リリースを直近2世代だけ残す。

DB migrationは前方互換のadditive changeを先に適用する。破壊的migrationが必要な場合は事前backupと復元試験を必須とする。

## 5. バックアップCLI

`bin/backup create`:

1. `var/maintenance`を原子的に作成し、新規状態変更を503にする。
2. 進行中リクエストの終了を最大60秒待つ。
3. `mysqldump --single-transaction --routines=false --triggers=true`でDBを取得する。
4. storageをtar archiveへ保存する。
5. アプリversion、migration version、件数、合計bytes、各成果物SHA-256を`manifest.json`へ記録する。
6. 一時ディレクトリから日付付きbackupディレクトリへrenameする。
7. メンテナンスを解除する。途中失敗時もfinally処理で解除し、不完全backupを`.failed`へ隔離する。

秘密鍵と`.env`はbackupへ含めない。別の安全な保管先へ手動で保管する。backup CLIは既定では世代削除を行わない。

## 6. 復元手順

1. 新しい空DBと空storageを準備する。既存環境へ上書きしない。
2. `bin/backup verify <path>`でmanifestと全checksumを検証する。
3. DB dumpをimportし、migration versionとテーブル件数を照合する。
4. storage archiveを一時領域へ展開し、パスとfile countを検証してから配置する。
5. `files.storage_key`に対応する実体、size、SHA-256を`bin/storage verify`で全件照合する。
6. 別URLでsmoke test後に切り替える。失敗時は元環境を維持する。

四半期ごと、および重要なschema変更前に復元試験を行う。

## 7. 定期保守

- 日次: `bin/logs prune --days=180`、`bin/storage clean-trash --older-than=24h`。
- 週次: backup作成、storage整合性のsample検査。
- 月次: Composer/npmの脆弱性確認、依存更新候補の確認。
- 公開後: 5xx増加、ディスク残量、DB接続失敗、backup失敗をログで確認する。

