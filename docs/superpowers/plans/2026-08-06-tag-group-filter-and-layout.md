# タグ/グループの使い分け明確化とレイアウト固定 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **重要:** このプロジェクトには自動テストランナーが存在しない。各タスクのRED/GREENは`tdd-gates`の`browser-manual-e2e`プロファイル（`~/.claude/skills/tdd-gates/references/profiles/browser-manual-e2e.md`）に従い、claude-in-chromeで実ブラウザを操作してDOM状態をJS評価で確認する。

**Goal:** bookmark-shelf（`bookmarks.html` / `bookmarks-filesync.html`）で、タグとグループのフィルタ状態・入力方法を視覚的に区別できるようにし、ヘッダー/サイドバーをスクロール固定し、カードの冗長なURL表示を削除する。

**Architecture:** 既存の「文字列連結によるHTML生成＋`render()`での全体再描画」設計を維持しつつ、タグ・グループ選択に既存カラー（ワイン＝タグ、セージ＝グループ）を再利用して一貫させる。新規追加フォーム・カードのインライン編集フォーム・一括グループ化バーの3箇所で使うタグ/グループの「チップ選択UI」は、`renderTagChipList()`/`renderGroupChipList()`という共通関数に切り出し、各フォームは自分のコンテナ要素とローカルな選択状態（Set/変数）を渡すだけで再利用する。

**Tech Stack:** Vanilla ES5-style JS（IIFE）、素のDOM API。ビルド・パッケージマネージャ・依存追加は無し（既存方針を維持）。

## Global Constraints

- 両ファイル（`bookmarks.html` / `bookmarks-filesync.html`）へ同一パターンを適用する（CLAUDE.mdの方針）。現時点の実測オフセットは、スクリプト部（`getSearchTagFiltered()`以降の関数群）が**+302行**、CSS部が**+75〜+76行**、body部（`<section class="add-panel">`以降、header内部を除く）が**+100行**。`bookmarks-filesync.html`の`<header>`〜`<section class="add-panel">`の間だけは、ファイル未接続時のUI（`file-status`、`disconnect-btn`、`unsupported-screen`、`connect-screen`、各ボタンの`disabled`属性）が追加されているため単純オフセットが通用しない。該当箇所は行番号ではなくアンカー文字列（例: `</header>`直前、`id="selection-bar"`直後）で挿入位置を特定すること。
- 自動テストランナーは存在しない。全シナリオはclaude-in-chromeによる手動E2Eで検証する（`browser-manual-e2e`プロファイル）。
- 対象spec: `docs/superpowers/specs/2026-08-06-tag-group-filter-and-layout-design.md`（要件ID: R1〜R6）。
- データモデル（`tags[]` / `group` / `groupOrder`）は変更しない。`sanitizeBookmark()`・インポート・ドラッグ&ドロップ・グループ管理モーダルのロジックは対象外。

## 共通テストデータ・アサーション（全タスク共通）

`bookmarks.html`はサニタイズを通さずlocalStorageを素通しするため、以下でシード可能:

```js
localStorage.setItem('bookmarks_data', JSON.stringify([
  { id: 'k1', url: 'https://example.com/a', title: 'Alpha', tags: ['work', 'urgent'], group: '仕事', groupOrder: 0,
    loginId: 'u@example.com', loginPassword: 'pw', memo: null, scheme: 'https', domain: 'example.com', order: 0 },
  { id: 'k2', url: 'https://example.com/b', title: 'Bravo', tags: ['ref'], group: '仕事', groupOrder: 0,
    loginId: null, loginPassword: null, memo: null, scheme: 'https', domain: 'example.com', order: 1 },
  { id: 'k3', url: 'https://example.com/c', title: 'Charlie', tags: [], group: null, groupOrder: 0,
    loginId: null, loginPassword: null, memo: null, scheme: 'https', domain: 'example.com', order: 2 }
])); location.reload();
```

DOM順は「仕事区画（k1, k2）→ 未分類区画（k3）」。既存のタグは`work`/`urgent`/`ref`、既存のグループは`仕事`のみ。

各タスクのDone条件に共通で含めるもの:
1. `bookmarks.html`でのシナリオ全件GREEN
2. `bookmarks-filesync.html`への同一差分適用と該当リージョンの`diff`一致確認（`grep -n`でアンカーを特定→`sed -n`で範囲抽出→`diff`）
3. DevToolsコンソールにエラーが出ていないこと
4. `progress.md`にRED/GREENの実行結果（評価式と返り値）を証拠として追記

---

### Task 1: サイドバーnavの色分け（R1）

**充足要件:** R1

**Files:**
- Modify: `bookmarks.html:333-357`（`.sidebar li button.active`ルール）
- Modify: `bookmarks-filesync.html:409-433`（対応箇所、offset +76）

**Interfaces:**
- Consumes: なし
- Produces: なし（CSSのみ、他タスクからの依存なし）

- [ ] **Step 1: RED確認**

シード投入・リロード後、claude-in-chromeで以下を評価する。

```js
document.querySelector('#tag-nav button[data-value="work"]').click();
getComputedStyle(document.querySelector('#tag-nav button.active')).backgroundColor
```

期待するRED: `rgb(227, 167, 63)`（`--amber`。ワイン色`rgb(122, 46, 54)`になっていない）

- [ ] **Step 2: 実装**

`bookmarks.html:333-357`の既存ルールを以下に置き換える。

