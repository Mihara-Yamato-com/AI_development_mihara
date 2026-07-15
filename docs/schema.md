# データ構造（IndexedDB）

IndexedDBのオブジェクトストア構成、各フィールドの型・意味をここに記載する。実装が進み次第、内容を追記・更新する。

## 重要: スキーマ変更時の注意

`onupgradeneeded`内でオブジェクトストアやインデックスを追加・変更した場合は、必ず`DB_VERSION`（`index.html`内の定数）を1つ上げること。バージョンを上げないと、既にDBを作成済みのブラウザでは`onupgradeneeded`が発火せず、新しいストアが作成されないままになる（1-Cで実際に発生した不具合）。

## オブジェクトストア一覧

### projects
| フィールド | 型 | 意味 |
|---|---|---|
| id | number (autoIncrement) | 主キー |
| name | string | プロジェクト名 |
| description | string | 説明文 |
| createdAt | number | 作成日時(epoch ms) |
| updatedAt | number | 更新日時(epoch ms) |
| locked | boolean | ロック状態 |

### scheduleItems
| フィールド | 型 | 意味 |
|---|---|---|
| id | number (autoIncrement) | 主キー |
| projectId | number | 紐づくプロジェクトのid（インデックス: `by_projectId`） |
| title | string | 作業内容 |
| startDate | string | 開始日(YYYY-MM-DD) |
| endDate | string | 終了日(YYYY-MM-DD) |
| status | string | 状態（未着手/進行中/完了） |
| order | number | 表示順（登録時のタイムスタンプ） |
| parentId | number\|null | 親項目のid（トップレベルは`null`）。2-Bで追加 |
| collapsed | boolean | 子項目を折りたたみ中か。2-Bで追加 |

補足: `parentId`/`collapsed`はオブジェクトストア自体の変更ではなくフィールド追加のため、`DB_VERSION`の変更は不要（IndexedDBはスキーマレスなので既存レコードに無いフィールドは単に`undefined`になるだけ）。コード側は`item.parentId || null`・`item.collapsed || false`として扱い、既存データがあっても壊れないようにしている。

### milestones
| フィールド | 型 | 意味 |
|---|---|---|
| id | number (autoIncrement) | 主キー |
| projectId | number | 紐づくプロジェクトのid（インデックス: `by_projectId`） |
| date | string | 日付(YYYY-MM-DD) |
| label | string | マイルストーンのラベル |

補足: `milestones`は新規オブジェクトストアのため、`DB_VERSION`を2→3に変更した（2-Cで追加）。
