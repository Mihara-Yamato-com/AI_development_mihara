# docs/rules 準拠のUIデザイン刷新 プラン

## Context

developブランチに`docs/rules/`(01_ui_design.md, 02_layout.md, 03_gantt.md, 04_design_system.md)が追加され、
「この通りに実装してほしい」という指示を受けた。内容はボタンの配色・角丸・Heroicons化、ヘッダー/操作バーの左右配置、
表示単位・表示列のセグメントコントロール化、ガントチャートヘッダーの2段化とラベル書式、ガントバーの表示形式(親/子)、
スケジュール表の親行配色など、既存UIのほぼ全域に及ぶデザインシステム刷新。CLAUDE.mdの規約(機能分割・着手順序の提案、
大きな設計判断は事前確認)に従い、着手前にプランとして整理した。

現状把握(Explore調査結果):
- ヘッダー/操作バーのボタンは現状プレーンな`<button>`(角丸なし、アイコンなし、色指定なし) — `index.html:17-45`
- アイコン・SVGスプライトの仕組みは存在しない(マインドマップの接続線用SVGのみ、UIボタンとは無関係) — 新規導入になる
- スケジュール表の行は親子とも同じ`schedule-row`クラスで、インデントのみinline style — `index.html:3893-3965`
- `buildVisibleScheduleList`(`index.html:2777-2804`)は`{item, depth, hasChildren}`を返すのみで、
  どのトップレベル項目に属するかの情報(グループ番号)は持っていない → 親行配色のため要拡張
- ガントヘッダーは1段のみ(`renderGanttChart`, `index.html:4188-4300`)。日付ラベル書式は`formatDateLabel`固定(M/D)
- 祝日判定`getHolidayName`(`index.html:4161-4164`)はあるが、「土日祝を除いた実営業日を数える」ための
  再利用可能な関数は無い(週末判定は`renderGanttChart`内にインラインで書かれているのみ) → 新規ヘルパーを追加する
- `period-select`(ドロップダウン)と表示列チェックボックスは、それぞれ`change`イベントで`currentPeriod`/`visibleColumns`を
  更新し再描画するだけのシンプルな構造 — セグメントボタン化してもこの状態管理はそのまま流用できる

## 実装方針・着手順序