```css
  .sidebar li button {
    width: 100%;
    text-align: left;
    background: none;
    border: none;
    color: var(--ivory);
    font-family: var(--font-mono);
    font-size: 0.86rem;
    padding: 6px 8px;
    border-radius: 4px;
    display: flex;
    justify-content: space-between;
    gap: 6px;
  }
  .sidebar li button:hover { background: rgba(245, 241, 230, 0.08); }
  #tag-nav button.active {
    background: var(--wine);
    color: var(--ivory);
    font-weight: 700;
  }
  #group-nav button.active {
    background: var(--sage);
    color: var(--ink);
    font-weight: 700;
  }
  .sidebar li button .count {
    opacity: 0.7;
    font-size: 0.78rem;
  }
  .sidebar li button.active .count { opacity: 0.85; }
```

- [ ] **Step 3: GREEN確認**

```js
document.querySelector('#tag-nav button[data-value="work"]').click();
getComputedStyle(document.querySelector('#tag-nav button.active')).backgroundColor
```
期待するGREEN: `rgb(122, 46, 54)`（`--wine`）

```js
document.querySelector('#tag-nav button.active').click(); // 解除
document.querySelector('#group-nav button[data-value="仕事"]').click();
getComputedStyle(document.querySelector('#group-nav button.active')).backgroundColor
```
期待するGREEN: `rgb(183, 196, 180)`（`--sage`）

- [ ] **Step 4: bookmarks-filesync.htmlへの同一差分適用と一致確認**

```bash
diff <(sed -n '333,357p' bookmarks.html) <(sed -n '409,433p' bookmarks-filesync.html)
```
（空出力になることを確認）

- [ ] **Step 5: Commit**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "feat: color-separate tag/group sidebar nav active states"
```

---

### Task 2: カードのURL表示を削除（R6）

**充足要件:** R6

**Files:**
- Modify: `bookmarks.html:674-682`（`.card-url`のCSSルール削除）、`bookmarks.html:1682`（HTML行削除）
- Modify: `bookmarks-filesync.html`の対応箇所（CSS offset +76、JS offset +302）

**Interfaces:**
- Consumes: なし
- Produces: なし

- [ ] **Step 1: RED確認**

```js
!!document.querySelector('.card-url')
```
期待するRED: `true`

- [ ] **Step 2: 実装（CSS削除）**

`bookmarks.html:674-682`の以下のブロックを削除する（前後の空行は1行分残す）。

```css

  .card-url {
    font-family: var(--font-mono);
    font-size: 0.72rem;
    color: var(--ink-soft);
    overflow-wrap: anywhere;
    margin: 0;
  }
```

- [ ] **Step 3: 実装（HTML削除）**

`bookmarks.html:1682`の以下の行を削除する。

```js
              '<p class="card-url">' + escapeHtml(b.url) + '</p>' +
```

- [ ] **Step 4: GREEN確認**

```js
!!document.querySelector('.card-url')
```
期待するGREEN: `false`

```js
[document.querySelector('[data-card-id="k1"] .card-title a').href,
 document.querySelector('[data-card-id="k1"] .seal img, [data-card-id="k1"] .seal svg') !== null]
```
期待するGREEN: `["https://example.com/a", true]`（リンクとfavicon表示は影響を受けていないことの確認）

- [ ] **Step 5: bookmarks-filesync.htmlへの同一差分適用と一致確認**

- [ ] **Step 6: Commit**

```bash
git commit -m "feat: remove redundant URL text display from card"
```

---

### Task 3: 常時表示フィルタインジケーター（R2）

**充足要件:** R2

**Files:**
- Modify: `bookmarks.html:799`直後（HTML、`.header-actions`の`</div>`と`</header>`の間）
- Modify: `bookmarks.html:690-704`直後（CSS、`.tag-stamp`ブロックの後）
- Modify: `bookmarks.html`の`render()`関数（`bookmarks.html:1141-1166`）とキーボード操作より前のイベント登録セクション
- Modify: `bookmarks-filesync.html`の対応箇所（HTMLはアンカー文字列`</header>`で特定、CSS offset +76、JS offset +302）

**Interfaces:**
- Consumes: `state.tag`, `state.group`（既存）
- Produces: `renderFilterIndicator()`、`#filter-indicator`要素（他タスクからの依存なし）

- [ ] **Step 1: RED確認**

```js
document.querySelector('#tag-nav button[data-value="work"]').click();
!!document.getElementById('filter-indicator')
```
期待するRED: `false`（要素自体が存在しない）

- [ ] **Step 2: 実装（HTML）**

`bookmarks.html:799`（`.header-actions`の`</div>`）の直後、`</header>`（800行目）の前に追加する。

```html
  <div class="filter-indicator" id="filter-indicator"></div>
```

- [ ] **Step 3: 実装（CSS）**

`bookmarks.html:704`（`.tag-stamp.active { background: var(--wine); color: var(--ivory); }`）の直後に追加する。

```css

  .filter-indicator {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    flex-basis: 100%;
  }
  .filter-indicator:empty { display: none; }
  .filter-chip {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: var(--font-mono);
    font-size: 0.78rem;
    padding: 3px 10px;
    border-radius: 999px;
    background: var(--ivory);
  }
  .filter-chip.tag-filter-chip { color: var(--wine); border: 1px solid var(--wine); }
  .filter-chip.group-filter-chip { color: var(--ink); border: 1px solid var(--sage); }
  .filter-chip button {
    background: none;
    border: none;
    color: inherit;
    font-weight: 700;
    cursor: pointer;
    padding: 0;
    line-height: 1;
  }
```

- [ ] **Step 4: 実装（JS）**

`renderTagNav()`の直前（`bookmarks.html:1172`の後）に関数を追加する。

