# 要件トレーサビリティ

## 機能要件

| 要件 | 設計箇所 | 主な受け入れ試験 |
|---|---|---|
| FR-001〜006 | architecture §3、security §2〜3、OpenAPI `/auth/*` | パスワード→MFA状態遷移、TOTP再利用拒否、復旧コード一回限り、logout、試行制限 |
| FR-007〜013 | architecture §5、database `files`、OpenAPI `/files*` | 対応形式、20 MiB拒否、1 GiB同時超過防止、検索、削除、使用量 |
| FR-014〜020 | architecture §4/6、database `shares` | 32-byte token、期限選択、パスワード、1/3/5回、停止、再共有 |
| FR-021〜022 | architecture §6、OpenAPI public preview/download | 対応形式inline、Office/ZIP拒否、認可後attachment、上限到達拒否 |
| FR-023〜026 | database `access_logs`、security §6、OpenAPI `/logs` | 全public操作の成功/失敗/IP/日時、秘密非記録、管理画面フィルター |

## 非機能要件

| 要件 | 設計・検証 |
|---|---|
| NFR-001 | deployment HTTPS強制、HSTS、HTTPからのredirect試験 |
| NFR-002〜003 | security §3、password hash検査、TOTP/復旧試験 |
| NFR-004 | architecture 認証状態、全管理APIの未認証・MFA未完了試験 |
| NFR-005 | database/APIのprepared statement規約、SQLi payload試験 |
| NFR-006 | React escaping、CSP、ファイル名XSS試験 |
| NFR-007 | CSRF header + Origin検査、欠落/不一致試験 |
| NFR-008 | deploymentの公開外storage、URL直接指定404試験 |
| NFR-009 | architecture §5、偽装MIME、壊れたZIP、DOCX/XLSX偽装試験 |
| NFR-010 | architecture §4、token長/一意性/hash保存/ログ非露出試験 |

## 工程完了条件

### 第1工程: 設計

- [x] 画面遷移とUI-001〜UI-011を定義
- [x] DB物理設計と排他境界を定義
- [x] OpenAPI 3.1を定義
- [x] セキュリティ、配置、backup/recoveryを定義
- [x] OpenAPI lint
- [ ] Mermaid表示確認（GitHub表示時に実施）

### 第2工程以降

実装時は要件定義書の受け入れ基準を自動試験または手動試験IDへ割り当て、この文書へ結果と証跡を追記する。

