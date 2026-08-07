# タグ管理機能 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** サイドバーの「タグ」タブに、グループ管理モーダルと同じパターンの「タグ管理」モーダルを追加し、タグのリネーム（全ブックマークへの一括反映・既存タグへの自然合流）とタグの削除（全ブックマークからの一括除去、確認ダイアログ付き）を可能にする。

**Architecture:** `bookmarks.html` にまず機能一式（CSS・HTML・JS）を実装し、ブラウザ相当の手動確認を経てから、`bookmarks-filesync.html` へ同一内容を移植する。2ファイルはUI/ロジックをほぼ完全に重複させる既存方針（`CLAUDE.md` 記載）に従う。並べ替えは対象外のため、グループ管理モーダルからドラッグ&ドロップ・上下ボタンに関する部分を除いた縮小版として実装する。

**Tech Stack:** 素のHTML/CSS/JS（ビルドなし、フレームワークなし）。自動テストは存在しないため、ブラウザでの手動確認（またはブラウザが使えない場合の静的確認）で検証する。

## Global Constraints

- 対象ファイルは `bookmarks.html` と `bookmarks-filesync.html` の2つのみ。
- タグの並べ替え機能は追加しない（常にアルファベット順のまま）。
- 明示的な「マージ」専用UIは追加しない。リネーム先が既存の別タグ名と衝突した場合は自然合流させる（グループのリネームと同じ挙動）。
- リネーム時、同一ブックマーク内で重複するタグ文字列が発生しないよう重複除去を行う。
- 削除は確認ダイアログ（対象件数を表示）を経てから実行する。
- グループ管理モーダル本体のコード・CSS・ロジックは一切変更しない。
- `getTags()` や `renderTagChipList()` など、既存のタグ一覧参照箇所のロジックは変更しない（`render()` 呼び出しにより自動的に反映される）。

---

### Task 1: `bookmarks.html` にタグ管理モーダルを実装する

**Files:**
- Modify: `bookmarks.html:624`（CSS追加）
- Modify: `bookmarks.html:1004-1006`（`#tag-nav-panel` にボタン追加）
- Modify: `bookmarks.html:1035`（`#group-manage-overlay` の直後にモーダルHTML追加）
- Modify: `bookmarks.html:1057`（`state.editingTag` 追加）
- Modify: `bookmarks.html:1286`（`editingTag` クリーンアップ追加）
- Modify: `bookmarks.html:1303`（モーダル再描画呼び出し追加）
- Modify: `bookmarks.html:1815`（タグ管理モーダルのJS一式を追加）
- Modify: `bookmarks.html:2271`（`closeTopmostLayer()` にタグモーダル分岐追加）
- Modify: `bookmarks.html:2383`（キーボードショートカットガードに追加）

**Interfaces:**
- Consumes: 既存の `getTags()`（`bookmarks.html:1244-1248`）、`escapeHtml()`、`saveData(bookmarks)`、`render()`、`state` オブジェクト
- Produces: `openTagManageModal()`, `closeTagManageModal()`, `renderTagManageModal()`, `renderTagManageRow(name)`, `bindTagManageRowButtons()`, `deleteTag(name)` — Task 2ではこれらと同名・同シグネチャの関数を `bookmarks-filesync.html` にも定義する

- [ ] **Step 1: CSSを追加する**

`bookmarks.html:621-624` は現在:

```css
  .group-manage-empty {
    font-size: 0.85rem;
    color: var(--ink-soft);
  }
```

この直後（625行目、`.group-section-empty` の手前）に次のCSSを追加する:

```css

  /* ---------- タグ管理モーダル ---------- */

  .tag-manage-open-btn {
    background: transparent;
    border: 1px solid var(--ink-soft);
    color: var(--ink);
    font-family: var(--font-mono);
    font-size: 0.75rem;
    font-weight: 600;
    padding: 3px 10px;
    border-radius: 4px;
  }
  .tag-manage-open-btn:hover { background: var(--ink); color: var(--ivory); }

  .tag-manage-list {
    list-style: none;
    margin: 0;
    padding: 0;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  .tag-manage-row {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 6px 8px;
    border: 1px solid var(--sage);
    border-radius: 6px;
    background: rgba(27, 53, 56, 0.03);
  }

  .tag-manage-name {
    flex: 1 1 auto;
    font-family: var(--font-display);
    font-weight: 700;
    font-size: 0.95rem;
    min-width: 0;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .tag-manage-empty {
    font-size: 0.85rem;
    color: var(--ink-soft);
  }

  .tag-edit-btn {
    background: transparent;
    border: none;
    color: var(--ink-soft);
    font-size: 0.85rem;
    line-height: 1;
    padding: 2px 4px;
    border-radius: 4px;
  }
  .tag-edit-btn:hover { color: var(--ink); background: rgba(27, 53, 56, 0.08); }

  .tag-delete-btn {
    background: transparent;
    border: none;
    color: var(--ink-soft);
    font-size: 0.85rem;
    line-height: 1;
    padding: 2px 4px;
    border-radius: 4px;
  }
  .tag-delete-btn:hover { color: var(--wine); background: rgba(122, 46, 54, 0.08); }

  .tag-rename-form {
    display: flex;
    align-items: center;
    gap: 6px;
    flex: 1 1 auto;
    min-width: 0;
  }
  .tag-rename-input {
    font-family: var(--font-display);
    font-weight: 700;
    font-size: 1.05rem;
    padding: 2px 6px;
    border: 1px solid var(--sage);
    border-radius: 4px;
    min-width: 0;
    flex: 1 1 auto;
  }
  .tag-rename-form .btn, .tag-rename-form .btn-quiet { padding: 4px 10px; font-size: 0.75rem; }
```

