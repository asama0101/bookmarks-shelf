# サイドバーのグループタブをデフォルト表示にする Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** サイドバーの初期表示タブを「タグ」から「グループ」に変更する（`bookmarks.html` / `bookmarks-filesync.html` の両方）。

**Architecture:** 変更はJSの状態初期値1行とHTMLの初期属性4箇所のみ。`switchSidebarTab()` 等の既存ロジックには一切手を入れない。両ファイルとも同じ構造(同じ行内容・近い行番号)を持つため、同一パターンの変更を2ファイルへ適用する。

**Tech Stack:** 素のHTML/CSS/JS（ビルドなし、フレームワークなし）。自動テストは存在しないため、ブラウザでの手動確認で検証する。

## Global Constraints

- 対象ファイルは `bookmarks.html` と `bookmarks-filesync.html` の2つのみ。他ファイルは変更しない。
- `switchSidebarTab()` のロジック、キーボードショートカット（`t`/`g`）の挙動、`state.tag`/`state.group` フィルタの独立性は変更しない。
- タブ選択状態の永続化（localStorage・ファイル同期データへの保存）は追加しない。
- 自動テストは存在しないプロジェクトのため、各ステップの「テスト」はブラウザでの手動確認に置き換える。

---

### Task 1: 両ファイルのグループタブをデフォルト表示にする

**Files:**
- Modify: `bookmarks.html:999-1000`（タブボタンの `aria-selected`）
- Modify: `bookmarks.html:1004`, `bookmarks.html:1008`（パネルの `hidden` 属性）
- Modify: `bookmarks.html:1060`（`state.sidebarTab` 初期値）
- Modify: `bookmarks-filesync.html:1099-1100`（タブボタンの `aria-selected`）
- Modify: `bookmarks-filesync.html:1104`, `bookmarks-filesync.html:1108`（パネルの `hidden` 属性）
- Modify: `bookmarks-filesync.html:1159`（`state.sidebarTab` 初期値）

**Interfaces:**
- Consumes: なし（既存の `switchSidebarTab()`、`state` オブジェクトの構造をそのまま利用）
- Produces: なし（他タスクからの依存なし。本プランは単一タスク）

- [ ] **Step 1: `bookmarks.html` のタブボタン初期属性を入れ替える**

`bookmarks.html:999-1000` を次のように変更する。

変更前:
```html
      <button type="button" class="sidebar-tab" id="sidebar-tab-tag" role="tab" aria-selected="true" aria-controls="tag-nav-panel">タグ</button>
      <button type="button" class="sidebar-tab" id="sidebar-tab-group" role="tab" aria-selected="false" aria-controls="group-nav-panel">グループ</button>
```

変更後:
```html
      <button type="button" class="sidebar-tab" id="sidebar-tab-tag" role="tab" aria-selected="false" aria-controls="tag-nav-panel">タグ</button>
      <button type="button" class="sidebar-tab" id="sidebar-tab-group" role="tab" aria-selected="true" aria-controls="group-nav-panel">グループ</button>
```

- [ ] **Step 2: `bookmarks.html` のパネル初期表示を入れ替える**

`bookmarks.html:1004` を次のように変更する。

変更前:
```html
    <div class="sidebar-panel" id="tag-nav-panel" role="tabpanel">
```

変更後:
```html
    <div class="sidebar-panel" id="tag-nav-panel" role="tabpanel" hidden>
```

`bookmarks.html:1008` を次のように変更する。

変更前:
```html
    <div class="sidebar-panel" id="group-nav-panel" role="tabpanel" hidden>
```

変更後:
```html
    <div class="sidebar-panel" id="group-nav-panel" role="tabpanel">
```

- [ ] **Step 3: `bookmarks.html` の `state.sidebarTab` 初期値を変更する**

`bookmarks.html:1060` を次のように変更する。

変更前:
```js
    sidebarTab: 'tag'
```

変更後:
```js
    sidebarTab: 'group'
```

- [ ] **Step 4: `bookmarks.html` を手動確認する**

1. ブラウザで `bookmarks.html` を開く（既存の `bookmarks_data` が残っていても、サイドバータブの初期表示はデータに依存しないためそのままでよい）。
2. 「グループ」タブが選択状態（下線・`aria-selected="true"`）で、グループ一覧（グループがなければ空でもよい）が表示され、「タグ」タブの一覧は非表示であることを目視確認する。
3. 「タグ」タブをクリックし、タグ一覧に切り替わることを確認する。
4. 「グループ」タブをクリックして戻し、グループ一覧に戻ることを確認する。
5. キーボードショートカット `t` を押し、タグタブに切り替わりタグ一覧の先頭ボタンにフォーカスが移ることを確認する。
6. キーボードショートカット `g` を押し、グループタブに切り替わりグループ一覧の先頭ボタンにフォーカスが移ることを確認する（グループが1件もない場合はフォーカス移動が発生しないことを確認する）。

Expected: 上記すべてが記載通りに動作する。

- [ ] **Step 5: `bookmarks-filesync.html` のタブボタン初期属性を入れ替える**

`bookmarks-filesync.html:1099-1100` を次のように変更する。

変更前:
```html
      <button type="button" class="sidebar-tab" id="sidebar-tab-tag" role="tab" aria-selected="true" aria-controls="tag-nav-panel">タグ</button>
      <button type="button" class="sidebar-tab" id="sidebar-tab-group" role="tab" aria-selected="false" aria-controls="group-nav-panel">グループ</button>
```

変更後:
```html
      <button type="button" class="sidebar-tab" id="sidebar-tab-tag" role="tab" aria-selected="false" aria-controls="tag-nav-panel">タグ</button>
      <button type="button" class="sidebar-tab" id="sidebar-tab-group" role="tab" aria-selected="true" aria-controls="group-nav-panel">グループ</button>
```

- [ ] **Step 6: `bookmarks-filesync.html` のパネル初期表示を入れ替える**

`bookmarks-filesync.html:1104` を次のように変更する。

変更前:
```html
    <div class="sidebar-panel" id="tag-nav-panel" role="tabpanel">
```

変更後:
```html
    <div class="sidebar-panel" id="tag-nav-panel" role="tabpanel" hidden>
```

`bookmarks-filesync.html:1108` を次のように変更する。

変更前:
```html
    <div class="sidebar-panel" id="group-nav-panel" role="tabpanel" hidden>
```

変更後:
```html
    <div class="sidebar-panel" id="group-nav-panel" role="tabpanel">
```

- [ ] **Step 7: `bookmarks-filesync.html` の `state.sidebarTab` 初期値を変更する**

`bookmarks-filesync.html:1159` を次のように変更する。

変更前:
```js
    sidebarTab: 'tag'
```

変更後:
```js
    sidebarTab: 'group'
```

- [ ] **Step 8: `bookmarks-filesync.html` を手動確認する**

`bookmarks-filesync.html` は Chrome または Edge で開くこと（File System Access API 非対応ブラウザでは「未対応」画面が出るため、確認自体ができない）。

1. Chrome/Edge で `bookmarks-filesync.html` を開く。ファイル選択ダイアログが出た場合は任意のJSONファイル（既存のブックマークJSONでも、空の `[]` を保存した新規ファイルでもよい）を選択して読み込む。
2. Step 4 と同じ確認項目（1〜6）をすべて実施する。

Expected: 上記すべてが記載通りに動作する。

- [ ] **Step 9: Commit**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "feat: default sidebar to the group tab on load"
```