```js
  function renderFilterIndicator() {
    var el = document.getElementById('filter-indicator');
    var chips = '';
    if (state.tag !== null) {
      chips += '<span class="filter-chip tag-filter-chip">タグ: ' + escapeHtml(state.tag) +
        '<button type="button" data-action="clear-tag-filter" aria-label="タグ絞り込みを解除">&times;</button></span>';
    }
    if (state.group !== null) {
      var label = state.group === '' ? '未分類' : state.group;
      chips += '<span class="filter-chip group-filter-chip">グループ: ' + escapeHtml(label) +
        '<button type="button" data-action="clear-group-filter" aria-label="グループ絞り込みを解除">&times;</button></span>';
    }
    el.innerHTML = chips;
  }

  document.getElementById('filter-indicator').addEventListener('click', function (e) {
    var btn = e.target.closest('button');
    if (!btn) return;
    var action = btn.getAttribute('data-action');
    if (action === 'clear-tag-filter') state.tag = null;
    else if (action === 'clear-group-filter') state.group = null;
    else return;
    render();
  });
```

`render()`関数（`bookmarks.html:1155`付近、`renderTagNav();`の直後）に呼び出しを追加する。

```js
    renderTagNav();
    renderGroupNav();
    renderFilterIndicator();
    renderSelectionBar();
```

- [ ] **Step 5: GREEN確認**

```js
document.querySelector('#tag-nav button[data-value="work"]').click();
document.getElementById('filter-indicator').textContent
```
期待するGREEN: `"タグ: work×"`

```js
document.querySelector('#group-nav button[data-value="仕事"]').click();
[document.querySelectorAll('#filter-indicator .filter-chip').length,
 document.getElementById('filter-indicator').textContent.indexOf('グループ: 仕事') !== -1]
```
期待するGREEN: `[2, true]`（タグ・グループ両方のチップが同時表示）

```js
document.querySelector('[data-action="clear-tag-filter"]').click();
[state.tag, document.querySelectorAll('#filter-indicator .filter-chip').length]
```
（`state`はモジュールスコープのためconsole評価不可な場合は`document.querySelector('#tag-nav button.active')`が`null`であることで代替確認）
期待するGREEN: タグnavに`.active`が無い、チップが1個（グループのみ）に減る

```js
document.querySelector('[data-action="clear-group-filter"]').click();
document.getElementById('filter-indicator').children.length
```
期待するGREEN: `0`（`:empty`によりindicator自体も非表示）

- [ ] **Step 6: bookmarks-filesync.htmlへの同一差分適用と一致確認**

HTML挿入位置は`grep -n '</header>' bookmarks-filesync.html`でアンカーを特定してから追加する（filesync側は未接続時UIの分だけ行番号がずれるため）。CSS・JSはoffset通り。

- [ ] **Step 7: Commit**

```bash
git commit -m "feat: add persistent tag/group filter indicator"
```

---

### Task 4: チップピッカー共通基盤 + 新規追加フォームへの適用（R3の一部 + R4-1）

**充足要件:** R3（新規追加フォーム分）、R4（共通基盤＋新規追加フォーム）

**Files:**
- Modify: `bookmarks.html:820-828`（`#add-form`のタグ/グループ`.field`）
- Modify: `bookmarks.html:704`直後（CSS、Task3で追加した`.filter-indicator`系ブロックの後）
- Modify: `bookmarks.html:1172`直前（JS、共通チップピッカー関数を追加。Task3の`renderFilterIndicator()`より前に置く）
- Modify: `bookmarks.html`の`render()`関数、`/* ---------- 追加フォーム ---------- */`セクション（`bookmarks.html:1776-1831`）
- Modify: `bookmarks-filesync.html`の対応箇所（HTML offset +100、CSS offset +76、JS offset +302）

**Interfaces:**
- Consumes: `getTags()`（`bookmarks.html:1112-1116`）, `getGroups()`（`bookmarks.html:1103-1110`）, `parseTags()`（`bookmarks.html:1009-1011`）, `escapeHtml()`（既存）
- Produces: `chipButtonHtml(value, isActive, extraClass)`, `renderTagChipList(containerEl, selectedTags, onChange)`, `renderGroupChipList(containerEl, selectedGroupGetter, onSelect)` — Task5・Task6がこれらをそのまま再利用する。DOM ids: `new-tags-chips`, `new-tags-text`, `new-group-chips`, `new-group-text`。

**注記:** `<datalist id="group-list"></datalist>`は`#edit-group`（Task5）と`#group-name-input`（Task6）がまだ参照しているため、本タスクでは削除せず現在の位置（グループ`.field`内）にそのまま残す。削除はTask6で行う。

- [ ] **Step 1: RED確認**

```js
document.getElementById('toggle-add-btn').click();
[!!document.getElementById('new-tags-chips'), !!document.getElementById('new-tags')]
```
期待するRED: `[false, true]`（新UI未実装、旧`#new-tags`がまだ存在）

- [ ] **Step 2: 実装（CSS）**

Task3で追加した`.filter-chip button { ... }`ブロックの直後に追加する。

```css

  .chip-list {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 6px;
  }
  .chip {
    font-family: var(--font-mono);
    font-size: 0.78rem;
    padding: 3px 10px;
    border-radius: 999px;
    cursor: pointer;
    background: transparent;
  }
  .chip.tag-chip { color: var(--wine); border: 1px solid var(--wine); }
  .chip.tag-chip.active { background: var(--wine); color: var(--ivory); }
  .chip.group-chip { color: var(--ink); border: 1px solid var(--sage); }
  .chip.group-chip.active { background: var(--sage); color: var(--ink); }
  .chip-empty {
    font-size: 0.78rem;
    color: var(--ink-soft);
    margin: 0 0 6px;
  }
  .field-explain {
    font-size: 0.72rem;
    color: var(--ink-soft);
    margin: 0 0 6px;
  }
```

