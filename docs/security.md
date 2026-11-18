# セキュリティ設計

## 1. 信頼境界と保護対象

保護対象は管理者資格情報、TOTP秘密、復旧コード、共有token、共有パスワード、保存ファイル、ファイル名、IPアドレス、アクセス履歴である。ブラウザ、HTTP入口、PHPアプリ、DB、非公開ストレージ、管理CLIを別の信頼境界として扱う。

## 2. セキュリティ制御

| 脅威 | 制御 |
|---|---|
| 管理者パスワード漏えい | `PASSWORD_ARGON2ID`を優先し、利用不可時だけ`PASSWORD_DEFAULT`。TOTP必須、復旧コードは一回限り。 |
| ログイン総当たり | IPとメールHMAC単位の試行制限、同一の失敗応答、構造化監査ログ。 |
| セッション固定・窃取 | 認証段階ごとのID再生成、Secure/HttpOnly/SameSite Cookie、30分無操作失効、管理/匿名Cookie分離。 |
| CSRF | 全状態変更でセッション紐付きCSRF tokenを検証。Originが同一であることも確認。 |
| XSS | Reactの既定エスケープ、`dangerouslySetInnerHTML`禁止、CSP、ファイル名をHTMLへ直接埋め込まない。 |
| SQL injection | PDO emulation無効、全値をprepared statementへbind。ORDER BY等はallowlist。 |
| IDOR | 管理APIは必ず`owner_id = authenticated_user_id`をSQL条件に含める。 |
| 共有token漏えい | 32 random bytes、URL fragment、resolve後の履歴除去、DBはhashのみ、`Referrer-Policy: no-referrer`。 |
| 共有パスワード攻撃 | `password_hash`、share ID + IPの試行制限、詳細を隠した応答。 |
| ファイル直接取得 | Web公開外へ保存し、認可済みPHPストリーム以外の経路を持たない。 |
| 偽装アップロード | サイズ、拡張子、finfo、magic/構造の多層検証。元ファイル名を保存パスに使わない。 |
| ZIP攻撃 | パストラバーサル、絶対パス、暗号化、entry数を検査。サーバでは展開しない。 |
| MIME sniffing | `X-Content-Type-Options: nosniff`、検証済みMIMEのみ送信。Office/ZIPはattachment固定。 |
| 競合による上限超過 | user/share行の`SELECT ... FOR UPDATE`と同一トランザクション内更新。 |
| 情報を含むエラー | 本番はstack trace非表示。request IDだけを利用者へ返し、詳細は非公開ログへ記録。 |
| 秘密のGit混入 | `.env`、鍵、`var/`、バックアップをignore。`.env.example`はダミー値だけ。 |

## 3. 暗号と秘密管理

- TOTPはRFC 6238、SHA-1、6桁、30秒step、許容窓は前後1step。同じuserの同じstepは一度しか受理しない。
- TOTP秘密は32 bytesの`APP_ENCRYPTION_KEY`を用いて libsodium secretbox で暗号化する。鍵はbase64で環境変数へ保存する。
- 認証試行のメールfingerprintは、別の`APP_HMAC_KEY`によるHMAC-SHA-256とする。
- 共有tokenは`random_bytes(32)`、復旧コードは各16 bytes相当のランダム性を持たせる。
- 秘密値、完全なCookie、Authorization情報、POST本文はログへ出力しない。
- バックアップは公開領域外へ置き、保存先の権限を所有者だけに制限する。

## 4. HTTPヘッダー

全応答:

```text
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Permissions-Policy: camera=(), microphone=(), geolocation=()
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' blob: data:; object-src 'none'; base-uri 'none'; frame-ancestors 'none'; form-action 'self'
```

- PDF previewは同一オリジンのblob URLを新規タブまたは埋め込みで表示する。CSPを緩和する場合も対象を`blob:`に限定する。
- cacheは管理JSONと共有メタデータを`Cache-Control: no-store`、ファイルstreamを`private, no-store`とする。

## 5. IPアドレス

- 原則は`REMOTE_ADDR`のみを信頼する。
- さくら側の信頼できるproxyを明示設定した場合だけ、そのproxyからの`X-Forwarded-For`最右端を採用する。
- DBには`inet_pton()`結果を16 bytes形式で保存し、表示時に`inet_ntop()`する。
- IPは個人情報として180日後にアクセス履歴とともに削除する。

## 6. 監査イベント

- public: resolve、unlock、preview、downloadの成功/失敗。
- auth: loginの成功/失敗/blocked。TOTP値や復旧コードは保存しない。
- アプリログ: request ID、ルート名、HTTP status、例外分類。生token、パスワード、ファイル内容、CSRF tokenは除外する。
- 失敗理由は`NOT_FOUND`、`EXPIRED`、`REVOKED`、`LIMIT_REACHED`、`PASSWORD_REQUIRED`、`INVALID_PASSWORD`、`RATE_LIMITED`、`STORAGE_ERROR`等の固定値。

## 7. 制約

- Phase 1ではマルウェア検査を行わない。管理者自身が信頼できるファイルだけを登録する前提をUIと運用文書へ明記する。
- ダウンロード回数制限はプレビューやクライアント側保存を技術的に完全制御しない。
- 共有URLを紛失した場合は再表示できない。既存共有を停止し、新しい共有を発行する。