- [ ] **Step 2: `#tag-nav-panel` に「管理」ボタンを追加する**

`bookmarks.html:1004-1006` は現在:

```html
    <div class="sidebar-panel" id="tag-nav-panel" role="tabpanel">
      <ul id="tag-nav"></ul>
    </div>
```

次のように変更する:

```html
    <div class="sidebar-panel" id="tag-nav-panel" role="tabpanel">
      <div class="sidebar-panel-actions">
        <button type="button" class="tag-manage-open-btn" id="tag-manage-open-btn" aria-label="タグを管理">管理</button>
      </div>
      <ul id="tag-nav"></ul>
    </div>
```

- [ ] **Step 3: タグ管理モーダルのHTMLを追加する**

`bookmarks.html:1026-1035` は現在:

```html
<div class="modal-overlay" id="group-manage-overlay" hidden>
  <div class="modal" role="dialog" aria-modal="true" aria-labelledby="group-manage-title">
    <div class="modal-header">
      <h2 id="group-manage-title">グループ管理</h2>
      <button type="button" class="modal-close-btn" id="group-manage-close-btn" aria-label="閉じる">&times;</button>
    </div>
    <ul class="group-manage-list" id="group-manage-list"></ul>
    <p class="group-manage-empty" id="group-manage-empty" hidden>グループがありません。ブックマークをグループにまとめると、ここに表示されます。</p>
  </div>
</div>
```

この直後（1036行目）に次のHTMLを追加する:

```html

<div class="modal-overlay" id="tag-manage-overlay" hidden>
  <div class="modal" role="dialog" aria-modal="true" aria-labelledby="tag-manage-title">
    <div class="modal-header">
      <h2 id="tag-manage-title">タグ管理</h2>
      <button type="button" class="modal-close-btn" id="tag-manage-close-btn" aria-label="閉じる">&times;</button>
    </div>
    <ul class="tag-manage-list" id="tag-manage-list"></ul>
    <p class="tag-manage-empty" id="tag-manage-empty" hidden>タグがありません。ブックマークにタグを付けると、ここに表示されます。</p>
  </div>
</div>
```

- [ ] **Step 4: `state.editingTag` を追加する**

`bookmarks.html:1052-1061` は現在:

```js
  var state = {
    search: '',
    tag: null,
    group: null,
    editingId: null,
    editingGroup: null,
    selectMode: false,
    selectedIds: new Set(),
    sidebarTab: 'tag'
  };
```

`editingGroup: null,` の直後に `editingTag: null,` を追加する:

```js
  var state = {
    search: '',
    tag: null,
    group: null,
    editingId: null,
    editingGroup: null,
    editingTag: null,
    selectMode: false,
    selectedIds: new Set(),
    sidebarTab: 'tag'
  };
```

- [ ] **Step 5: `render()` に `editingTag` のクリーンアップを追加する**

`bookmarks.html:1286` は現在:

```js
    if (state.editingGroup !== null && getGroups().indexOf(state.editingGroup) === -1) state.editingGroup = null;
```

この直後に次の行を追加する:

```js
    // リネーム中のタグが他の操作(削除・別リネームでの合流)で消滅した場合、編集状態を閉じる
    if (state.editingTag !== null && getTags().indexOf(state.editingTag) === -1) state.editingTag = null;
```

- [ ] **Step 6: `render()` にタグ管理モーダルの再描画呼び出しを追加する**

`bookmarks.html:1303` は現在:

```js
    if (!groupManageOverlay.hidden) renderGroupManageModal(); // モーダルが開いている間は常に最新の並び・件数を反映する
```

この直後に次の行を追加する:

```js
    if (!tagManageOverlay.hidden) renderTagManageModal(); // モーダルが開いている間は常に最新の一覧を反映する
```