- [ ] **Step 3: 実装（HTML、`#add-form`）**

`bookmarks.html:820-828`を以下に置き換える。

```html
    <div class="field">
      <label>タグ</label>
      <p class="field-explain">タグ＝複数付けられる横断ラベル</p>
      <div class="chip-list" id="new-tags-chips"></div>
      <input id="new-tags-text" type="text" placeholder="新しいタグ(カンマ区切りで複数可)">
    </div>
    <div class="field">
      <label>グループ</label>
      <p class="field-explain">グループ＝1つだけ選べる置き場所</p>
      <div class="chip-list" id="new-group-chips"></div>
      <input id="new-group-text" type="text" placeholder="新しいグループ名">
      <datalist id="group-list"></datalist>
    </div>
```

- [ ] **Step 4: 実装（JS、共通チップピッカー関数）**

`renderTagNav()`の直前（Task3で追加した`renderFilterIndicator()`のさらに前）に追加する。

```js
  /* ---------- タグ/グループ チップピッカー(共通) ---------- */

  function chipButtonHtml(value, isActive, extraClass) {
    return '<button type="button" class="chip ' + extraClass + (isActive ? ' active' : '') + '" data-value="' + escapeHtml(value) + '">' + escapeHtml(value) + '</button>';
  }

  // タグは複数選択。selectedTagsはSet<string>で、クリックのたびにトグルしてonChangeを呼ぶ。
  function renderTagChipList(containerEl, selectedTags, onChange) {
    var tags = getTags();
    containerEl.innerHTML = tags.length
      ? tags.map(function (t) { return chipButtonHtml(t, selectedTags.has(t), 'tag-chip'); }).join('')
      : '<p class="chip-empty">タグがまだありません</p>';
    containerEl.querySelectorAll('button').forEach(function (btn) {
      btn.addEventListener('click', function () {
        var val = btn.getAttribute('data-value');
        if (selectedTags.has(val)) selectedTags.delete(val); else selectedTags.add(val);
        onChange();
      });
    });
  }

  // グループは単一選択。selectedGroupGetterは現在値を返す関数、onSelectは新しい値(選び直しでの解除=null含む)を受け取る。
  function renderGroupChipList(containerEl, selectedGroupGetter, onSelect) {
    var groups = getGroups();
    var current = selectedGroupGetter();
    containerEl.innerHTML = groups.length
      ? groups.map(function (g) { return chipButtonHtml(g, current === g, 'group-chip'); }).join('')
      : '<p class="chip-empty">グループがまだありません</p>';
    containerEl.querySelectorAll('button').forEach(function (btn) {
      btn.addEventListener('click', function () {
        var val = btn.getAttribute('data-value');
        onSelect(current === val ? null : val);
      });
    });
  }
```

- [ ] **Step 5: 実装（JS、追加フォームの結線）**

`/* ---------- 追加フォーム ---------- */`セクション（`bookmarks.html:1776-1831`）を以下に置き換える。

```js
  /* ---------- 追加フォーム ---------- */

  var addPanel = document.getElementById('add-panel');
  var addForm = document.getElementById('add-form');
  var newUrlInput = document.getElementById('new-url');
  var newTitleInput = document.getElementById('new-title');
  var newTagsChipsEl = document.getElementById('new-tags-chips');
  var newTagsTextInput = document.getElementById('new-tags-text');
  var newGroupChipsEl = document.getElementById('new-group-chips');
  var newGroupTextInput = document.getElementById('new-group-text');
  var addFormSelectedTags = new Set();
  var addFormSelectedGroup = null;

  function refreshAddFormChipPickers() {
    renderTagChipList(newTagsChipsEl, addFormSelectedTags, refreshAddFormChipPickers);
    renderGroupChipList(newGroupChipsEl, function () { return addFormSelectedGroup; }, function (val) {
      addFormSelectedGroup = val;
      if (val !== null) newGroupTextInput.value = '';
      refreshAddFormChipPickers();
    });
  }

  // 新規グループ欄に文字を入力したら、既存チップの単一選択は自動解除する(グループは単一値であるという制約をUI上でも保つ)
  newGroupTextInput.addEventListener('input', function () {
    if (newGroupTextInput.value.trim() === '') return;
    if (addFormSelectedGroup === null) return;
    addFormSelectedGroup = null;
    refreshAddFormChipPickers();
  });

  document.getElementById('toggle-add-btn').addEventListener('click', function () {
    addPanel.hidden = !addPanel.hidden;
    if (!addPanel.hidden) newUrlInput.focus();
  });

  // タイトル未入力時のみ、URLからの仮タイトルで自動補完する
  newUrlInput.addEventListener('input', function () {
    if (newTitleInput.value.trim() !== '') return;
    var scheme = classifyUrl(newUrlInput.value.trim());
    if (!scheme) return;
    newTitleInput.value = deriveTentativeTitle(newUrlInput.value.trim(), scheme);
  });

  addForm.addEventListener('submit', function (e) {
    e.preventDefault();
    var url = newUrlInput.value.trim();
    var scheme = classifyUrl(url);
    if (!scheme) {
      alert('URLは http:// / https:// / file:/// のいずれかで始まる必要があります。');
      return;
    }

    var title = newTitleInput.value.trim() || deriveTentativeTitle(url, scheme);
    var tags = Array.from(new Set(Array.from(addFormSelectedTags).concat(parseTags(newTagsTextInput.value))));
    var newGroupTyped = newGroupTextInput.value.trim();
    var group = newGroupTyped || addFormSelectedGroup;
    var loginId = document.getElementById('new-login-id').value.trim() || null;
    var loginPassword = document.getElementById('new-login-password').value || null;
    var memo = document.getElementById('new-memo').value.trim() ? document.getElementById('new-memo').value : null;

    bookmarks.push({
      id: makeId(),
      url: url,
      title: title,
      tags: tags,
      group: group,
      groupOrder: getGroupOrderForName(group),
      loginId: loginId,
      loginPassword: loginPassword,
      memo: memo,
      scheme: scheme,
      domain: deriveDomain(url, scheme),
      order: bookmarks.length
    });

    saveData(bookmarks);
    addForm.reset();
    addFormSelectedTags.clear();
    addFormSelectedGroup = null;
    addPanel.hidden = true;
    render();
  });
```

