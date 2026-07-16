# デザインシステム仕様

## 1. 基本方針

- 白を基調とした業務システム向けのデザインとする。
- 視認性と操作性を最優先する。
- 過度な装飾、強いグラデーション、大きすぎる影は使用しない。
- ボタン、入力フォーム、モーダル、パネルには一貫した角丸を適用する。
- フォントは以下を使用する。

```css
font-family: 'DM Sans', sans-serif;
```

---

## 2. CSS変数

色、角丸、影、余白、文字サイズは可能な限りCSS変数に集約する。

推奨定義：

```css
:root {
  --white: #ffffff;

  --text1: #172033;
  --text2: #475569;
  --text-muted: #64748b;

  --primary: #2563eb;
  --primary-hover: #1d4ed8;
  --primary-soft: #eff6ff;

  --navy: #17345f;

  --danger: #dc2626;
  --danger-soft: #fef2f2;

  --warning: #d97706;
  --warning-soft: #fffbeb;

  --success: #16a34a;
  --success-soft: #f0fdf4;

  --status-not-started: #6b7280;
  --status-in-progress: #ca8a04;
  --status-completed: #16a34a;

  --bord1: #e2e8f0;
  --bord2: #cbd5e1;

  --surface-muted: #f8fafc;
  --surface-selected: #ffffff;
  --surface-unselected: #f1f5f9;

  --radius-s: 6px;
  --radius: 8px;
  --radius-l: 12px;

  --shadow-s: 0 1px 2px rgba(15, 23, 42, 0.08);
  --shadow-m: 0 8px 24px rgba(15, 23, 42, 0.12);

  --transition-fast: 150ms;
  --transition-panel: 220ms;

  --schedule-table-width: 420px;
}
```

既存のCSS変数がある場合は、重複して新設せず、既存命名へ統合する。

---

## 3. 共通ボタン

### 3.1 基本スタイル

青背景などの個別指定がないボタンには、以下の基本デザインを適用する。

```css
.button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  padding: 4px 10px;
  border-radius: var(--radius);
  border: 1px solid var(--bord2);
  background: var(--white);
  color: var(--text2);
  cursor: pointer;
  font-family: 'DM Sans', sans-serif;
  font-size: 10.5px;
  font-weight: 500;
  line-height: 1.4;
  transition:
    background-color var(--transition-fast),
    border-color var(--transition-fast),
    color var(--transition-fast),
    box-shadow var(--transition-fast),
    transform var(--transition-fast);
  white-space: nowrap;
  user-select: none;
  box-shadow: var(--shadow-s);
}
```

### 3.2 ホバー・フォーカス・押下

```css
.button:hover:not(:disabled) {
  background: var(--surface-muted);
  border-color: var(--primary);
}

.button:focus-visible {
  outline: 2px solid color-mix(in srgb, var(--primary) 35%, transparent);
  outline-offset: 2px;
}

.button:active:not(:disabled) {
  transform: translateY(1px);
}

.button:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}
```

- `transition: all` は使用せず、変化対象を明示する。
- キーボード操作時にフォーカス位置が分かるようにする。
- 無効状態は見た目だけでなく、操作も無効にする。

### 3.3 アイコン

- すべてのボタンには原則として先頭にHeroiconsを表示する。
- アイコンだけのボタンには、`aria-label` または同等の説明を付与する。
- Heroiconsで適切なアイコンが存在しない場合は、勝手に別ライブラリを追加せず報告する。

### 3.4 ボタン種別

#### Primary

保存、追加など主要操作に使用する。

- 背景：青
- 文字：白
- アイコン：白
- 枠線：青またはなし

#### Danger

削除、全削除など破壊的操作に使用する。

- 背景：白
- 文字：赤
- アイコン：赤
- 枠線：赤

#### Default

その他の一般操作に使用する。

- 背景：白
- 文字：黒または濃いグレー
- 枠線：グレー

---

## 4. ボタン別カラー

| 対象 | 背景 | 文字 | 枠線 | アイコン |
|---|---|---|---|---|
| 新規プロジェクト | 青 | 白 | 青またはなし | 白 |
| スケジュール追加 | 青 | 白 | 青またはなし | 白 |
| 削除 | 白 | 赤 | 赤 | 赤 |
| Excel | 白 | 黄緑 | 黄緑 | 黄緑 |
| PDF | 白 | 赤系 | 赤系 | 赤系 |
| マイルストーン | 白 | 黄系 | 黄系 | 黄系 |
| タスク管理 | 白 | 青 | 青 | 青 |
| マインドマップ | 白 | 赤系 | 赤系 | 赤系 |
| 即時メモ | 白 | 黄系 | 黄系 | 黄系 |
| その他 | 白 | 黒または濃いグレー | グレー | 同色 |

---

