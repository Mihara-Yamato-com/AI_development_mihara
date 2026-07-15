# データ構造（IndexedDB）

IndexedDBのオブジェクトストア構成、各フィールドの型・意味をここに記載する。実装が進み次第、内容を追記・更新する。

## 重要: スキーマ変更時の注意

`onupgradeneeded`内でオブジェクトストアやインデックスを追加・変更した場合は、必ず`DB_VERSION`（`index.html`内の定数）を1つ上げること。バージョンを上げないと、既にDBを作成済みのブラウザでは`onupgradeneeded`が発火せず、新しいストアが作成されないままになる（1-Cで実際に発生した不具合）。

**複数ブランチを同じブラウザで検証する場合の注意**: 各ブランチが独立に`DB_VERSION`を上げていると、番号が衝突する（3-B/3-Cで発生）だけでなく、後で試したブランチの番号がブラウザのIndexedDBの現在バージョンより**低い**と`indexedDB.open()`が`VersionError`で失敗し、`db = await openDatabase()`の時点で例外が発生して以降の`bindEvents()`が実行されず、アプリの全ボタンが無反応になる（3-Dで実際に発生した不具合）。`init()`内で`openDatabase()`の失敗を捕捉し、DevToolsでの`GantForgeDB`削除を促すアラートを出すようにしている。複数ブランチを行き来して検証する際は、都度DevTools Application タブで`GantForgeDB`を削除してから開き直すのが安全。

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

### changeLogs
| フィールド | 型 | 意味 |
|---|---|---|
| id | number (autoIncrement) | 主キー |
| projectId | number | 紐づくプロジェクトのid（インデックス: `by_projectId`） |
| timestamp | number | 記録日時(epoch ms) |
| message | string | 変更内容を表す文言（例:「作業「〇〇」を追加しました」） |

補足: `changeLogs`は新規オブジェクトストアのため、`DB_VERSION`を3→4に変更した（3-Aで追加）。スケジュール項目の追加・編集・削除・状態変更・並び替え、マイルストーンの追加・削除、プロジェクトのロック操作・概要編集のタイミングで1件ずつ追記する。折りたたみ状態の変更など表示上のみの操作は対象外。

### memos
| フィールド | 型 | 意味 |
|---|---|---|
| id | number (autoIncrement) | 主キー |
| projectId | number | 紐づくプロジェクトのid（インデックス: `by_projectId`） |
| title | string | 題名 |
| body | string | 本文（Markdown） |
| updatedAt | number | 最終更新日時(epoch ms) |

補足: `memos`は新規オブジェクトストアのため`DB_VERSION`を上げた（3-Dで追加）。題名・本文とも入力するたびに約400ms後にIndexedDBへ自動保存する（打鍵ごとの書き込みを避けるためdebounceしている。`scheduleMemoSave`参照）。確定ボタンは無い。一覧は更新が新しい順。追加・削除は`changeLogs`に記録するが、内容編集は打鍵のたびに走ってしまうため記録しない。

本文はMarkdownの簡易サブセットのみ対応（`renderMarkdown`参照）：見出し`#`〜`###`、`**太字**`、`*斜体*`、`` `インラインコード` ``、`-`/`*`による箇条書き、空行区切りの段落。表・番号付きリスト・リンク・画像などは非対応。外部ライブラリを使わない方針のため自前実装している。

### quickMemos
| フィールド | 型 | 意味 |
|---|---|---|
| id | number (autoIncrement) | 主キー |
| projectId | number | 紐づくプロジェクトのid（インデックス: `by_projectId`） |
| text | string | 本文（Markdown記法で書けるが、レンダリングはしない） |
| timestamp | number | 投稿日時(epoch ms) |

補足: `quickMemos`は新規オブジェクトストアのため`DB_VERSION`を上げた（3-Dで追加、`memos`と同時）。`memos`と異なり複数件を素早く追加できる簡易メモで、題名やプレビューは無く、入力したテキストをそのまま表示する（「出力のみ」の仕様に対応）。追加・削除のみで編集機能は無い。一覧は新しい順。

### tasks
| フィールド | 型 | 意味 |
|---|---|---|
| id | number (autoIncrement) | 主キー |
| projectId | number | 紐づくプロジェクトのid（インデックス: `by_projectId`） |
| title | string | タスク内容 |
| status | string | 状態（未着手/進行中/完了） |
| order | number | カンバン内表示順（登録時のタイムスタンプ） |

補足: `tasks`はスケジュール項目(`scheduleItems`)とは独立したデータで、タスク管理パネル(P1)専用。新規オブジェクトストアのため`DB_VERSION`を上げた（3-Bで追加）。追加・編集・削除・状態変更(ドラッグ&ドロップ含む)は`changeLogs`にも記録する。
