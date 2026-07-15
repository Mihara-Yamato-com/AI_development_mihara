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

### comments
| フィールド | 型 | 意味 |
|---|---|---|
| id | number (autoIncrement) | 主キー |
| scheduleItemId | number | 紐づく作業(スケジュール項目)のid（インデックス: `by_scheduleItemId`） |
| text | string | コメント本文 |
| timestamp | number | 投稿日時(epoch ms) |

補足: `comments`は他のストアと異なり`projectId`ではなく`scheduleItemId`に紐づく（コメントは作業単位のため）。新規オブジェクトストアのため`DB_VERSION`を上げた（3-Cで追加）。閲覧・追加のみ行う（仕様通り、削除・編集機能は無し）。スケジュール項目自体を削除した際は、紐づくコメントも`deleteScheduleItem`内でカスケード削除する。追加時は`changeLogs`にも記録する。

コメントの追加はスケジュール編集モーダル内でのみ行う（作業内容・日付編集と合わせて、その作業のコメントを見ながら追加できる）。一方、操作バーの「コメント」ボタンから開く専用サイドパネル（`#comment-panel`、変更履歴と同じ`.side-panel`の枠組みを再利用）は、選択中プロジェクトの全作業のコメントを新しい順にまとめて閲覧できる読み取り専用のビューで、こちらからの追加はできない（変更履歴パネルと同じ位置づけ）。

### mindMapNodes
| フィールド | 型 | 意味 |
|---|---|---|
| id | number (autoIncrement) | 主キー |
| projectId | number | 紐づくプロジェクトのid（インデックス: `by_projectId`） |
| parentId | number\|null | 親ノードのid（ルートノードは`null`） |
| text | string | ノードのラベル |
| color | string | ノードの色（blue/purple/lightblue/green/brown/red/pink/orange/grayのいずれか） |
| x | number | キャンバス内のX座標(px) |
| y | number | キャンバス内のY座標(px) |

補足: `mindMapNodes`は新規オブジェクトストアのため`DB_VERSION`を上げた（3-Eで追加）。マインドマップパネルを開いた際、ルートノード(`parentId: null`)が無ければプロジェクト名で自動作成する。ノードの配置は自動レイアウトではなく、ドラッグによる手動配置(x/y座標を直接保持)。接続線は親子関係から毎回SVGで再計算して描画する（座標として保存はしない）。ルートノードは削除不可。ノードの追加・削除・リネームは`changeLogs`に記録するが、ドラッグ移動・色変更は記録しない（操作頻度が高く、ログが埋もれるため）。

### snapshots
| フィールド | 型 | 意味 |
|---|---|---|
| id | number (autoIncrement) | 主キー |
| projectId | number | 紐づくプロジェクトのid（インデックス: `by_projectId`） |
| createdAt | number | 保存日時(epoch ms) |
| data | Object | 保存時点のプロジェクト情報+紐づく全データ(下記参照) |

`data`の内訳:
| フィールド | 内容 |
|---|---|
| project | `{ name, description, locked }` |
| scheduleItems | `scheduleItems`の全レコード(id・parentIdは保存時点のまま) |
| milestones | `milestones`の全レコード |
| tasks | `tasks`の全レコード |
| comments | `comments`の全レコード(`scheduleItemId`は保存時点のid) |
| memos | `memos`の全レコード |
| quickMemos | `quickMemos`の全レコード |
| mindMapNodes | `mindMapNodes`の全レコード(id・parentIdは保存時点のまま) |

補足: `snapshots`は新規オブジェクトストアのため`DB_VERSION`を上げた（3-Fで追加）。プロジェクトに紐づく全データ（`changeLogs`を除く）を丸ごと1レコードのJSONとして保存する。`changeLogs`は監査ログのため対象外（復元すると過去の記録が消えてしまうため）。

**復元の仕組み**: 復元時は`clearProjectData`で現在のプロジェクトに紐づく全データを削除してから、スナップショットの内容を`insertRawRecord`で再投入する。`scheduleItems`と`mindMapNodes`は自己参照の`parentId`を持つため、再投入時に旧id→新idの対応表を作り、2周目で`parentId`を新しいidに付け替える。`comments`も同様に`scheduleItemId`を新idへ付け替えてから投入する。

保存・削除は`changeLogs`にも記録する。復元も記録するが、内容が大きいため各フィールドの詳細は記録しない。「現在と比較」はスケジュール項目のタイトルを突き合わせて追加/削除/変更（日付・状態）を簡易的に一覧表示するもので、厳密な差分アルゴリズムではない。

## データ出力(JSON)・データ取込 (4-A)

`snapshots`の`data`収集ロジックを`collectProjectData(projectId)`として切り出し、JSON出力・スナップショット保存の両方から共通で使う。同様に、スナップショット復元で使っていたid付け替え投入ロジックも`insertProjectData(projectId, data)`として切り出し、JSON取込からも使う。

- **JSON出力**: `collectProjectData`の結果をそのまま`JSON.stringify`し、`Blob`+`<a download>`でブラウザにダウンロードさせる（サーバー不要）。ファイル名は`プロジェクト名_YYYYMMDD_HHMM.json`（ファイル名に使えない文字は`_`に置換）。
- **データ取込**: `<input type="file">`で選択したJSONを`FileReader`(`File.text()`)で読み込み、`JSON.parse`する。最低限`project.name`が文字列であることだけ確認し、不正なら取り込まない。**既存プロジェクトは上書きせず、常に新規プロジェクトとして追加する**（スナップショット復元は「同じプロジェクトを過去の状態に戻す」用途だが、JSON取込は「外部データを持ち込む」用途のため、既存データを壊さない方を優先した）。