## 5. 表示単位

### 5.1 表示方式

- ドロップダウンは禁止する。
- 横並びのセグメントボタンとして表示する。
- 項目は以下とする。

```text
[日] [週] [1か月] [3か月]
```

### 5.2 選択状態

選択中：

- 背景：白
- 文字：青
- 枠線：青または選択が分かる表現
- `aria-pressed="true"` 等で状態を伝える

未選択：

- 背景：薄いグレー
- 文字：黒または濃いグレー
- 選択中との差が明確に分かること

---

## 6. 表示列

### 6.1 表示方式

- チェックボックスは禁止する。
- 横並びのセグメントボタンとして表示する。
- 項目は以下とする。

```text
[開始] [終了] [状態]
```

### 6.2 選択状態

選択中：

- 背景：白
- 文字：青

未選択：

- 背景：薄いグレー
- 文字：黒または濃いグレー

### 6.3 表示順

選択した順番に関係なく、スケジュール表では以下の順番で表示する。

1. 開始
2. 終了
3. 状態

---

## 7. 入力フォーム

- 入力フィールド、セレクト、テキストエリア、日付入力には角丸を適用する。
- 通常時の枠線は薄いグレーとする。
- 選択中・入力中のフィールドは枠線を青くする。
- `outline: none` のみでフォーカス表示を消すことは禁止する。
- placeholderは入力例として使用し、項目名の代わりにしない。
- 必須項目は視覚的かつ支援技術でも判別できるようにする。
- エラー時は赤枠とエラーメッセージを表示する。

推奨例：

```css
.form-control {
  width: 100%;
  border: 1px solid var(--bord2);
  border-radius: var(--radius);
  background: var(--white);
  color: var(--text1);
  font: inherit;
  padding: 8px 10px;
  transition:
    border-color var(--transition-fast),
    box-shadow var(--transition-fast);
}

.form-control:focus {
  border-color: var(--primary);
  box-shadow: 0 0 0 3px color-mix(in srgb, var(--primary) 18%, transparent);
  outline: none;
}
```

---

## 8. 状態色

状態表示は以下とする。

| 状態 | 色 |
|---|---|
| 未着手 | グレー |
| 進行中 | 黄色 |
| 完了 | 緑 |

- 色だけで状態を表さず、必ず状態名も表示する。
- 背景色を使う場合は、文字が読みやすいコントラストを確保する。

---

## 9. 優先度色

| 優先度 | 表示色 |
|---|---|
| 高 | 赤 |
| 中 | 黄色またはオレンジ |
| 低 | 青 |
| なし | グレー |

---

## 10. 日付色

- 期限超過：赤
- 期限内：青
- 完了済み：期限を過ぎていても青

---

## 11. カラー選択

選択可能な色は以下とする。

- 青
- 水色
- 緑
- 茶色
- 赤
- 紫
- ピンク
- オレンジ
- 黄緑
- グレー

### 表示方法

- 色名を並べるのではなく、四角形のカラースウォッチとして表示する。
- 選択中の色には枠線、チェック、拡大などの明確な選択表示を付ける。
- 各色には支援技術向けの名前を設定する。

---

## 12. 角丸

以下に角丸を適用する。

- ボタン
- 入力フォーム
- モーダル
- パネル
- タスクカード
- メモフレーム
- ドロップダウン
- 確認ダイアログ

リストの行をすべてカード状にする必要はない。角丸はコンポーネントの外枠に適用し、情報密度を下げすぎないようにする。

---

## 13. アニメーション

- 操作の理解を助ける短いアニメーションだけを使用する。
- モーダル表示は下からの短いスライドインとする。
- パネルは表示時にスライドイン、閉じる時にスライドアウトする。
- 目安は150ms〜250msとする。
- 過度なバウンド、回転、長いフェードは使用しない。
- `prefers-reduced-motion` を考慮する。

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    scroll-behavior: auto !important;
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## 14. 参考デザイン

以下を参考にしつつ、GanttForgeの仕様を優先する。

- Microsoft Project
- Backlog
- Jira Advanced Roadmaps
- Notion

参考サービスの外観をそのまま複製せず、業務システムとしての視認性を優先する。

---

## 15. 受け入れ条件

- DM Sansが全体に適用される。
- 通常ボタンに共通スタイルが適用される。
- 主要ボタンは青背景・白文字で表示される。
- 削除などの破壊的操作は赤系で表示される。
- ボタンには原則Heroiconsが付く。
- 表示単位は横並びセグメントで表示される。
- 表示列は横並びセグメントで表示される。
- 選択中と未選択の状態が明確に区別できる。
- 入力中のフィールドに青いフォーカス表示が出る。
- 色だけに依存せず状態や優先度を判別できる。
- アニメーションは短く、動きを減らす設定にも対応する。
