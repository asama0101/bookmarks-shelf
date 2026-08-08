# タグ/グループナビの矢印キー移動と検索欄Escクリア Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** タグ/グループナビ内で矢印キーによるボタン間移動を可能にし、検索欄フォーカス中かつ入力ありの状態でのEscキーで検索をクリアできるようにする。

**Architecture:** `bookmarks.html` に4つの変更を加える（新規ヘルパー関数 `moveNavFocus()`、`handleGlobalShortcut()` の矢印キー分岐への1条件追加、既存の検索クリア処理を `clearSearch()` として関数化、Escapeディスパッチャへの1条件追加、ヘルプパネルの文言更新）。同一内容を `bookmarks-filesync.html` に移植する。

**Tech Stack:** Vanilla JS (ES5-style, IIFE), 単一HTMLファイル、ビルドなし。

## Global Constraints

- 対象ファイルは `bookmarks.html` と `bookmarks-filesync.html` の両方。挙動は完全に同一にする（`bookmarks-filesync.html` 固有のガード追加は不要 — この機能はファイル未接続状態に影響されない）。
- 矢印キー移動: フォーカスが `#tag-nav` または `#group-nav` 内のボタンにある間、↓→で次のボタン、↑←で前のボタンへ移動する。リストの端（最初/最後のボタン）では何もしない（`moveCardFocus()` と同じ「無ければ何もしない」規約）。
- Escクリア: `#search-input` にフォーカスがあり、かつ `state.search` が非空の場合のみ、Escキーで検索をクリアし検索欄にフォーカスを残す。空の場合は既存の `closeTopmostLayer()` チェーンにフォールスルーする。
- 新規関数名は `moveNavFocus(btn, direction)` と `clearSearch()` を使う（他の名前を使わない）。
- モーダル表示中の追加ガードは不要（`handleGlobalShortcut()` 呼び出し自体が既存ガードで抑止されるため）。
- 既存のショートカット割り当て・`closeTopmostLayer()`・`handleCardShortcut()`・カード側の矢印キー移動ロジックは一切変更しない。

---

### Task 1: `bookmarks.html` — ナビ矢印キー移動とEscクリアの実装

**Files:**
- Modify: `bookmarks.html:2398-2417`（検索欄セットアップ）
- Modify: `bookmarks.html:2578-2587`（`handleGlobalShortcut()` の矢印キー分岐）
- Modify: `bookmarks.html:2650-2653`（`keydown` ディスパッチャのEscape分岐）
- Modify: `bookmarks.html:1074`（ヘルプパネルのEsc説明行）

**Interfaces:**
- Consumes: 既存の `state`, `render()`, `searchInput`, `searchClearBtn`（すべて同ファイル内で本タスクの変更箇所より前に既に宣言済み）
- Produces: `moveNavFocus(btn, direction)` — タグ/グループナビ内のボタン間移動ヘルパー。`clearSearch()` — 検索クリア処理（×ボタンとEscの両方から呼ばれる）。どちらも Task 2 でファイル間の対応関係を確認する対象になる。

- [ ] **Step 1: `moveNavFocus()` ヘルパー関数を追加する**

`bookmarks.html:2578` の直前（`handleGlobalShortcut()` 関数の直前）に以下を追加する:

```js
  // タグ/グループナビ内でのボタン間移動(次/前が無ければ何もしない。moveCardFocusと同じ規約)
  function moveNavFocus(btn, direction) {
    var list = btn.closest('ul');
    var buttons = Array.prototype.slice.call(list.querySelectorAll('button'));
    var next = buttons[buttons.indexOf(btn) + direction];
    if (next) next.focus();
  }

```

- [ ] **Step 2: `handleGlobalShortcut()` の矢印キー分岐にナビ判定を追加する**

`bookmarks.html:2578-2587` の現在の内容:

```js
      case 'ArrowDown':
      case 'ArrowRight':
      case 'ArrowUp':
      case 'ArrowLeft':
        // 既にカードにフォーカス中なら、この関数では処理せずhandleCardShortcut側の移動処理に任せる
        if (e.target.closest('#group-sections .card[tabindex]')) return false;
        e.preventDefault();
        var arrowTarget = document.querySelector('#group-sections .card[tabindex="0"]');
        if (arrowTarget) arrowTarget.focus();
        return true;
```