- [ ] **Step 7: タグ管理モーダルのJS一式を追加する**

`bookmarks.html:1804-1815` は現在:

```js
  function bindGroupManageRowDragEvents() {
    groupManageList.querySelectorAll('.group-manage-row[draggable="true"]').forEach(function (row) {
      bindDragReorder(row, row.getAttribute('data-group'), {
        getDragged: function () { return manageDraggedGroup; },
        setDragged: function (v) { manageDraggedGroup = v; },
        payload: 'group', // 値自体は未使用(dropはmanageDraggedGroupを参照)。URL状のグループ名がドロップ失敗時に誤ナビゲーションを起こさないよう固定文字列にする
        onDrop: function (draggedKey, targetKey, after) {
          moveGroupSection(draggedKey, targetKey, after); // render()経由でrenderGroupManageModal()も再描画される
        }
      });
    });
  }
```

この関数の直後（1816行目、`/* ---------- ドラッグ&ドロップ(並べ替え・グループ間移動) ---------- */` コメントの手前）に次のブロックを追加する:

```js

  /* ---------- タグ管理モーダル ---------- */

  var tagManageOverlay = document.getElementById('tag-manage-overlay');
  var tagManageList = document.getElementById('tag-manage-list');
  var tagManageEmpty = document.getElementById('tag-manage-empty');

  var tagManageOpenBtn = document.getElementById('tag-manage-open-btn');
  tagManageOpenBtn.addEventListener('click', openTagManageModal);
  document.getElementById('tag-manage-close-btn').addEventListener('click', closeTagManageModal);
  tagManageOverlay.addEventListener('click', function (e) {
    if (e.target === tagManageOverlay) closeTagManageModal(); // 背景クリックで閉じる
  });

  function openTagManageModal() {
    tagManageOverlay.hidden = false;
    renderTagManageModal();
    var firstRow = tagManageList.querySelector('.tag-manage-row');
    (firstRow || document.getElementById('tag-manage-close-btn')).focus();
  }

  function closeTagManageModal() {
    tagManageOverlay.hidden = true;
    state.editingTag = null; // 閉じたら行内のリネームフォームも一緒に閉じておく
    tagManageOpenBtn.focus(); // 開いた場所へフォーカスを戻す
  }

  // タグ管理モーダルのリスト全体を再描画する。render()から、モーダルが開いている間だけ呼ばれる。
  function renderTagManageModal() {
    var names = getTags();
    tagManageEmpty.hidden = names.length !== 0;
    tagManageList.innerHTML = names.map(function (name) {
      return renderTagManageRow(name);
    }).join('');
    bindTagManageRowButtons();
  }

  function renderTagManageRow(name) {
    if (state.editingTag === name) {
      return '<li class="tag-manage-row" data-tag="' + escapeHtml(name) + '">' +
        '<form class="tag-rename-form" data-tag-rename-form data-tag="' + escapeHtml(name) + '">' +
          '<input type="text" class="tag-rename-input" name="tagName" value="' + escapeHtml(name) + '" required>' +
          '<button type="submit" class="btn btn-amber">保存</button>' +
          '<button type="button" class="btn-quiet" data-action="cancel-tag-rename">キャンセル</button>' +
        '</form>' +
      '</li>';
    }
    return '<li class="tag-manage-row" tabindex="-1" data-tag="' + escapeHtml(name) + '">' +
      '<span class="tag-manage-name">' + escapeHtml(name) + '</span>' +
      '<button type="button" class="tag-edit-btn" data-action="edit-tag" data-tag="' + escapeHtml(name) + '" aria-label="' + escapeHtml(name) + 'の名前を変更">&#9998;</button>' +
      '<button type="button" class="tag-delete-btn" data-action="delete-tag" data-tag="' + escapeHtml(name) + '" aria-label="' + escapeHtml(name) + 'を削除">&#10005;</button>' +
    '</li>';
  }

  // 行の鉛筆(リネーム開始)・削除ボタン・リネームフォームのsubmit/キャンセルを割り当てる
  function bindTagManageRowButtons() {
    tagManageList.querySelectorAll('[data-action="edit-tag"]').forEach(function (btn) {
      btn.addEventListener('click', function () { state.editingTag = btn.getAttribute('data-tag'); renderTagManageModal(); });
    });
    tagManageList.querySelectorAll('[data-action="cancel-tag-rename"]').forEach(function (btn) {
      btn.addEventListener('click', function () { state.editingTag = null; renderTagManageModal(); });
    });
    tagManageList.querySelectorAll('[data-action="delete-tag"]').forEach(function (btn) {
      btn.addEventListener('click', function () { deleteTag(btn.getAttribute('data-tag')); });
    });
    tagManageList.querySelectorAll('[data-tag-rename-form]').forEach(function (form) {
      form.addEventListener('submit', function (e) {
        e.preventDefault();
        var oldName = form.getAttribute('data-tag');
        var newName = form.tagName.value.trim();
        if (newName && newName !== oldName) {
          bookmarks.forEach(function (b) {
            var tags = b.tags || [];
            if (tags.indexOf(oldName) === -1) return;
            var renamed = tags.map(function (t) { return t === oldName ? newName : t; });
            b.tags = renamed.filter(function (t, i) { return renamed.indexOf(t) === i; }); // 同一ブックマーク内での重複除去(改名先が既存の別タグと衝突した場合)
          });
          saveData(bookmarks);
          if (state.tag === oldName) state.tag = newName;
        }
        state.editingTag = null;
        render(); // カードのタグ表示・タグナビも変わるため、モーダルだけでなく全体を再描画する
      });
    });
  }

  // 指定タグを持つ全ブックマークから、そのタグだけを取り除く(ブックマーク自体は削除しない)
  function deleteTag(name) {
    var count = bookmarks.filter(function (b) { return (b.tags || []).indexOf(name) !== -1; }).length;
    if (count === 0) return;
    if (!confirm('タグ「' + name + '」を全てのブックマーク(' + count + '件)から削除しますか?')) return;
    bookmarks.forEach(function (b) {
      if (!b.tags) return;
      b.tags = b.tags.filter(function (t) { return t !== name; });
    });
    saveData(bookmarks);
    render();
  }
```