`render()`関数（Task3で`renderFilterIndicator();`を追加した箇所の少し下、`refreshGroupDatalist();`の直後）に呼び出しを追加する。

```js
    refreshGroupDatalist();
    refreshAddFormChipPickers();
```

- [ ] **Step 6: GREEN確認**

タグ選択（複数トグル）:
```js
document.getElementById('toggle-add-btn').click();
document.querySelector('#new-tags-chips button[data-value="work"]').click();
document.querySelector('#new-tags-chips button[data-value="ref"]').click();
Array.from(document.querySelectorAll('#new-tags-chips button.active')).map(function(b){return b.getAttribute('data-value');}).sort()
```
期待するGREEN: `["ref", "work"]`

グループ選択と新規グループ欄の排他制御:
```js
document.querySelector('#new-group-chips button[data-value="仕事"]').click();
newGroupTextInput = document.getElementById('new-group-text');
newGroupTextInput.value = '個人';
newGroupTextInput.dispatchEvent(new Event('input'));
!!document.querySelector('#new-group-chips button.active')
```
期待するGREEN: `false`（テキスト入力で既存チップ選択が解除される）

新規タグ・グループを含めた保存:
```js
document.getElementById('new-url').value = 'https://example.com/d';
document.getElementById('new-url').dispatchEvent(new Event('input'));
document.getElementById('new-title').value = 'Delta';
document.getElementById('new-tags-text').value = 'new1, new2';
document.querySelector('#add-form button[type="submit"]').click();
var saved = JSON.parse(localStorage.getItem('bookmarks_data')).find(function(b){return b.url==='https://example.com/d';});
[saved.tags.sort(), saved.group]
```
期待するGREEN: `[["new1", "new2", "ref", "work"], "個人"]`（選択チップ＋新規入力欄の両方が反映される）

送信後のリセット:
```js
document.getElementById('toggle-add-btn').click();
[document.querySelectorAll('#new-tags-chips button.active').length, document.getElementById('new-group-text').value]
```
期待するGREEN: `[0, ""]`

空データでの表示:
```js
localStorage.setItem('bookmarks_data', JSON.stringify([]));
location.reload();
document.getElementById('toggle-add-btn').click();
[document.getElementById('new-tags-chips').textContent, document.getElementById('new-group-chips').textContent]
```
期待するGREEN: `["タグがまだありません", "グループがまだありません"]`

- [ ] **Step 7: bookmarks-filesync.htmlへの同一差分適用と一致確認**

HTML挿入位置（`#add-form`部分）はoffset +100で一致するはずだが、`grep -n 'id="new-group"'`でアンカー確認してから適用する。

- [ ] **Step 8: Commit**

```bash
git commit -m "feat: add shared tag/group chip picker, apply to add form"
```

---

### Task 5: 編集フォームへのチップピッカー適用（R3の一部 + R4-2）

**充足要件:** R3（編集フォーム分）、R4（編集フォーム）

**Files:**
- Modify: `bookmarks.html:1707-1708`（`renderEditCard()`内のタグ/グループ`.field`）
- Modify: `bookmarks.html:1729-1764`（`bindEditCardEvents()`）
- Modify: `bookmarks-filesync.html`の対応箇所（offset +302）

**Interfaces:**
- Consumes: Task4の`renderTagChipList()`, `renderGroupChipList()`, `parseTags()`, `getGroupOrderForName()`
- Produces: `refreshEditFormChipPickers()`（同タスク内で完結、他タスクからの依存なし）

- [ ] **Step 1: RED確認**

```js
document.querySelector('[data-card-id="k1"] [data-action="edit"]').click();
[!!document.getElementById('edit-tags-chips'), !!document.getElementById('edit-tags')]
```
期待するRED: `[false, true]`

- [ ] **Step 2: 実装（HTML）**

`bookmarks.html:1707-1708`を以下に置き換える。

```js
            '<div class="field"><label>タグ</label><p class="field-explain">タグ＝複数付けられる横断ラベル</p><div class="chip-list" id="edit-tags-chips"></div><input id="edit-tags-text" type="text" placeholder="新しいタグ(カンマ区切りで複数可)"></div>' +
            '<div class="field"><label>グループ</label><p class="field-explain">グループ＝1つだけ選べる置き場所</p><div class="chip-list" id="edit-group-chips"></div><input id="edit-group-text" type="text" placeholder="新しいグループ名"></div>' +
```

- [ ] **Step 3: 実装（JS）**

`bindEditCardEvents(card, b)`関数（`bookmarks.html:1729-1764`）を以下に置き換える。