これを以下に置き換える:

```js
      case 'ArrowDown':
      case 'ArrowRight':
      case 'ArrowUp':
      case 'ArrowLeft':
        var navBtn = e.target.closest('#tag-nav button, #group-nav button');
        if (navBtn) {
          e.preventDefault();
          moveNavFocus(navBtn, (e.key === 'ArrowDown' || e.key === 'ArrowRight') ? 1 : -1);
          return true;
        }
        // 既にカードにフォーカス中なら、この関数では処理せずhandleCardShortcut側の移動処理に任せる
        if (e.target.closest('#group-sections .card[tabindex]')) return false;
        e.preventDefault();
        var arrowTarget = document.querySelector('#group-sections .card[tabindex="0"]');
        if (arrowTarget) arrowTarget.focus();
        return true;
```

- [ ] **Step 3: 検索欄セットアップを `clearSearch()` を使う形に書き換える**

`bookmarks.html:2398-2417` の現在の内容:

```js
  var searchInput = document.getElementById('search-input');
  var searchClearBtn = document.getElementById('search-clear-btn');

  function updateSearchClearVisibility() {
    searchClearBtn.hidden = !state.search;
  }

  searchInput.addEventListener('input', function (e) {
    state.search = e.target.value;
    updateSearchClearVisibility();
    render();
  });

  searchClearBtn.addEventListener('click', function () {
    state.search = '';
    searchInput.value = '';
    updateSearchClearVisibility();
    searchInput.focus();
    render();
  });
```

これを以下に置き換える:

```js
  var searchInput = document.getElementById('search-input');
  var searchClearBtn = document.getElementById('search-clear-btn');

  function updateSearchClearVisibility() {
    searchClearBtn.hidden = !state.search;
  }

  // 検索をクリアして検索欄へフォーカスを戻す。×ボタンのクリック・検索欄フォーカス中のEscの両方から呼ばれる。
  function clearSearch() {
    state.search = '';
    searchInput.value = '';
    updateSearchClearVisibility();
    searchInput.focus();
    render();
  }

  searchInput.addEventListener('input', function (e) {
    state.search = e.target.value;
    updateSearchClearVisibility();
    render();
  });

  searchClearBtn.addEventListener('click', clearSearch);
```

- [ ] **Step 4: `keydown` ディスパッチャのEscape分岐に検索クリア優先処理を追加する**

`bookmarks.html:2650-2653` の現在の内容:

```js
    if (e.key === 'Escape') {
      closeTopmostLayer();
      return;
    }
```

これを以下に置き換える:

```js
    if (e.key === 'Escape') {
      if (e.target === searchInput && state.search) {
        clearSearch();
        return;
      }
      closeTopmostLayer();
      return;
    }
```

- [ ] **Step 5: ヘルプパネルのEsc説明行を更新する**

`bookmarks.html:1074` の現在の内容:

```html
    <dt><kbd>Esc</kbd></dt><dd>ヘルプ &rarr; グループ管理 &rarr; タグ管理 &rarr; フォーム &rarr; 選択モードの順に閉じる</dd>
```

これを以下に置き換える:

```html
    <dt><kbd>Esc</kbd></dt><dd>検索欄にフォーカス中かつ入力中なら検索をクリア。それ以外はヘルプ &rarr; グループ管理 &rarr; タグ管理 &rarr; フォーム &rarr; 選択モードの順に閉じる</dd>
```

- [ ] **Step 6: ブラウザで動作確認する**

`xdg-open bookmarks.html` で開き、以下を確認する:

1. `t` でタグタブに切り替え、`↓`/`→` でタグナビ内を次のタグへ、`↑`/`←` で前のタグへ移動できる。リストの端で矢印キーを押しても何も起きない（カード一覧に飛ばない）。
2. `g` でグループタブに切り替え、同様にグループナビ内を矢印キーで移動できる。
3. 検索欄に何か入力し、フォーカスしたまま Esc を押すと、検索がクリアされ、検索欄にフォーカスが残り、一覧の絞り込みも解除される。
4. 検索欄が空の状態でフォーカスしたまま Esc を押すと、これまで通り `closeTopmostLayer()` の挙動になる（例えばヘルプが開いていれば閉じる）。
5. `?` キーでヘルプパネルを開き、Escの説明文が更新されていることを確認する。
6. カード一覧での矢印キー移動・`Home`/`End`・`Enter`・`e`・`c`・`p`・`Delete`/`Backspace`・`/`・`n` が、これまで通り動作することを確認する（回帰確認）。

- [ ] **Step 7: コミットする**

```bash
git add bookmarks.html
git commit -m "feat: add tag/group nav arrow-key movement and search Esc-clear to bookmarks.html"
```

---

### Task 2: `bookmarks-filesync.html` — 同内容の移植と対称性確認

**Files:**
- Modify: `bookmarks-filesync.html:2702-2721`（検索欄セットアップ、Task 1 の `bookmarks.html:2398-2417` に相当）
- Modify: `bookmarks-filesync.html:2882-2891`（`handleGlobalShortcut()` の矢印キー分岐、Task 1 の `bookmarks.html:2578-2587` に相当）
- Modify: `bookmarks-filesync.html:2954-2957`（`keydown` ディスパッチャのEscape分岐、Task 1 の `bookmarks.html:2650-2653` に相当）
- Modify: `bookmarks-filesync.html:1174`（ヘルプパネルのEsc説明行、Task 1 の `bookmarks.html:1074` に相当）

**Interfaces:**
- Consumes: Task 1 で確定した `moveNavFocus(btn, direction)` と `clearSearch()` の実装内容（コードは同一、行番号のみ異なる）。`bookmarks-filesync.html` 側の既存 `searchInput`/`searchClearBtn`/`state`/`render()` を使う。
- Produces: `bookmarks-filesync.html` 内の `moveNavFocus()`・`clearSearch()`（`bookmarks.html` と同名・同内容）。

- [ ] **Step 1: 現在の行番号を確認する**

`bookmarks.html` の変更中に行番号がずれている可能性があるため、着手前に以下を実行して実際の行番号を確認する:

```bash
grep -n "function handleGlobalShortcut\|case 'ArrowDown'\|var searchInput\|var searchClearBtn\|searchClearBtn.addEventListener\|e.key === 'Escape'\|Esc</kbd" bookmarks-filesync.html
```

- [ ] **Step 2: `moveNavFocus()` ヘルパー関数を追加する**

`handleGlobalShortcut()` 関数の直前に、Task 1 Step 1 と全く同じ内容を追加する:

```js
  // タグ/グループナビ内でのボタン間移動(次/前が無ければ何もしない。moveCardFocusと同じ規約)
  function moveNavFocus(btn, direction) {
    var list = btn.closest('ul');
    var buttons = Array.prototype.slice.call(list.querySelectorAll('button'));
    var next = buttons[buttons.indexOf(btn) + direction];
    if (next) next.focus();
  }

```

- [ ] **Step 3: `handleGlobalShortcut()` の矢印キー分岐にナビ判定を追加する**

現在の内容（`bookmarks.html` の変更前と同一パターン）:

```js
      case 'ArrowDown':
      case 'ArrowRight':
      case 'ArrowUp':
      case 'ArrowLeft':
        // 既にカードにフォーカス中なら、この関数では処理せずhandleCardShortcut側の移動処理に任せる
        if (e.target.closest('#group-sections .card[tabindex]')) return false;
        e.preventDefault();
        var arrowTarget = document.querySelector('#group-sections .card[tabindex="0"]');
        if (arrowTarget) arrowTarget.focus();
        return true;
```

これを以下に置き換える:

```js
      case 'ArrowDown':
      case 'ArrowRight':
      case 'ArrowUp':
      case 'ArrowLeft':
        var navBtn = e.target.closest('#tag-nav button, #group-nav button');
        if (navBtn) {
          e.preventDefault();
          moveNavFocus(navBtn, (e.key === 'ArrowDown' || e.key === 'ArrowRight') ? 1 : -1);
          return true;
        }
        // 既にカードにフォーカス中なら、この関数では処理せずhandleCardShortcut側の移動処理に任せる
        if (e.target.closest('#group-sections .card[tabindex]')) return false;
        e.preventDefault();
        var arrowTarget = document.querySelector('#group-sections .card[tabindex="0"]');
        if (arrowTarget) arrowTarget.focus();
        return true;
```

- [ ] **Step 4: 検索欄セットアップを `clearSearch()` を使う形に書き換える**

現在の内容（`bookmarks.html` の変更前と同一パターン）:

```js
  var searchInput = document.getElementById('search-input');
  var searchClearBtn = document.getElementById('search-clear-btn');

  function updateSearchClearVisibility() {
    searchClearBtn.hidden = !state.search;
  }

  searchInput.addEventListener('input', function (e) {
    state.search = e.target.value;
    updateSearchClearVisibility();
    render();
  });

  searchClearBtn.addEventListener('click', function () {
    state.search = '';
    searchInput.value = '';
    updateSearchClearVisibility();
    searchInput.focus();
    render();
  });
```

これを以下に置き換える:

```js
  var searchInput = document.getElementById('search-input');
  var searchClearBtn = document.getElementById('search-clear-btn');

  function updateSearchClearVisibility() {
    searchClearBtn.hidden = !state.search;
  }

  // 検索をクリアして検索欄へフォーカスを戻す。×ボタンのクリック・検索欄フォーカス中のEscの両方から呼ばれる。
  function clearSearch() {
    state.search = '';
    searchInput.value = '';
    updateSearchClearVisibility();
    searchInput.focus();
    render();
  }

  searchInput.addEventListener('input', function (e) {
    state.search = e.target.value;
    updateSearchClearVisibility();
    render();
  });

  searchClearBtn.addEventListener('click', clearSearch);
```

- [ ] **Step 5: `keydown` ディスパッチャのEscape分岐に検索クリア優先処理を追加する**

現在の内容（`bookmarks.html` の変更前と同一パターン）:

```js
    if (e.key === 'Escape') {
      closeTopmostLayer();
      return;
    }
```

これを以下に置き換える:

```js
    if (e.key === 'Escape') {
      if (e.target === searchInput && state.search) {
        clearSearch();
        return;
      }
      closeTopmostLayer();
      return;
    }
```

- [ ] **Step 6: ヘルプパネルのEsc説明行を更新する**

現在の内容:

```html
    <dt><kbd>Esc</kbd></dt><dd>ヘルプ &rarr; グループ管理 &rarr; タグ管理 &rarr; フォーム &rarr; 選択モードの順に閉じる</dd>
```

これを以下に置き換える:

```html
    <dt><kbd>Esc</kbd></dt><dd>検索欄にフォーカス中かつ入力中なら検索をクリア。それ以外はヘルプ &rarr; グループ管理 &rarr; タグ管理 &rarr; フォーム &rarr; 選択モードの順に閉じる</dd>
```

- [ ] **Step 7: 対称性を grep で検証する**

`bookmarks.html` と `bookmarks-filesync.html` の該当識別子の出現回数が一致することを確認する:

```bash
for name in moveNavFocus clearSearch navBtn; do
  echo "== $name =="
  grep -c "$name" bookmarks.html
  grep -c "$name" bookmarks-filesync.html
done
```

Expected: 各識別子について両ファイルで同じ回数が出力される。

- [ ] **Step 8: ブラウザで動作確認する**

`xdg-open bookmarks-filesync.html` で開き（Chrome/Edge必須）、ファイルを接続した上で Task 1 Step 6 と同じ6項目を確認する。加えて、ファイル未接続の状態でもこれらのショートカット自体は無効化する既存の仕組みに影響が出ていないこと（矢印キー・Escの挙動が接続前後で壊れていないこと）を確認する。

- [ ] **Step 9: コミットする**

```bash
git add bookmarks-filesync.html
git commit -m "feat: add tag/group nav arrow-key movement and search Esc-clear to bookmarks-filesync.html"
```