- [ ] **Step 8: `closeTopmostLayer()` にタグ管理モーダルの分岐を追加する**

`bookmarks.html:2262-2271` は現在:

```js
    // グループ管理モーダルが開いている場合、内側のリネームフォーム→モーダル本体の順に1段階ずつ閉じる
    if (!groupManageOverlay.hidden) {
      if (state.editingGroup !== null) {
        state.editingGroup = null;
        renderGroupManageModal();
        return;
      }
      closeGroupManageModal();
      return;
    }
```

この直後（2272行目、`var closedEditingCard = ...` の手前）に次のブロックを追加する:

```js

    // タグ管理モーダルが開いている場合も同様に、内側のリネームフォーム→モーダル本体の順に1段階ずつ閉じる
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

- [ ] **Step 9: グローバルショートカットガードにタグ管理モーダルを追加する**

`bookmarks.html:2383` は現在:

```js
    if (!groupManageOverlay.hidden) return; // モーダル表示中はEscape以外のグローバルショートカットを無効化する
```

この直後に次の行を追加する:

```js
    if (!tagManageOverlay.hidden) return; // モーダル表示中はEscape以外のグローバルショートカットを無効化する
```

- [ ] **Step 10: `bookmarks.html` を手動確認する**

自動テストは存在しないため、ブラウザでの手動確認を行う（ヘッドレス環境でブラウザが使えない場合は、後述の代替確認で置き換える）。

1. `bookmarks.html` を開き、少なくとも2件のブックマークに同じタグ（例: `test-tag`）を、別の1件に別タグ（例: `other-tag`）を付ける（既存の追加フォームを使う）。
2. サイドバーの「タグ」タブを開き、新設した「管理」ボタンをクリックしてタグ管理モーダルが開くことを確認する。`test-tag` と `other-tag` が一覧に表示されることを確認する。
3. `test-tag` の行の鉛筆ボタンをクリックし、リネームフォームが開くことを確認する。名前を `renamed-tag` に変えて保存し、モーダルの一覧が更新されること、`test-tag` を持っていた2件のブックマークのタグ表示が `renamed-tag` に変わっていることを確認する。
4. `renamed-tag` を `other-tag`（既存の別タグ）にリネームし、確認ダイアログなしでそのまま合流すること、`other-tag` を持つブックマークが3件になること、リネーム前に両方のタグを持っていたブックマークが存在した場合はタグ表示に `other-tag` が重複して2つ表示されないことを確認する（このテストケースでは重複ケースを作るため、事前にどれか1件のブックマークに `renamed-tag` と `other-tag` の両方を手動で付けておくこと）。
5. `other-tag` の削除ボタンをクリックし、確認ダイアログに正しい件数（3件）が表示されることを確認する。OKを押し、対象ブックマークから `other-tag` が消え、タグナビからも `other-tag` が消えることを確認する。
6. モーダルを再度開いた状態で Escape キーを押し、モーダルが閉じることを確認する。リネームフォームを開いた状態で Escape を押すと、フォームだけが閉じてモーダルは開いたままであることを確認する。
7. モーダル表示中に `n` キーを押しても追加パネルが開かないこと（グローバルショートカットが無効化されていること）を確認する。
8. グループ管理モーダル（「グループ」タブ内の「管理」ボタン）を開き、既存のリネーム・並べ替え機能が引き続き正常に動作することを確認する（今回の変更による副作用が無いことの確認）。

ヘッドレス環境等でブラウザ（`DISPLAY`）が使えない場合は、代わりに次の静的確認を行い、その旨をレポートに明記する:
- `grep -n "tag-manage" bookmarks.html` で、Step 1〜9で追加したすべてのID・クラス名（`tag-manage-overlay`, `tag-manage-list`, `tag-manage-empty`, `tag-manage-open-btn`, `tag-manage-row`, `tag-manage-name`, `tag-edit-btn`, `tag-delete-btn`, `tag-rename-form`, `tag-rename-input`）がHTML側の定義とJS側の参照で過不足なく対応していることを確認する。
- `state.editingTag`、`tagManageOverlay`、`renderTagManageModal`、`closeTagManageModal` の各識別子が定義前に参照されていないか（変数・関数の宣言順序）を目視確認する。
- ブラウザでのインタラクティブ確認（上記1〜8）は未実施であり、マージ前に人間による実ブラウザでの確認が必要である旨をレポートに明記する。

- [ ] **Step 11: Commit**

```bash
git add bookmarks.html
git commit -m "feat: add tag management modal (rename, delete) to bookmarks.html"
```

---

### Task 2: `bookmarks-filesync.html` に同一の変更を移植する

**Files:**
- Modify: `bookmarks-filesync.html:697`（CSS追加）
- Modify: `bookmarks-filesync.html:1104-1105`（`#tag-nav-panel` にボタン追加）
- Modify: `bookmarks-filesync.html:1134`（`#group-manage-overlay` の直後にモーダルHTML追加）
- Modify: `bookmarks-filesync.html:1151-1160`（`state.editingTag` 追加）
- Modify: `bookmarks-filesync.html:1588`（`editingTag` クリーンアップ追加）
- Modify: `bookmarks-filesync.html:1605`（モーダル再描画呼び出し追加）
- Modify: `bookmarks-filesync.html:2118`（`bindGroupManageRowDragEvents()` 関数の直後にタグ管理モーダルのJS一式を追加）
- Modify: `bookmarks-filesync.html:2573`（`closeTopmostLayer()` にタグモーダル分岐追加）
- Modify: `bookmarks-filesync.html:2685`（キーボードショートカットガードに追加）