```js
  var editFormSelectedTags = new Set();
  var editFormSelectedGroup = null;

  function refreshEditFormChipPickers() {
    var tagsChipsEl = document.getElementById('edit-tags-chips');
    var groupChipsEl = document.getElementById('edit-group-chips');
    var groupTextInput = document.getElementById('edit-group-text');
    if (!tagsChipsEl || !groupChipsEl) return; // 編集フォームが閉じている間は対象要素が存在しない
    renderTagChipList(tagsChipsEl, editFormSelectedTags, refreshEditFormChipPickers);
    renderGroupChipList(groupChipsEl, function () { return editFormSelectedGroup; }, function (val) {
      editFormSelectedGroup = val;
      if (val !== null) groupTextInput.value = '';
      refreshEditFormChipPickers();
    });
  }

  function bindEditCardEvents(card, b) {
    var form = card.querySelector('[data-edit-form]');
    var cancelBtn = card.querySelector('[data-action="cancel-edit"]');
    var groupTextInput = document.getElementById('edit-group-text');

    editFormSelectedTags = new Set(b.tags || []);
    editFormSelectedGroup = b.group || null;
    refreshEditFormChipPickers();

    groupTextInput.addEventListener('input', function () {
      if (groupTextInput.value.trim() === '') return;
      if (editFormSelectedGroup === null) return;
      editFormSelectedGroup = null;
      refreshEditFormChipPickers();
    });

    cancelBtn.addEventListener('click', function () {
      state.editingId = null;
      render();
    });

    form.addEventListener('submit', function (e) {
      e.preventDefault();
      var url = form.url.value.trim();
      var scheme = classifyUrl(url);
      if (!scheme) {
        alert('URLは http:// / https:// / file:/// のいずれかで始まる必要があります。');
        return;
      }
      b.url = url;
      b.scheme = scheme;
      b.domain = deriveDomain(url, scheme);
      b.title = form.title.value.trim() || deriveTentativeTitle(url, scheme);
      var tagsTextInput = document.getElementById('edit-tags-text');
      b.tags = Array.from(new Set(Array.from(editFormSelectedTags).concat(parseTags(tagsTextInput.value))));
      var groupTyped = groupTextInput.value.trim();
      var newGroup = groupTyped || editFormSelectedGroup;
      if (newGroup !== b.group) {
        b.group = newGroup;
        b.groupOrder = getGroupOrderForName(newGroup);
      }
      b.loginId = form.loginId.value.trim() || null;
      b.loginPassword = form.loginPassword.value || null;
      b.memo = form.memo.value.trim() ? form.memo.value : null;

      saveData(bookmarks);
      state.editingId = null;
      render();
    });
  }
```

- [ ] **Step 4: GREEN確認**

既存のタグ/グループが初期選択されていること:
```js
document.querySelector('[data-card-id="k1"] [data-action="edit"]').click();
[Array.from(document.querySelectorAll('#edit-tags-chips button.active')).map(function(b){return b.getAttribute('data-value');}).sort(),
 document.querySelector('#edit-group-chips button.active').getAttribute('data-value')]
```
期待するGREEN: `[["urgent", "work"], "仕事"]`（k1の既存タグ・グループがチップ選択済み）

タグ解除→保存:
```js
document.querySelector('#edit-tags-chips button[data-value="urgent"]').click();
document.querySelector('#edit-form, [data-edit-form]').closest('form').requestSubmit ?
  document.querySelector('[data-edit-form]').requestSubmit() :
  document.querySelector('[data-edit-form] button[type="submit"]').click();
JSON.parse(localStorage.getItem('bookmarks_data')).find(function(b){return b.id==='k1';}).tags
```
期待するGREEN: `["work"]`

グループ変更（新規グループ欄で入力しチップ選択が解除される）:
```js
document.querySelector('[data-card-id="k2"] [data-action="edit"]').click();
document.getElementById('edit-group-text').value = '個人';
document.getElementById('edit-group-text').dispatchEvent(new Event('input'));
[!!document.querySelector('#edit-group-chips button.active'),
 document.getElementById('edit-group-text').value]
```
期待するGREEN: `[false, "個人"]`

キャンセルで変更が破棄される:
```js
document.querySelector('[data-card-id="k2"] [data-action="cancel-edit"]').click();
JSON.parse(localStorage.getItem('bookmarks_data')).find(function(b){return b.id==='k2';}).group
```
期待するGREEN: `"仕事"`（未保存の変更は残らない）

- [ ] **Step 5: bookmarks-filesync.htmlへの同一差分適用と一致確認**

- [ ] **Step 6: Commit**

```bash
git commit -m "feat: apply tag/group chip picker to inline edit form"
```

---

### Task 6: 一括グループ化バーへのチップピッカー適用（R4-3）+ 未使用datalistの削除

**充足要件:** R4（一括グループ化バー）

**Files:**
- Modify: `bookmarks.html:802-808`（`#selection-bar`のHTML）
- Modify: `bookmarks.html:1223-1233`（`renderSelectionBar()`）
- Modify: `bookmarks.html:1235-1254`（選択バーのイベント登録）
- Modify: `bookmarks.html:1256-1272`（`#group-assign-btn`のクリックハンドラ）
- Modify: `bookmarks.html:1168-1171`（`refreshGroupDatalist()`削除）、`render()`内の呼び出し削除、`bookmarks.html:827`（`<datalist id="group-list">`削除）
- Modify: `bookmarks-filesync.html`の対応箇所（HTML offset +100、JS offset +302）

**Interfaces:**
- Consumes: Task4の`renderGroupChipList()`, `getGroupOrderForName()`
- Produces: なし（最終タスク、他への依存なし）

- [ ] **Step 1: RED確認**

```js
document.getElementById('toggle-select-btn').click();
[!!document.getElementById('assign-group-chips'), !!document.getElementById('group-list')]
```
期待するRED: `[false, true]`

- [ ] **Step 2: 実装（HTML、`#selection-bar`）**

`bookmarks.html:802-808`を以下に置き換える。

