# SecureShare 詳細設計

このディレクトリは、要件定義書 v1.0 に基づく Phase 1 の実装仕様を管理する。
要件原本は `.docs/` に保持し、Git 管理対象には含めない。

## 文書一覧

| 文書 | 内容 |
|---|---|
| [architecture.md](architecture.md) | 構成、認証状態、ファイル処理、共有フロー |
| [screens.md](screens.md) | UI-001〜UI-011、画面遷移、レスポンシブ方針 |
| [database.md](database.md) | ER 図、物理テーブル、制約、排他制御 |
| [openapi.yaml](openapi.yaml) | `/api/v1` の OpenAPI 3.1 定義 |
| [security.md](security.md) | 脅威、セキュリティ制御、監査ログ |
| [deployment-and-operations.md](deployment-and-operations.md) | さくら配置、バックアップ、復元、保守 |
| [traceability.md](traceability.md) | 要件 ID と設計・試験の対応 |

## 確定値

- ファイル上限: 20 MiB (`20 * 1024 * 1024` bytes)
- ユーザー容量上限: 1 GiB (`1024 * 1024 * 1024` bytes)
- 共有期限: 1 / 3 / 7 / 14 / 30 日、既定 7 日
- ダウンロード上限: 無制限 / 1 / 3 / 5 回
- 管理セッション: 無操作 30 分
- 履歴保持: 180 日
- 日時: DB と API は UTC、画面表示はブラウザのローカルタイム