**Interfaces:**
- Consumes: Task 1と同じ関数群（`getTags()`, `escapeHtml()`, `saveData(bookmarks)`, `render()`）の `bookmarks-filesync.html` 側の定義。関数名・シグネチャはTask 1と完全に同一。
- Produces: なし（本プラン最終タスク）

- [ ] **Step 1: CSSを追加する**

`bookmarks-filesync.html:697-700` は現在:

```css
  .group-manage-empty {
    font-size: 0.85rem;
    color: var(--ink-soft);
  }
```

この直後に、Task 1 Step 1 と全く同じ内容のCSSブロックを追加する:

```css

  /* ---------- タグ管理モーダル ---------- */

  .tag-manage-open-btn {
    background: transparent;
    border: 1px solid var(--ink-soft);
    color: var(--ink);
    font-family: var(--font-mono);
    font-size: 0.75rem;
    font-weight: 600;
    padding: 3px 10px;
    border-radius: 4px;
  }
  .tag-manage-open-btn:hover { background: var(--ink); color: var(--ivory); }

  .tag-manage-list {
    list-style: none;
    margin: 0;
    padding: 0;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  .tag-manage-row {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 6px 8px;
    border: 1px solid var(--sage);
    border-radius: 6px;
    background: rgba(27, 53, 56, 0.03);
  }

  .tag-manage-name {
    flex: 1 1 auto;
    font-family: var(--font-display);
    font-weight: 700;
    font-size: 0.95rem;
    min-width: 0;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .tag-manage-empty {
    font-size: 0.85rem;
    color: var(--ink-soft);
  }

  .tag-edit-btn {
    background: transparent;
    border: none;
    color: var(--ink-soft);
    font-size: 0.85rem;
    line-height: 1;
    padding: 2px 4px;
    border-radius: 4px;
  }
  .tag-edit-btn:hover { color: var(--ink); background: rgba(27, 53, 56, 0.08); }

  .tag-delete-btn {
    background: transparent;
    border: none;
    color: var(--ink-soft);
    font-size: 0.85rem;
    line-height: 1;
    padding: 2px 4px;
    border-radius: 4px;
  }
  .tag-delete-btn:hover { color: var(--wine); background: rgba(122, 46, 54, 0.08); }

  .tag-rename-form {
    display: flex;
    align-items: center;
    gap: 6px;
    flex: 1 1 auto;
    min-width: 0;
  }
  .tag-rename-input {
    font-family: var(--font-display);
    font-weight: 700;
    font-size: 1.05rem;
    padding: 2px 6px;
    border: 1px solid var(--sage);
    border-radius: 4px;
    min-width: 0;
    flex: 1 1 auto;
  }
  .tag-rename-form .btn, .tag-rename-form .btn-quiet { padding: 4px 10px; font-size: 0.75rem; }
```