```html
<section class="selection-bar" id="selection-bar" hidden>
  <span class="count" id="selection-count"></span>
  <div class="chip-list" id="assign-group-chips"></div>
  <input type="text" id="group-name-input" placeholder="新しいグループ名">
  <button class="btn btn-amber" id="group-assign-btn" type="button">グループにまとめる</button>
  <button class="btn-quiet" id="group-clear-btn" type="button">グループ解除</button>
  <button class="btn-quiet" id="selection-clear-btn" type="button">選択解除</button>
</section>
```

- [ ] **Step 3: 実装（HTML/JS、`<datalist>`と`refreshGroupDatalist()`の削除）**

`bookmarks.html:827`の`<datalist id="group-list"></datalist>`行を削除する（`#new-group`はTask4で、`#edit-group`はTask5で、`#group-name-input`は本タスクStep2で、それぞれ`list="group-list"`参照を既に持たないため、この時点で参照元がゼロになる）。

`bookmarks.html:1168-1171`の以下の関数を削除する。

```js
  function refreshGroupDatalist() {
    var datalist = document.getElementById('group-list');
    datalist.innerHTML = getGroups().map(function (g) { return '<option value="' + escapeHtml(g) + '">'; }).join('');
  }
```

`render()`内の`refreshGroupDatalist();`呼び出し行を削除する。

- [ ] **Step 4: 実装（JS、選択バーの結線）**

`renderSelectionBar()`（`bookmarks.html:1223-1233`）を以下に置き換える。

```js
  var assignSelectedGroup = null;

  function refreshAssignGroupChips() {
    var chipsEl = document.getElementById('assign-group-chips');
    var textInput = document.getElementById('group-name-input');
    renderGroupChipList(chipsEl, function () { return assignSelectedGroup; }, function (val) {
      assignSelectedGroup = val;
      if (val !== null) textInput.value = '';
      refreshAssignGroupChips();
    });
  }

  function renderSelectionBar() {
    var bar = document.getElementById('selection-bar');
    bar.hidden = !state.selectMode;
    if (!state.selectMode) return;

    var count = state.selectedIds.size;
    document.getElementById('selection-count').textContent = count + ' 件選択中';
    document.getElementById('group-assign-btn').disabled = count === 0;
    document.getElementById('group-clear-btn').disabled = count === 0;
    document.getElementById('selection-clear-btn').disabled = count === 0;
    refreshAssignGroupChips();
  }
```

選択バーのイベント登録（`bookmarks.html:1235-1254`）のうち、`toggle-select-btn`のハンドラを以下に置き換える（`selection-clear-btn`・`group-clear-btn`のハンドラは変更しない）。

```js
  document.getElementById('toggle-select-btn').addEventListener('click', function () {
    state.selectMode = !state.selectMode;
    if (!state.selectMode) {
      state.selectedIds.clear();
      assignSelectedGroup = null;
    }
    render();
  });
```

`group-name-input`への入力で既存チップ選択を解除するリスナーを、`group-clear-btn`のイベント登録の直後に追加する。

```js
  document.getElementById('group-name-input').addEventListener('input', function () {
    var input = document.getElementById('group-name-input');
    if (input.value.trim() === '') return;
    if (assignSelectedGroup === null) return;
    assignSelectedGroup = null;
    refreshAssignGroupChips();
  });
```

- [ ] **Step 5: 実装（JS、`#group-assign-btn`のクリックハンドラ）**

`bookmarks.html:1256-1272`を以下に置き換える。

```js
  document.getElementById('group-assign-btn').addEventListener('click', function () {
    if (!state.selectedIds.size) return;
    var input = document.getElementById('group-name-input');
    var typed = input.value.trim();
    var name = typed || assignSelectedGroup;
    if (!name) {
      alert('グループ名を入力するか、既存のグループを選択してください。');
      return;
    }
    var go = getGroupOrderForName(name);
    bookmarks.forEach(function (b) {
      if (state.selectedIds.has(b.id)) { b.group = name; b.groupOrder = go; }
    });
    saveData(bookmarks);
    state.selectedIds.clear();
    input.value = '';
    assignSelectedGroup = null;
    render();
  });
```

- [ ] **Step 6: GREEN確認**

既存グループをチップで選んで一括適用:
```js
document.getElementById('toggle-select-btn').click();
document.querySelector('[data-card-id="k3"] .card-select').click();
document.querySelector('#assign-group-chips button[data-value="仕事"]').click();
document.getElementById('group-assign-btn').click();
JSON.parse(localStorage.getItem('bookmarks_data')).find(function(b){return b.id==='k3';}).group
```
期待するGREEN: `"仕事"`

新規グループ名を入力して一括適用（テキスト優先）:
```js
document.getElementById('toggle-select-btn').click();
document.querySelector('[data-card-id="k1"] .card-select').click();
document.getElementById('group-name-input').value = '新グループ';
document.getElementById('group-name-input').dispatchEvent(new Event('input'));
document.getElementById('group-assign-btn').click();
JSON.parse(localStorage.getItem('bookmarks_data')).find(function(b){return b.id==='k1';}).group
```
期待するGREEN: `"新グループ"`

未選択でアラート:
```js
document.getElementById('toggle-select-btn').click();
window.alert = function (msg) { window.__lastAlert = msg; };
document.getElementById('group-assign-btn').click();
window.__lastAlert
```
期待するGREEN: `"グループ名を入力するか、既存のグループを選択してください。"`

datalist完全削除の確認:
```js
!!document.getElementById('group-list')
```
期待するGREEN: `false`

- [ ] **Step 7: bookmarks-filesync.htmlへの同一差分適用と一致確認**

