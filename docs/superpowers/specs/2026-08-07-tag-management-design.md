# タグ管理機能

- 日付: 2026-08-07
- 対象ファイル: `bookmarks.html`, `bookmarks-filesync.html`

## 背景

現状、タグはブックマークごとの `tags[]` 配列に持たせているだけの単なる文字列ラベルで、専用の管理UIが無い。誤字の修正や不要になったタグの整理は、該当する全ブックマークを一つずつ編集フォームで開き直す必要があり手間が大きい。既に同様の課題を持つ「グループ」には管理モーダル（`#group-manage-overlay`、`bookmarks.html:1660-1752` 付近）があるため、同じパターンをタグに適用する。

## 要件

- タグのリネーム: タグ名を変更すると、そのタグを持つ全ブックマークに一括反映される。
- タグの削除: 指定したタグを、それを持つ全ブックマークの `tags[]` から一括除去する（ブックマーク自体は削除しない）。
- 上記2つ以外（並べ替え、専用マージUI）は今回のスコープ外。
- UI・操作感はグループ管理モーダルと同一パターンとする。

## 設計

### UI配置

- `#tag-nav-panel`（bookmarks.html / bookmarks-filesync.html 両方）内に、`#group-nav-panel` の `.sidebar-panel-actions` ラッパー＋「管理」ボタン（`#group-manage-open-btn`）と同じ構造で、`.sidebar-panel-actions` ラッパー＋「管理」ボタン（`#tag-manage-open-btn`、`aria-label="タグを管理"`）を新設する。
- クリックで新規モーダル `#tag-manage-overlay`（タイトル「タグ管理」、`role="dialog"` `aria-modal="true"`）を開く。閉じるボタン（`#tag-manage-close-btn`）と背景クリックでの閉じる挙動もグループ管理モーダルと同様。

### モーダルの中身

- `getTags()` の全タグ（アルファベット順、既存の並び順のまま）を `<ul id="tag-manage-list">` にリスト表示。
- グループ管理モーダルと異なり、ドラッグハンドル・上下移動ボタンは持たない（並べ替え非対応のため）。
- 各行（`renderTagManageRow(name)`）:
  - 通常時: タグ名表示 + リネームボタン（鉛筆アイコン、`data-action="edit-tag"`）+ 削除ボタン（`data-action="delete-tag"`）
  - `state.editingTag === name` の時: グループのリネームフォーム（`bookmarks.html:1700-1707`）と同じ構造のインラインフォーム（テキスト入力 + 保存 + キャンセル）
- タグが1件も無い場合の空状態メッセージ: 「タグがありません。ブックマークにタグを付けると、ここに表示されます。」（`#tag-manage-empty`）

### リネーム処理

グループのリネーム処理（`bookmarks.html:1732-1751`）と同じ構造の `submit` ハンドラを持つ。

- 新しい名前が空文字（trim後）または変更なしの場合は何もしない。
- 新しい名前が既存の別タグ名と衝突する場合: そのまま自然合流させる（同名タグは同一概念として扱う既存の設計に合わせる）。実装上は、全ブックマークをループし、`tags` 配列内の旧名を新名に置換した後、**同一ブックマーク内での重複を除去**する（1ブックマークが偶然すでに新名のタグも持っていた場合に配列内に同じ文字列が2つ並ぶのを防ぐため）。
- `state.tag`（現在の絞り込み中タグ）が旧名だった場合は新名に追従させる（グループの `if (state.group === oldName) state.group = newName;` と同じ扱い）。
- 完了後 `state.editingTag = null` にしてから `saveData(bookmarks)` → `render()`。

### 削除処理

- 削除ボタン押下時、対象タグを持つブックマーク件数を数え、`confirm('タグ「' + name + '」を全てのブックマーク(' + count + '件)から削除しますか?')` で確認する（既存の削除確認ダイアログ `bookmarks.html:2059` と同じ形式）。
- 確認後、全ブックマークの `tags` 配列から当該タグ文字列を除去する（`Array.prototype.filter`）。
- `saveData(bookmarks)` → `render()`。削除したタグが `state.tag` で絞り込み中だった場合は、既存の汎用クリーンアップ（後述）が自動的に解除する。