- [ ] **Step 2: `#tag-nav-panel` に「管理」ボタンを追加する**

`bookmarks-filesync.html:1104-1106` は現在:

```html
    <div class="sidebar-panel" id="tag-nav-panel" role="tabpanel">
      <ul id="tag-nav"></ul>
    </div>
```

Task 1 Step 2 と全く同じ内容に変更する:

```html
    <div class="sidebar-panel" id="tag-nav-panel" role="tabpanel">
      <div class="sidebar-panel-actions">
        <button type="button" class="tag-manage-open-btn" id="tag-manage-open-btn" aria-label="タグを管理">管理</button>
      </div>
      <ul id="tag-nav"></ul>
    </div>
```

- [ ] **Step 3: タグ管理モーダルのHTMLを追加する**

`bookmarks-filesync.html:1126-1134` は現在:

```html
<div class="modal-overlay" id="group-manage-overlay" hidden>
  <div class="modal" role="dialog" aria-modal="true" aria-labelledby="group-manage-title">
    <div class="modal-header">
      <h2 id="group-manage-title">グループ管理</h2>
      <button type="button" class="modal-close-btn" id="group-manage-close-btn" aria-label="閉じる">&times;</button>
    </div>
    <ul class="group-manage-list" id="group-manage-list"></ul>
    <p class="group-manage-empty" id="group-manage-empty" hidden>グループがありません。ブックマークをグループにまとめると、ここに表示されます。</p>
  </div>
</div>
```

この直後に、Task 1 Step 3 と全く同じ内容のHTMLを追加する:

```html

<div class="modal-overlay" id="tag-manage-overlay" hidden>
  <div class="modal" role="dialog" aria-modal="true" aria-labelledby="tag-manage-title">
    <div class="modal-header">
      <h2 id="tag-manage-title">タグ管理</h2>
      <button type="button" class="modal-close-btn" id="tag-manage-close-btn" aria-label="閉じる">&times;</button>
    </div>
    <ul class="tag-manage-list" id="tag-manage-list"></ul>
    <p class="tag-manage-empty" id="tag-manage-empty" hidden>タグがありません。ブックマークにタグを付けると、ここに表示されます。</p>
  </div>
</div>
```

- [ ] **Step 4: `state.editingTag` を追加する**

`bookmarks-filesync.html:1151-1160` は現在:

```js
  var state = {
    search: '',
    tag: null,
    group: null,
    editingId: null,
    editingGroup: null,
    selectMode: false,
    selectedIds: new Set(),
    sidebarTab: 'tag'
  };
```

`editingGroup: null,` の直後に `editingTag: null,` を追加する:

```js
  var state = {
    search: '',
    tag: null,
    group: null,
    editingId: null,
    editingGroup: null,
    editingTag: null,
    selectMode: false,
    selectedIds: new Set(),
    sidebarTab: 'tag'
  };
```

- [ ] **Step 5: `render()` に `editingTag` のクリーンアップを追加する**

`bookmarks-filesync.html:1588` は現在:

```js
    if (state.editingGroup !== null && getGroups().indexOf(state.editingGroup) === -1) state.editingGroup = null;
```

この直後に、Task 1 Step 5 と全く同じ行を追加する:

```js
    // リネーム中のタグが他の操作(削除・別リネームでの合流)で消滅した場合、編集状態を閉じる
    if (state.editingTag !== null && getTags().indexOf(state.editingTag) === -1) state.editingTag = null;
```

- [ ] **Step 6: `render()` にタグ管理モーダルの再描画呼び出しを追加する**

`bookmarks-filesync.html:1605` は現在:

```js
    if (!groupManageOverlay.hidden) renderGroupManageModal(); // モーダルが開いている間は常に最新の並び・件数を反映する
```

この直後に、Task 1 Step 6 と全く同じ行を追加する:

```js
    if (!tagManageOverlay.hidden) renderTagManageModal(); // モーダルが開いている間は常に最新の一覧を反映する
```

- [ ] **Step 7: タグ管理モーダルのJS一式を追加する**

`bookmarks-filesync.html:2106-2118` は現在:

```js
  function bindGroupManageRowDragEvents() {
    groupManageList.querySelectorAll('.group-manage-row[draggable="true"]').forEach(function (row) {
      bindDragReorder(row, row.getAttribute('data-group'), {
        getDragged: function () { return manageDraggedGroup; },
        setDragged: function (v) { manageDraggedGroup = v; },
        payload: 'group', // 値自体は未使用(dropはmanageDraggedGroupを参照)。URL状のグループ名がドロップ失敗時に誤ナビゲーションを起こさないよう固定文字列にする
        onDrop: function (draggedKey, targetKey, after) {
          moveGroupSection(draggedKey, targetKey, after); // render()経由でrenderGroupManageModal()も再描画される
        }
      });
    });
  }
```

この関数の直後（`/* ---------- ドラッグ&ドロップ(並べ替え・グループ間移動) ---------- */` コメントの手前）に、Task 1 Step 7 と全く同じJSブロックを追加する:

```js

  /* ---------- タグ管理モーダル ---------- */

  var tagManageOverlay = document.getElementById('tag-manage-overlay');
  var tagManageList = document.getElementById('tag-manage-list');
  var tagManageEmpty = document.getElementById('tag-manage-empty');

  var tagManageOpenBtn = document.getElementById('tag-manage-open-btn');
  tagManageOpenBtn.addEventListener('click', openTagManageModal);
  document.getElementById('tag-manage-close-btn').addEventListener('click', closeTagManageModal);
  tagManageOverlay.addEventListener('click', function (e) {
    if (e.target === tagManageOverlay) closeTagManageModal(); // 背景クリックで閉じる
  });

  function openTagManageModal() {
    tagManageOverlay.hidden = false;
    renderTagManageModal();
    var firstRow = tagManageList.querySelector('.tag-manage-row');
    (firstRow || document.getElementById('tag-manage-close-btn')).focus();
  }

  function closeTagManageModal() {
    tagManageOverlay.hidden = true;
    state.editingTag = null; // 閉じたら行内のリネームフォームも一緒に閉じておく
    tagManageOpenBtn.focus(); // 開いた場所へフォーカスを戻す
  }

  // タグ管理モーダルのリスト全体を再描画する。render()から、モーダルが開いている間だけ呼ばれる。
  function renderTagManageModal() {
    var names = getTags();
    tagManageEmpty.hidden = names.length !== 0;
    tagManageList.innerHTML = names.map(function (name) {
      return renderTagManageRow(name);
    }).join('');
    bindTagManageRowButtons();
  }

  function renderTagManageRow(name) {
    if (state.editingTag === name) {
      return '<li class="tag-manage-row" data-tag="' + escapeHtml(name) + '">' +
        '<form class="tag-rename-form" data-tag-rename-form data-tag="' + escapeHtml(name) + '">' +
          '<input type="text" class="tag-rename-input" name="tagName" value="' + escapeHtml(name) + '" required>' +
          '<button type="submit" class="btn btn-amber">保存</button>' +
          '<button type="button" class="btn-quiet" data-action="cancel-tag-rename">キャンセル</button>' +
        '</form>' +
      '</li>';
    }
    return '<li class="tag-manage-row" tabindex="-1" data-tag="' + escapeHtml(name) + '">' +
      '<span class="tag-manage-name">' + escapeHtml(name) + '</span>' +
      '<button type="button" class="tag-edit-btn" data-action="edit-tag" data-tag="' + escapeHtml(name) + '" aria-label="' + escapeHtml(name) + 'の名前を変更">&#9998;</button>' +
      '<button type="button" class="tag-delete-btn" data-action="delete-tag" data-tag="' + escapeHtml(name) + '" aria-label="' + escapeHtml(name) + 'を削除">&#10005;</button>' +
    '</li>';
  }

  // 行の鉛筆(リネーム開始)・削除ボタン・リネームフォームのsubmit/キャンセルを割り当てる
  function bindTagManageRowButtons() {
    tagManageList.querySelectorAll('[data-action="edit-tag"]').forEach(function (btn) {
      btn.addEventListener('click', function () { state.editingTag = btn.getAttribute('data-tag'); renderTagManageModal(); });
    });
    tagManageList.querySelectorAll('[data-action="cancel-tag-rename"]').forEach(function (btn) {
      btn.addEventListener('click', function () { state.editingTag = null; renderTagManageModal(); });
    });
    tagManageList.querySelectorAll('[data-action="delete-tag"]').forEach(function (btn) {
      btn.addEventListener('click', function () { deleteTag(btn.getAttribute('data-tag')); });
    });
    tagManageList.querySelectorAll('[data-tag-rename-form]').forEach(function (form) {
      form.addEventListener('submit', function (e) {
        e.preventDefault();
        var oldName = form.getAttribute('data-tag');
        var newName = form.tagName.value.trim();
        if (newName && newName !== oldName) {
          bookmarks.forEach(function (b) {
            var tags = b.tags || [];
            if (tags.indexOf(oldName) === -1) return;
            var renamed = tags.map(function (t) { return t === oldName ? newName : t; });
            b.tags = renamed.filter(function (t, i) { return renamed.indexOf(t) === i; }); // 同一ブックマーク内での重複除去(改名先が既存の別タグと衝突した場合)
          });
          saveData(bookmarks);
          if (state.tag === oldName) state.tag = newName;
        }
        state.editingTag = null;
        render(); // カードのタグ表示・タグナビも変わるため、モーダルだけでなく全体を再描画する
      });
    });
  }

  // 指定タグを持つ全ブックマークから、そのタグだけを取り除く(ブックマーク自体は削除しない)
  function deleteTag(name) {
    var count = bookmarks.filter(function (b) { return (b.tags || []).indexOf(name) !== -1; }).length;
    if (count === 0) return;
    if (!confirm('タグ「' + name + '」を全てのブックマーク(' + count + '件)から削除しますか?')) return;
    bookmarks.forEach(function (b) {
      if (!b.tags) return;
      b.tags = b.tags.filter(function (t) { return t !== name; });
    });
    saveData(bookmarks);
    render();
  }
```