- [ ] **Step 8: Commit**

```bash
git commit -m "feat: apply group chip picker to bulk-assign bar, remove unused group datalist"
```

---

### Task 7: ヘッダー/サイドバーのスクロール固定（R5）

**充足要件:** R5

**Files:**
- Modify: `bookmarks.html:53-61`（`.app-header`）
- Modify: `bookmarks.html:309-316`直後（`.sidebar`用の新規メディアクエリ）
- Modify: `bookmarks.html:161-168`（`.help-panel`の`top`値）
- Modify: `bookmarks-filesync.html`の対応箇所（offset +75〜+76）

**Interfaces:**
- Consumes: なし
- Produces: なし（最終タスク、他への依存なし）

- [ ] **Step 1: RED確認**

シード投入・リロード後、`.main`内に十分な高さのダミーコンテンツがあることを前提に（既存の3件シードで確認可能なウィンドウ高に調整、または`window.innerHeight`を縮小するテストビューポートを使う）:

```js
window.scrollTo(0, 400);
document.querySelector('.app-header').getBoundingClientRect().top
```
期待するRED: `0`ではない負の値（スクロールに追従して画面外へ流れている）

- [ ] **Step 2: 実装（`.app-header`）**

`bookmarks.html:53-61`の既存ルールに`position`/`top`/`z-index`を追加する。

```css
  .app-header {
    display: flex;
    align-items: center;
    gap: 16px;
    flex-wrap: wrap;
    padding: 18px 24px;
    background: var(--ink);
    color: var(--ivory);
    position: sticky;
    top: 0;
    z-index: 10;
  }
```

- [ ] **Step 3: 実装（`.sidebar`、デスクトップ幅のみ）**

`bookmarks.html:316`（`.sidebar`ルールの閉じ`}`）の直後に新規メディアクエリを追加する。

```css

  @media (min-width: 761px) {
    .sidebar {
      position: sticky;
      top: 64px;
      max-height: calc(100vh - 64px);
      overflow-y: auto;
    }
  }
```

- [ ] **Step 4: 実装（`.help-panel`のtop調整）**

`bookmarks.html:166`の`top: 0;`を`top: 64px;`に変更する。

```css
  .help-panel {
    background: var(--card);
    border-bottom: 1px solid var(--sage);
    padding: 18px 24px;
    position: sticky;
    top: 64px;
    z-index: 5;
  }
```

- [ ] **Step 5: GREEN確認**

```js
window.scrollTo(0, 400);
document.querySelector('.app-header').getBoundingClientRect().top
```
期待するGREEN: `0`

```js
document.querySelector('.sidebar').getBoundingClientRect().top
```
期待するGREEN: `64`（デスクトップ幅、ヘッダー直下に張り付く）

```js
document.getElementById('help-panel').hidden = false; // ?キー相当
window.scrollTo(0, 400);
document.getElementById('help-panel').getBoundingClientRect().top
```
期待するGREEN: `64`（ヘッダーの直下、ヘッダーと重ならない）

サイドバー内部スクロール（タグ/グループを増やして長さを検証）:
```js
var seeded = JSON.parse(localStorage.getItem('bookmarks_data'));
for (var i = 0; i < 40; i++) {
  seeded.push({ id: 'tag' + i, url: 'https://example.com/t' + i, title: 'T' + i, tags: ['tag' + i], group: null, groupOrder: 0,
    loginId: null, loginPassword: null, memo: null, scheme: 'https', domain: 'example.com', order: 100 + i });
}
localStorage.setItem('bookmarks_data', JSON.stringify(seeded));
location.reload();
getComputedStyle(document.querySelector('.sidebar')).overflowY
```
期待するGREEN: `"auto"`（タグが増えてもサイドバー自体の高さは`max-height`で制限され、内部だけスクロールする）

モバイル幅では固定されない（ビューポート幅を760px以下に変更できる場合）:
```js
// claude-in-chromeのビューポートリサイズ機能で幅700pxに変更後
getComputedStyle(document.querySelector('.sidebar')).position
```
期待するGREEN: `"static"`（`@media (min-width: 761px)`の対象外）

- [ ] **Step 6: bookmarks-filesync.htmlへの同一差分適用と一致確認**

- [ ] **Step 7: Commit**

```bash
git commit -m "feat: make header and sidebar sticky on desktop widths"
```

---

## 実装順序

```
Task 1 (サイドバーnav色分け)         ← 独立
Task 2 (カードURL表示削除)           ← 独立
Task 3 (フィルタインジケーター)      ← 独立
Task 4 (チップピッカー基盤+追加フォーム) ← Task5/6の前提
Task 5 (編集フォーム)                ← Task4に依存
Task 6 (一括グループ化バー+datalist削除) ← Task4に依存、Task5の後(datalist参照ゼロ化の前提)
Task 7 (スクロール固定)              ← 独立
```

推奨実行順: **Task 1 → Task 2 → Task 3 → Task 4 → Task 5 → Task 6 → Task 7**（Task1/2/3/7はTask4〜6と並行実装も可能）。

## 完了後（受け入れ確認後にdoc-updaterへ委任）

全タスク完了後、`CLAUDE.md`の以下を更新する。

- Architecture節「4. 描画層」に、タグ/グループ入力が新規追加・インライン編集・一括グループ化の3箇所で共通の`renderTagChipList()`/`renderGroupChipList()`チップUIに統一されたこと、`#new-group`/`#edit-group`/`#group-name-input`の自由入力＋datalist方式が廃止されたことを追記する。
- 「データモデル」節は変更不要（`tags[]`/`group`/`groupOrder`のスキーマ自体は無変更のため）。