### 状態管理

- `state.editingTag`（初期値 `null`）を追加。`state.editingGroup` と対になるフィールドで、モーダル内でどの行がリネーム中かを保持する。
- `render()` 内に、`state.editingGroup` の既存クリーンアップ（`bookmarks.html:1286`: `if (state.editingGroup !== null && getGroups().indexOf(state.editingGroup) === -1) state.editingGroup = null;`）と同型の処理を `state.editingTag` にも追加する: `if (state.editingTag !== null && getTags().indexOf(state.editingTag) === -1) state.editingTag = null;`
- `state.tag` の既存クリーンアップ（`bookmarks.html:1275`）は変更不要（タグ削除時に対象タグが消えれば既存ロジックがそのまま `state.tag` をクリアする）。

### Escape キー連携

`closeTopmostLayer()`（`bookmarks.html:2256-2292`）に、グループ管理モーダルの分岐（`bookmarks.html:2263-2271`）と対になる分岐を追加する:

```js
if (!tagManageOverlay.hidden) {
  if (state.editingTag !== null) {
    state.editingTag = null;
    renderTagManageModal();
    return;
  }
  closeTagManageModal();
  return;
}
```

配置順序はグループ管理モーダルの分岐と同列（ヘルプパネルの直後）とする。両モーダルが同時に開くことは無い（一方がオーバーレイ表示中は、もう一方を開くボタンがそのオーバーレイの背後に隠れてクリックできない）ため、優先順位の曖昧さは発生しない。

グローバルショートカットの無効化ガード（`bookmarks.html:2383`: `if (!groupManageOverlay.hidden) return;`）にも同様の行を追加する: `if (!tagManageOverlay.hidden) return;`

### CSS

グループ管理モーダルのCSS（`bookmarks.html:522-621` 付近の `.group-manage-*` 系）は流用せず、`.tag-manage-*` という別名で必要な分だけ複製する（ドラッグハンドル・上下ボタン用のスタイルは不要なため、グループ側より少ない）。既存の安定した `.group-manage-*` 側のコードは一切変更しない。

## 変更しないもの

- タグの並べ替え・表示順（常にアルファベット順のまま）
- 明示的な「マージ」専用UI（リネーム時の自然合流のみでカバー）
- `getTags()` / `renderTagChipList()` などタグ一覧を参照している既存箇所のロジック（`getTags()` を都度呼び直しているため、リネーム・削除後の `render()` で自動的に反映される）
- グループ管理モーダル本体のコード・CSS

## 検証方法

自動テストは存在しないプロジェクトのため、ブラウザでの手動確認のみ（`bookmarks.html` と `bookmarks-filesync.html` の両方で実施）:

- 複数ブックマークに同じタグを付けた状態でタグ管理モーダルを開き、一覧に表示されることを確認
- タグをリネームし、そのタグを持つ全ブックマークのタグ表示・タグナビの一覧・絞り込み中だった場合のフィルタ表示が新名に切り替わることを確認
- 既存の別タグ名にリネームし、自然合流（重複タグが二重表示されない）することを確認
- タグを削除し、確認ダイアログに正しい件数が表示されること、削除後に対象ブックマークからタグが消え、タグナビ・絞り込みが自動的にクリアされることを確認
- モーダルを開いた状態で Escape を押し、リネームフォーム→モーダル本体の順に閉じることを確認
- モーダル表示中に他のキーボードショートカット（`n`, `t`, `g` 等）が無効化されていることを確認
- グループ管理モーダルの既存動作（リネーム・並べ替え）に副作用が無いことを確認

## スコープ外

- ショートカットキーの改善、追加フォームの簡略化は別スペックで扱う。