- [ ] **Step 8: `closeTopmostLayer()` にタグ管理モーダルの分岐を追加する**

`bookmarks-filesync.html:2564-2573` は現在:

```js
    // グループ管理モーダルが開いている場合、内側のリネームフォーム→モーダル本体の順に1段階ずつ閉じる
    if (!groupManageOverlay.hidden) {
      if (state.editingGroup !== null) {
        state.editingGroup = null;
        renderGroupManageModal();
        return;
      }
      closeGroupManageModal();
      return;
    }
```

この直後に、Task 1 Step 8 と全く同じブロックを追加する:

```js

    // タグ管理モーダルが開いている場合も同様に、内側のリネームフォーム→モーダル本体の順に1段階ずつ閉じる
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

- [ ] **Step 9: グローバルショートカットガードにタグ管理モーダルを追加する**

`bookmarks-filesync.html:2685` は現在:

```js
    if (!groupManageOverlay.hidden) return; // モーダル表示中はEscape以外のグローバルショートカットを無効化する
```

この直後に、Task 1 Step 9 と全く同じ行を追加する:

```js
    if (!tagManageOverlay.hidden) return; // モーダル表示中はEscape以外のグローバルショートカットを無効化する
```

- [ ] **Step 10: `bookmarks-filesync.html` を手動確認する**

`bookmarks-filesync.html` は Chrome または Edge で開くこと（File System Access API 非対応ブラウザでは「未対応」画面が出るため、確認自体ができない）。

1. Chrome/Edge で `bookmarks-filesync.html` を開き、ファイル選択ダイアログでJSONファイルを読み込む（既存のブックマークJSONでも、空の `[]` を保存した新規ファイルでもよい）。
2. Task 1 Step 10 の 1〜8 と同じ確認項目をすべて実施する。

ヘッドレス環境等でブラウザが使えない場合は、Task 1 Step 10 と同様の静的確認（`grep -n "tag-manage" bookmarks-filesync.html` で全識別子の対応を確認、識別子の宣言順序を目視確認）を行い、未実施の項目をレポートに明記する。

- [ ] **Step 11: 2ファイルの対称性を確認する**

次のコマンドで、両ファイルに追加したタグ管理関連の識別子の出現回数が一致することを確認する（識別子ごとの出現回数が同じであれば、両ファイルに同じ量・同じ構造のコードが追加されていることの強い裏付けになる）:

```bash
for id in "tag-manage-open-btn" "tag-manage-list" "tag-manage-empty" "tag-manage-row" "tag-manage-name" "tag-edit-btn" "tag-delete-btn" "tag-rename-form" "tag-rename-input" "editingTag" "tagManageOverlay" "tagManageList" "tagManageEmpty" "tagManageOpenBtn" "openTagManageModal" "closeTagManageModal" "renderTagManageModal" "renderTagManageRow" "bindTagManageRowButtons" "deleteTag"; do
  a=$(grep -c -- "$id" bookmarks.html)
  b=$(grep -c -- "$id" bookmarks-filesync.html)
  if [ "$a" != "$b" ]; then echo "MISMATCH: $id (bookmarks.html=$a, bookmarks-filesync.html=$b)"; fi
done
echo "checked"
```

`checked` の前に `MISMATCH` 行が1つも出力されない場合は対称性OKとみなす。`MISMATCH` が出た場合は、意図的な差分でない限り、`bookmarks-filesync.html` 側を `bookmarks.html` に合わせて修正すること。

- [ ] **Step 12: Commit**

```bash
git add bookmarks-filesync.html
git commit -m "feat: add tag management modal (rename, delete) to bookmarks-filesync.html"
```