以下A〜Gの順で進める。5-Aは`feature/5-design-overhaul`ブランチにまとめて実装しdevelopへPR済み(#35)。
5-B以降は、Phase3(3-a, 3-b, 3-c...)と同様にサブステップごとに`feature/5-b-<内容>`のような作業ブランチを
developから切って進め、完了したサブステップごとにdevelopへのPRを出せる状態になったタイミングでユーザーに通告する
(2026-07-16、ユーザー指示によりブランチ分割方針に変更)。

**進捗状況**:
- [x] 5-A: 完了(実装日: 2026-07-15, ブランチ: feature/5-design-overhaul, PR #35)
- [x] 5-B: 実装完了(実装日: 2026-07-16, ブランチ: feature/5-b-button-icons)
- [x] 5-C: 実装完了(実装日: 2026-07-16, ブランチ: feature/5-c-header-toolbar-order)
- [x] 5-D: 実装完了(実装日: 2026-07-16, ブランチ: feature/5-d-segment-controls)
- [ ] 5-E: 未着手
- [ ] 5-F: 未着手
- [ ] 5-G: 未着手

次回セッション開始時は、このplanファイルを読み直し、内容をユーザーに再提示してから実装を再開すること。

### 5-A: デザイン基盤(Heroiconsスプライト + 共通ボタンCSS)
- `<body>`直下に`<svg style="display:none">`のHeroicons(outline, MIT License)シンボルスプライトを1つ定義する。
  必要なアイコン(新規/削除/出力/取込/Excel/PDF/ロック/追加/カレンダー/旗/鉛筆/時計/吹き出し/文書/雷/カンバン/カメラ/共有/
  ×/チェック/シェブロン等、約25種)を`<symbol id="icon-xxx" viewBox="0 0 24 24">`としてまとめて追加する
  (JSライブラリ・CDNではなく静的SVGパスの埋め込みなので、CLAUDE.mdの外部ライブラリ確認ルールの対象外と判断)
- 共通CSSクラスを追加: `.btn`(角丸・padding・flex+gapでアイコンとテキストを横並び)、
  `.btn-solid-blue`(新規プロジェクト/スケジュール追加用)、`.btn-outline-red`(削除全般/マインドマップ用)、
  `.btn-outline-green`(Excel用)、`.btn-outline-yellow`(マイルストーン/即時メモ用)、`.btn-outline-blue`(タスク管理用)、
  `.btn`のデフォルト(黒枠/黒文字)が「その他」に該当
- ヘッダー・操作バーの各ボタンに`.btn`+該当バリエーションクラス+アイコン`<use>`を適用する

### 5-B: 残りのボタンへのアイコン適用
- モーダル(キャンセル/保存/追加/閉じる等)、行操作ボタン(編集/削除/上へ/下へ)、各サイドパネルの閉じるボタン、
  スナップショットのホバーアイコン(比較/復元/削除)、マインドマップの右クリックメニュー・パネルボタンなど、
  ヘッダー/操作バー以外の全ボタンに同じ`.btn`基盤+アイコンを適用する

**実装メモ(2026-07-16)**:
- 新規アイコンを7種追加: `icon-x-mark`(キャンセル/閉じる), `icon-check`(保存/追加/確定), `icon-chevron-up`/`icon-chevron-down`(行の並び替え),
  `icon-arrows-right-left`(スナップショット比較), `icon-arrow-path`(スナップショット復元), `icon-arrows-pointing-out`(マインドマップ全体表示),
  `icon-paper-airplane`(コメント送信)
- アイコンのみで文字ラベルを持たない小型ボタン用に`.btn-icon-only`クラスを追加(padding: 4px)。スナップショット/メモ/即時メモの削除ボタン、
  行操作ボタン、タスクカード編集ボタン、パネルの×閉じるボタンに使用
- 配色は docs/rules の表に従い、削除系は全て`btn-outline-red`、マイルストーン追加・即時メモ追加は`btn-outline-yellow`、
  タスク追加は`btn-outline-blue`、スケジュール追加モーダルの確定ボタンはヘッダーの「スケジュール追加」と揃えて`btn-solid-blue`とした。
  保存/キャンセル/閉じる等、rulesの色分類に該当しない汎用アクションは`.btn`のデフォルト(黒)のまま
- 未使用になった`.btn-close-x`クラスは削除(スナップショット/マインドマップパネルの×ボタンは`.btn .btn-icon-only`に統一)
- メモ表示切替(編集/プレビュー/分割)のタブボタンとマインドマップのカラースウォッチは、5-Dのセグメントコントロール化と
  性質が近いため今回は対象外とし、現状のまま維持した

### 5-C: ヘッダー・操作バーの左右配置の並び替え
- ヘッダー左: 「GantForge」タイトル表示 → プロジェクト選択 → 新規プロジェクト作成 → プロジェクト削除
  (docs/rulesは「GanttForge」表記だが、CLAUDE.md記載の正式プロジェクト名「GantForge」を採用する)
- ヘッダー右: Excel出力 → PDF出力 → JSON出力 → データ取込
- 操作バー左: ロック → スケジュール追加 → 表示単位 → 今日 → すべて開く → 表示列 → マイルストーン → 編集
- 操作バー右: タスク管理 → マインドマップ → 変更履歴 → コメント → メモ → 即時メモ → スナップショット
  (現在「コメント」は左寄り・「変更履歴」は編集の後ろにあるため移動が必要。IDや`bindEvents`のイベント登録は
  一切変更せず、HTML内のボタンの並び順のみ変更する)

**実装メモ(2026-07-16)**:
- ヘッダー/操作バーそれぞれに`.app-header-left`/`.app-header-right`、`.toolbar-left`/`.toolbar-right`の
  ラッパーdivを追加し、親要素は`justify-content: space-between`で左右に分離した
- ボタンのid・`bindEvents`のイベント登録は変更せず、HTML内の記述順序とラッパーの所属のみ変更した
  (getElementByIdはDOM上の位置に依存しないため副作用なし)
- 「コメント」を操作バー右へ、「変更履歴」を「編集」の後ろから操作バー右の並びへ移動した

### 5-D: 表示単位・表示列のセグメントコントロール化
- `<select id="period-select">`を廃止し、`.segment-control`内に`[日][週][1か月][3か月]`の4ボタンを配置。
  クリックで`currentPeriod`を切り替え、選択中のみ`.is-active`(青)を付与して`renderGanttChart()`を呼ぶ
  (`period-select`の`change`ハンドラの中身をそのままクリックハンドラに移植)
- 表示列の3チェックボックス(開始/終了/状態)を同様に`.segment-control`のトグルボタン化。
  表示中=青、非表示=デフォルトのスタイルとし、`visibleColumns`の更新と`renderScheduleTable()`呼び出しはそのまま流用

**実装メモ(2026-07-16)**:
- `.segment-control`/`.segment-btn`/`.segment-btn.is-active`をCSSに追加(角丸枠+内部区切り線、選択中のみ青)
- `period-select`を`#period-segment`(`data-period`属性付きボタン4つ)に、開始/終了/状態のチェックボックスを
  `#column-segment`(`data-column`属性付きボタン3つ)に置き換え。イベントはクリック委譲で1つのlistenerに統合し、
  `currentPeriod`の切り替え・`visibleColumns`の更新・`renderGanttChart()`/`renderScheduleTable()`呼び出しは
  既存のロジックをそのまま流用した
- 表示単位のラベルは04_design_system.mdの表記に合わせ「1か月」「3か月」とした(旧`<select>`は「月」「3ヶ月」表記だった)
- docs/rules 01は「全ボタンにHeroicons付与」だが、04のセグメントコントロールのモックアップはテキストのみのため、
  この2種のセグメントボタンはアイコンなし(色による選択状態表現)とした
- 不要になった`.toolbar select`のCSSルールを削除(toolbar内に`<select>`が無くなったため)

### 5-E: ガントチャートヘッダーの2段化とラベル書式
- `renderGanttChart`(`index.html:4188-4300`)を改修し、ヘッダーを2段構成にする
  - 上段: 日・週表示中は「YYYY年MM月」、1か月表示中は「YYYY年Qn」、3か月表示中は「YYYY年」を、
    該当期間が連続する日数分の幅で結合表示するバンドを描画する(月/四半期/年の切り替わり位置を検出してグループ化)
  - 下段: 日→`M/D`(既存`formatDateLabel`を流用)、週→月曜日のみ`M/D`(既存ロジック踏襲)、
    1か月→月初のみ`M月`(新規`formatMonthLabel`)、3か月→四半期初のみ`Q1(1-3月)`形式(新規`formatQuarterLabel`、
    暦年の四半期: Q1=1-3月, Q2=4-6月, Q3=7-9月, Q4=10-12月)
  - `headerHeightPx`を2段分に拡張し、行位置計算・SVG系オーバーレイ(グリッド線/今日線/マイルストーン線)の
    高さ計算に影響しないよう調整する

### 5-F: ガントバーの表示形式(親はバー内・子はバー上)と実営業日
- 新規共有ヘルパー`isBusinessDay(date)`(土日+`getHolidayName`を使った祝日判定を1箇所に集約)と
  `countBusinessDays(startDate, endDate)`を追加する(既存のインライン週末判定と重複しないよう共通化)
- バーのラベルを「作業名（n日）」(nは実営業日数)に変更
- 親項目(`entry.hasChildren`)はラベルをバー内(現状通り`.gantt-bar`内)に表示。
  子項目(leaf)はラベルをバー上(バーの上に浮かせた別要素、例:バーの少し上に配置する`.gantt-bar-label`)に表示
- 描画後にDOM計測(`scrollWidth > clientWidth`)で文字が収まらない場合はラベルを非表示にする

### 5-G: スケジュール表の親行配色
- `buildVisibleScheduleList`が返すオブジェクトに`groupIndex`(トップレベル項目の並び順インデックス)を追加する
  (既存の呼び出し元(ガントチャート/Excel出力)は新フィールドを無視するだけなので後方互換)
- 親行(`entry.hasChildren`)に、`groupIndex`を4〜5色程度の薄い青系パレットで循環させた背景色をinline styleで適用する
  (子行は現状通り無色のまま)

## 検証方法

各ステップ完了後、`git status`で差分を確認しつつ、Windows Chromeのヘッドレスモード
(`chrome.exe --headless --disable-gpu --screenshot=... file:///...`、本セッションで確立済みの手法)で
実際の画面をスクリーンショットして目視確認する。IndexedDBに依存する動作(プロジェクト作成後のガントバー等)は
実データを1件作成した状態でスクリーンショットを取り、レイアウト崩れがないか確認する。
全ステップ完了後、ユーザーにもブラウザでの手動確認手順を案内する。
