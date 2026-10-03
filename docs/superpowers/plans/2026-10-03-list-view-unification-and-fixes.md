# リスト表示一本化・お気に入り拡張・UX改善 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 密度モード(ゆったり/コンパクト/リスト)を廃止してリスト表示一本化し、お気に入り行の機能拡張(コピー/編集)・検索対象拡張(メモ)・フィルタ選択時のスクロールリセット・リスト行のタグ表記追加・キーボードショートカットの最小化を行う。

**Architecture:** `bookmarks.html`/`bookmarks-filesync.html`の両方に対し、既存の`renderCardList()`/`renderFavoriteCardList()`を正式な`renderCard()`/`renderFavoriteCard()`として昇格させ、密度分岐コード(ディスパッチャ・トグルUI・CSS・state・localStorageキー)を削除する。以降のUX改善6件はこの単一関数の上に積み上げる。

**Tech Stack:** 単一HTMLファイル、ビルド・依存ゼロ、vanilla ES5-style JS(IIFE)。

**Spec:** `/home/asama/bookmarks-shelf/docs/superpowers/specs/2026-10-03-list-view-unification-and-fixes-design.md`(design-audit PASS済み、11/12・92%、最新コミット`73a1991`)

## Global Constraints

- 対象ファイルは`bookmarks.html`・`bookmarks-filesync.html`の両方。全タスクで両方に同一パターンを適用する。
- コメント・ドキュメントは日本語、コード・変数・関数名は英語で書く。
- 要求以上の機能追加はしない(YAGNI)。既存の正常なロジックを壊さない。
- `.card`/`.group-grid`/`.seal`/`.card-title`/`.card-footer`の**基本セレクタ自体は書き換えない**(`renderEditCard()`が密度モードと無関係に使用しているため)。`.card--list`/`.group-grid--list`/`.card-favorite--list`は削除せず維持し、常時適用する修飾クラスとして扱う。
- `bookmarks.html`固有の`SORT_MODE_KEY`/`loadSortMode()`/`state.sortMode`/`#toggle-sort-mode-btn`/頻度順ソート機能は本Planの対象外、無変更(密度モード関連とは別の既存機能)。
- テストはbrowser-manual-e2eプロファイルだが、このセッションにclaude-in-chromeツールは接続されていない。代替手段として、実ファイルから関数本体を直接抽出し(brace-countingによる抽出、再実装・手書きコピーは禁止)、Node.jsの`assert`モジュール(`console.assert`は使わない — 失敗してもexit 0になり検出できないため)で実行検証する。各タスクの報告には、この代替手段を使ったことを明記し、"E2E Evidence"と誤表記しないこと。
- 各タスクのコミットメッセージ末尾に以下を付与する:
  ```
  Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_01621j21ZvaFTbxSaonKKkWX
  ```

## Review Focus

- R1削除後、`renderEditCard()`が使う`.card`/`.group-grid`/`.seal`/`.card-title`/`.card-footer`の基本セレクタが無傷であること(誤って書き換えていないか) — Task 1のテストで確認する。
- R1で`bookmarks.html`固有の`SORT_MODE_KEY`/`loadSortMode()`/`state.sortMode`/`#toggle-sort-mode-btn`を誤って削除していないこと — Task 1のテストで確認する。
- R3の`edit`アクションがお気に入り行から正しく`state.editingId`を設定し、お気に入りセクション自体は変化せず、通常のグループセクション側に編集フォームが開くこと(既存の`render()`機構どおり) — Task 3のテストで確認する。
- R4のメモ検索追加が既存のtitle/url/tags検索結果に影響しないこと(文字列結合の末尾に追加するだけ、既存の一致条件を変えない) — Task 5のテストで確認する。
- R7のショートカット削除後、ヘルプパネルの`<dt>`/`<dd>`一覧が削除した5キーを含まず、残した4系統のキーのみを正確に記載していること — Task 6のテストで確認する。

---

### Task 1: 密度モード廃止・リスト表示への一本化(R1)

**Files:**
- Modify: `bookmarks.html`
- Modify: `bookmarks-filesync.html`

**Interfaces:**
- Consumes: なし(最初のタスク)。
- Produces: `renderCard(b)` — 密度分岐の無い単一関数、`data-card-id`を持つリスト行マークアップ(`class="card card--list"`)を返す。`renderFavoriteCard(b)` — 同様に単一関数、`data-fav-id`を持つリスト行マークアップ(`class="card card-favorite card-favorite--list"`)を返す。以降のTask 2/3/4がこの2関数を直接編集する。

以下、各ステップは`bookmarks.html`→`bookmarks-filesync.html`の順。行番号はHEAD時点のもの。本タスク内で複数の編集を行うため、2番目以降のステップを適用する際は提示された「変更前」テキストをファイル内で検索して特定すること(行番号はステップ適用後にずれる)。

- [ ] **Step 1: CSS削除 — `.density-toggle`/`.density-btn`**

`bookmarks.html:152-171`(filesync `:161-180`)を削除する:

```css
  .density-toggle {
    display: inline-flex;
    border: 1px solid rgba(245, 241, 230, 0.4);
    border-radius: 4px;
    overflow: hidden;
  }
  .density-btn {
    all: unset;
    cursor: pointer;
    padding: 8px 12px;
    display: flex;
    align-items: center;
    font-family: var(--font-body);
    font-size: 0.85rem;
    color: var(--ivory);
    border-right: 1px solid rgba(245, 241, 230, 0.4);
  }
  .density-btn:last-child { border-right: none; }
  .density-btn:hover { background: rgba(245, 241, 230, 0.12); }
  .density-btn.active { background: var(--amber); color: var(--amber-ink); }
```

両ファイルで内容は同一。ブロック全体を削除する(前後の既存ルールはそのまま残す)。

- [ ] **Step 2: CSS削除 — `.card--compact`/`.group-grid--compact`**

`bookmarks.html:1008-1016`(filesync `:1085-1093`)を削除する:

```css
  .card--compact .card-stub { width: 14px; }
  .card--compact .card-body { padding: 8px 9px; }
  .card--compact .seal { width: 18px; height: 18px; }
  .card--compact .seal img, .card--compact .seal svg { width: 11px; height: 11px; }
  .card--compact .card-title { font-size: 0.78rem; }
  .card--compact .card-footer { margin-top: 6px; gap: 2px; }
  .card--compact .card-actions { gap: 2px; }

  .group-grid--compact { grid-template-columns: repeat(auto-fill, minmax(170px, 1fr)); }
```

**`.card--list`/`.group-grid--list`(直後に続くブロック)は削除しない。維持する。**

- [ ] **Step 3: CSS削除 — `.card-favorite .card-stub`(コンパクト/ゆったり専用の1行のみ)**

`bookmarks.html:1098`(filesync `:1175`)の、以下の1行だけを削除する:

```css
  .card-favorite .card-stub { width: 14px; }
```

直後に続く`.card-favorite--list { ... }`ブロック、および`.fav-grip { ... }`は削除しない。維持する。

- [ ] **Step 4: マークアップ削除 — ヘッダーの密度トグル**

`bookmarks.html`のヘッダー(`:1163-1166`付近)から、以下の`<div class="density-toggle" ...>...</div>`ブロックを削除する:

```html
    <div class="density-toggle" id="density-toggle" role="group" aria-label="表示密度">
      <button type="button" class="density-btn" data-density="comfy">ゆったり</button>
      <button type="button" class="density-btn" data-density="compact">コンパクト</button>
      <button type="button" class="density-btn" data-density="list">リスト</button>
    </div>
```

このブロックの前後にある他のボタン(`#toggle-add-btn`、`#toggle-select-btn`、`#toggle-sort-mode-btn`、`#export-btn`、`#import-btn`等)はそのまま残す。

`bookmarks-filesync.html`も同一ブロック(`:1246-1250`付近)を削除する。周辺の`disabled`属性付きボタン群(filesync固有の未接続状態の表現)はそのまま残す。

- [ ] **Step 5: JS削除 — `DENSITY_MODE_KEY`/`loadDensityMode()`/state初期化**

`bookmarks.html:1330-1378`付近、変更前:

```js
  var STORAGE_KEY = 'bookmarks_data';
  var DENSITY_MODE_KEY = 'bookmarks_density_mode';
  var SORT_MODE_KEY = 'bookmarks_sort_mode';
  var UNGROUPED = null; // 未分類(グループ未割り当て)を表す内部値

  function loadDensityMode() {
    try {
      var v = localStorage.getItem(DENSITY_MODE_KEY);
      return (v === 'comfy' || v === 'compact' || v === 'list') ? v : 'compact';
    } catch (e) { return 'compact'; }
  }

  function loadSortMode() {
    try { return localStorage.getItem(SORT_MODE_KEY) === 'frequency' ? 'frequency' : 'manual'; }
    catch (e) { return 'manual'; }
  }

  var state = {
    search: '',
    tag: null,
    group: null,
    editingId: null,
    editingGroup: null,
    editingTag: null,
    selectMode: false,
    selectedIds: new Set(),
    sidebarTab: 'group',
    sortMode: loadSortMode(),
    densityMode: loadDensityMode()
  };
```

変更後(`DENSITY_MODE_KEY`変数・`loadDensityMode()`関数・`densityMode`プロパティを削除。`SORT_MODE_KEY`/`loadSortMode()`/`sortMode`は維持、末尾カンマの付け直しに注意):

```js
  var STORAGE_KEY = 'bookmarks_data';
  var SORT_MODE_KEY = 'bookmarks_sort_mode';
  var UNGROUPED = null; // 未分類(グループ未割り当て)を表す内部値

  function loadSortMode() {
    try { return localStorage.getItem(SORT_MODE_KEY) === 'frequency' ? 'frequency' : 'manual'; }
    catch (e) { return 'manual'; }
  }

  var state = {
    search: '',
    tag: null,
    group: null,
    editingId: null,
    editingGroup: null,
    editingTag: null,
    selectMode: false,
    selectedIds: new Set(),
    sidebarTab: 'group',
    sortMode: loadSortMode()
  };
```

`bookmarks-filesync.html:1430-1455`付近、変更前(このファイルには`STORAGE_KEY`/`SORT_MODE_KEY`/`loadSortMode()`/`sortMode`は存在しない):

```js
  var UNGROUPED = null; // 未分類(グループ未割り当て)を表す内部値
  var DENSITY_MODE_KEY = 'bookmarks_density_mode';

  function loadDensityMode() {
    try {
      var v = localStorage.getItem(DENSITY_MODE_KEY);
      return (v === 'comfy' || v === 'compact' || v === 'list') ? v : 'compact';
    } catch (e) { return 'compact'; }
  }

  var state = {
    search: '',
    tag: null,
    group: null,
    editingId: null,
    editingGroup: null,
    editingTag: null,
    selectMode: false,
    selectedIds: new Set(),
    sidebarTab: 'group',
    densityMode: loadDensityMode()
  };
```

変更後:

```js
  var UNGROUPED = null; // 未分類(グループ未割り当て)を表す内部値

  var state = {
    search: '',
    tag: null,
    group: null,
    editingId: null,
    editingGroup: null,
    editingTag: null,
    selectMode: false,
    selectedIds: new Set(),
    sidebarTab: 'group'
  };
```

- [ ] **Step 6: JS削除 — `render()`内の密度トグル同期ブロック**

`bookmarks.html:1709-1711`付近、変更前:

```js
    var sortModeBtn = document.getElementById('toggle-sort-mode-btn');
    sortModeBtn.textContent = state.sortMode === 'frequency' ? '手動順で表示' : 'よく使う順で表示';
    sortModeBtn.classList.toggle('active', state.sortMode === 'frequency');

    document.querySelectorAll('#density-toggle [data-density]').forEach(function (btn) {
      btn.classList.toggle('active', btn.getAttribute('data-density') === state.densityMode);
    });

    syncHeaderHeight();
```

変更後(`sortModeBtn`ブロックは維持、密度トグル同期ブロックのみ削除):

```js
    var sortModeBtn = document.getElementById('toggle-sort-mode-btn');
    sortModeBtn.textContent = state.sortMode === 'frequency' ? '手動順で表示' : 'よく使う順で表示';
    sortModeBtn.classList.toggle('active', state.sortMode === 'frequency');

    syncHeaderHeight();
```

`bookmarks-filesync.html:1988-1993`付近、変更前(`sortModeBtn`ブロックはこのファイルに存在しない):

```js
    document.querySelectorAll('#density-toggle [data-density]').forEach(function (btn) {
      btn.classList.toggle('active', btn.getAttribute('data-density') === state.densityMode);
    });

    syncHeaderHeight();
```

変更後:

```js
    syncHeaderHeight();
```

- [ ] **Step 7: JS削除 — `#density-toggle`のclickリスナー**

`bookmarks.html:2866-2870`付近、変更前(直前の`toggle-sort-mode-btn`リスナーは維持):

```js
  document.getElementById('toggle-sort-mode-btn').addEventListener('click', function () {
    state.sortMode = state.sortMode === 'frequency' ? 'manual' : 'frequency';
    try { localStorage.setItem(SORT_MODE_KEY, state.sortMode); } catch (e) {}
    render();
  });

  document.getElementById('density-toggle').addEventListener('click', function (e) {
    var btn = e.target.closest('[data-density]');
    if (!btn) return;
    state.densityMode = btn.getAttribute('data-density');
    try { localStorage.setItem(DENSITY_MODE_KEY, state.densityMode); } catch (err) {}
    render();
  });
```

変更後:

```js
  document.getElementById('toggle-sort-mode-btn').addEventListener('click', function () {
    state.sortMode = state.sortMode === 'frequency' ? 'manual' : 'frequency';
    try { localStorage.setItem(SORT_MODE_KEY, state.sortMode); } catch (e) {}
    render();
  });
```

`bookmarks-filesync.html:2160-2164`付近、変更前(直後の`selection-clear-btn`リスナーは維持):

```js
  document.getElementById('density-toggle').addEventListener('click', function (e) {
    var btn = e.target.closest('[data-density]');
    if (!btn) return;
    state.densityMode = btn.getAttribute('data-density');
    try { localStorage.setItem(DENSITY_MODE_KEY, state.densityMode); } catch (err) {}
    render();
  });

  document.getElementById('selection-clear-btn').addEventListener('click', function () {
```

変更後:

```js
  document.getElementById('selection-clear-btn').addEventListener('click', function () {
```

- [ ] **Step 8: JS統合 — `renderCard`系3関数+ディスパッチャ → 単一`renderCard()`**

両ファイル`bookmarks.html:2559-2716`相当(filesyncは+265行)、変更前(`renderCardComfy`・`renderCardCompact`・`renderCardList`・ディスパッチャの4関数):

```js
  function renderCardComfy(b) {
    var scheme = b.scheme;
    var iconHtml = resolveIconHtml(b);

    var tagsHtml = (b.tags || []).map(function (t) {
      return '<button type="button" tabindex="-1" class="tag-stamp clickable' + (state.tag === t ? ' active' : '') + '" data-action="filter-tag" data-tag="' + escapeHtml(t) + '">#' + escapeHtml(t) + '</button>';
    }).join('');

    var grip = !state.selectMode ? '<span class="stub-grip" aria-hidden="true">&#8942;</span><span class="stub-grip" aria-hidden="true">&#8942;</span>' : '';

    var checkboxHtml = state.selectMode
      ? '<input type="checkbox" tabindex="-1" class="card-select" data-action="select" aria-label="このブックマークを選択"' + (state.selectedIds.has(b.id) ? ' checked' : '') + '>'
      : '';

    var credentialsHtml = '';
    if (b.loginId || b.loginPassword) {
      credentialsHtml = '<div class="card-credentials">' +
        (b.loginId
          ? '<span class="cred-row"><span class="cred-label">ID</span><span class="cred-value">' + escapeHtml(b.loginId) + '</span>' +
            '<button type="button" tabindex="-1" class="btn-quiet" data-action="copy-login-id">コピー</button></span>'
          : '') +
        (b.loginPassword
          ? '<span class="cred-row"><span class="cred-label">PW</span><span class="cred-value" data-action="password-display">&#8226;&#8226;&#8226;&#8226;&#8226;&#8226;&#8226;&#8226;</span>' +
            '<button type="button" tabindex="-1" class="btn-quiet" data-action="toggle-reveal">表示</button>' +
            '<button type="button" tabindex="-1" class="btn-quiet" data-action="copy-login-password">コピー</button></span>'
          : '') +
      '</div>';
    }

    var memoHtml = b.memo ? '<p class="card-memo">' + escapeHtml(b.memo) + '</p>' : '';

    var pathCopyHtml = (scheme === 'file' || scheme === 'unc')
      ? '<button type="button" tabindex="-1" class="btn-quiet" data-action="copy-path">パスをコピー</button>'
      : '';

    return (
      '<article class="card" data-card-id="' + escapeHtml(b.id) + '" tabindex="-1">' +
        '<div class="card-stub">' + grip + '</div>' +
        '<div class="card-body">' +
          '<div class="card-top">' +
            checkboxHtml +
            '<div class="seal">' + iconHtml + '</div>' +
            '<div class="card-title-wrap">' +
              '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer" tabindex="-1">' + escapeHtml(b.title) + '</a></h3>' +
            '</div>' +
          '</div>' +
          (tagsHtml ? '<div class="card-tags">' + tagsHtml + '</div>' : '') +
          credentialsHtml +
          memoHtml +
          '<div class="card-footer">' +
            '<div class="card-actions">' +
              pathCopyHtml +
              '<button type="button" tabindex="-1" class="btn-quiet' + (b.pinned ? ' active' : '') + '" data-action="toggle-pin" aria-label="' + (b.pinned ? 'ピン留めを解除' : 'ピン留め') + '">' + (b.pinned ? '★' : '☆') + '</button>' +
              '<button type="button" tabindex="-1" class="btn-quiet" data-action="edit">編集</button>' +
              '<button type="button" tabindex="-1" class="btn-danger" data-action="delete">削除</button>' +
            '</div>' +
          '</div>' +
        '</div>' +
      '</article>'
    );
  }

  function renderCardCompact(b) {
    var scheme = b.scheme;
    var iconHtml = resolveIconHtml(b);
    var grip = !state.selectMode ? '<span class="stub-grip" aria-hidden="true">&#8942;</span>' : '';
    var checkboxHtml = state.selectMode
      ? '<input type="checkbox" tabindex="-1" class="card-select" data-action="select" aria-label="このブックマークを選択"' + (state.selectedIds.has(b.id) ? ' checked' : '') + '>'
      : '';
    var memoFilled = !!b.memo;
    var idFilled = !!b.loginId;
    var pwFilled = !!b.loginPassword;

    var indicatorsHtml =
      '<span class="ind-btn' + (memoFilled ? ' filled' : ' empty') + '" aria-hidden="true"><svg><use href="#ic-memo"></use></svg></span>' +
      (idFilled
        ? '<button type="button" tabindex="-1" class="ind-btn filled" data-action="copy-login-id" aria-label="IDをコピー"><svg><use href="#ic-key"></use></svg></button>'
        : '<span class="ind-btn empty" aria-hidden="true"><svg><use href="#ic-key"></use></svg></span>') +
      (pwFilled
        ? '<button type="button" tabindex="-1" class="ind-btn filled" data-action="copy-login-password" aria-label="パスワードをコピー"><svg><use href="#ic-lock"></use></svg></button>'
        : '<span class="ind-btn empty" aria-hidden="true"><svg><use href="#ic-lock"></use></svg></span>');

    var pathCopyHtml = (scheme === 'file' || scheme === 'unc')
      ? '<button type="button" tabindex="-1" class="icon-btn" data-action="copy-path" aria-label="パスをコピー">&#10697;</button>'
      : '';

    return (
      '<article class="card card--compact" data-card-id="' + escapeHtml(b.id) + '" tabindex="-1">' +
        '<div class="card-stub">' + grip + '</div>' +
        '<div class="card-body">' +
          '<div class="card-top">' +
            checkboxHtml +
            '<div class="seal">' + iconHtml + '</div>' +
            '<div class="card-title-wrap">' +
              '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer" tabindex="-1">' + escapeHtml(b.title) + '</a></h3>' +
            '</div>' +
          '</div>' +
          '<div class="card-footer">' +
            '<div class="card-actions">' +
              '<button type="button" tabindex="-1" class="icon-btn' + (b.pinned ? ' amber' : '') + '" data-action="toggle-pin" aria-label="' + (b.pinned ? 'ピン留めを解除' : 'ピン留め') + '">' + (b.pinned ? '★' : '☆') + '</button>' +
              indicatorsHtml +
              pathCopyHtml +
              '<button type="button" tabindex="-1" class="icon-btn" data-action="edit" aria-label="編集">&#9998;</button>' +
              '<button type="button" tabindex="-1" class="icon-btn" data-action="delete" aria-label="削除">&#10005;</button>' +
            '</div>' +
          '</div>' +
        '</div>' +
      '</article>'
    );
  }

  function renderCardList(b) {
    var iconHtml = resolveIconHtml(b);
    var checkboxHtml = state.selectMode
      ? '<input type="checkbox" tabindex="-1" class="card-select" data-action="select" aria-label="このブックマークを選択"' + (state.selectedIds.has(b.id) ? ' checked' : '') + '>'
      : '';
    var memoFilled = !!b.memo;
    var idFilled = !!b.loginId;
    var pwFilled = !!b.loginPassword;

    var memoPreviewHtml = memoFilled
      ? '<span class="row-memo-preview"><svg><use href="#ic-memo"></use></svg>' + escapeHtml(b.memo) + '</span>'
      : '';

    var indicatorsHtml =
      (idFilled
        ? '<button type="button" tabindex="-1" class="ind-btn filled" data-action="copy-login-id" aria-label="IDをコピー"><svg><use href="#ic-key"></use></svg></button>'
        : '<span class="ind-btn empty" aria-hidden="true"><svg><use href="#ic-key"></use></svg></span>') +
      (pwFilled
        ? '<button type="button" tabindex="-1" class="ind-btn filled" data-action="copy-login-password" aria-label="パスワードをコピー"><svg><use href="#ic-lock"></use></svg></button>'
        : '<span class="ind-btn empty" aria-hidden="true"><svg><use href="#ic-lock"></use></svg></span>');

    var pathCopyHtml = (b.scheme === 'file' || b.scheme === 'unc')
      ? '<button type="button" tabindex="-1" class="icon-btn" data-action="copy-path" aria-label="パスをコピー">&#10697;</button>'
      : '';

    return (
      '<article class="card card--list" data-card-id="' + escapeHtml(b.id) + '" tabindex="-1">' +
        checkboxHtml +
        '<div class="seal">' + iconHtml + '</div>' +
        '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer" tabindex="-1">' + escapeHtml(b.title) + '</a></h3>' +
        memoPreviewHtml +
        '<div class="card-footer"><div class="card-actions">' +
          '<button type="button" tabindex="-1" class="icon-btn' + (b.pinned ? ' amber' : '') + '" data-action="toggle-pin" aria-label="' + (b.pinned ? 'ピン留めを解除' : 'ピン留め') + '">' + (b.pinned ? '★' : '☆') + '</button>' +
          indicatorsHtml +
          pathCopyHtml +
          '<button type="button" tabindex="-1" class="icon-btn" data-action="edit" aria-label="編集">&#9998;</button>' +
          '<button type="button" tabindex="-1" class="icon-btn" data-action="delete" aria-label="削除">&#10005;</button>' +
        '</div></div>' +
      '</article>'
    );
  }

  function renderCard(b) {
    if (state.densityMode === 'compact') return renderCardCompact(b);
    if (state.densityMode === 'list') return renderCardList(b);
    return renderCardComfy(b);
  }
```

変更後(`renderCardComfy`・`renderCardCompact`・ディスパッチャを削除し、`renderCardList`の本体を`renderCard`という名前で残す):

```js
  function renderCard(b) {
    var iconHtml = resolveIconHtml(b);
    var checkboxHtml = state.selectMode
      ? '<input type="checkbox" tabindex="-1" class="card-select" data-action="select" aria-label="このブックマークを選択"' + (state.selectedIds.has(b.id) ? ' checked' : '') + '>'
      : '';
    var memoFilled = !!b.memo;
    var idFilled = !!b.loginId;
    var pwFilled = !!b.loginPassword;

    var memoPreviewHtml = memoFilled
      ? '<span class="row-memo-preview"><svg><use href="#ic-memo"></use></svg>' + escapeHtml(b.memo) + '</span>'
      : '';

    var indicatorsHtml =
      (idFilled
        ? '<button type="button" tabindex="-1" class="ind-btn filled" data-action="copy-login-id" aria-label="IDをコピー"><svg><use href="#ic-key"></use></svg></button>'
        : '<span class="ind-btn empty" aria-hidden="true"><svg><use href="#ic-key"></use></svg></span>') +
      (pwFilled
        ? '<button type="button" tabindex="-1" class="ind-btn filled" data-action="copy-login-password" aria-label="パスワードをコピー"><svg><use href="#ic-lock"></use></svg></button>'
        : '<span class="ind-btn empty" aria-hidden="true"><svg><use href="#ic-lock"></use></svg></span>');

    var pathCopyHtml = (b.scheme === 'file' || b.scheme === 'unc')
      ? '<button type="button" tabindex="-1" class="icon-btn" data-action="copy-path" aria-label="パスをコピー">&#10697;</button>'
      : '';

    return (
      '<article class="card card--list" data-card-id="' + escapeHtml(b.id) + '" tabindex="-1">' +
        checkboxHtml +
        '<div class="seal">' + iconHtml + '</div>' +
        '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer" tabindex="-1">' + escapeHtml(b.title) + '</a></h3>' +
        memoPreviewHtml +
        '<div class="card-footer"><div class="card-actions">' +
          '<button type="button" tabindex="-1" class="icon-btn' + (b.pinned ? ' amber' : '') + '" data-action="toggle-pin" aria-label="' + (b.pinned ? 'ピン留めを解除' : 'ピン留め') + '">' + (b.pinned ? '★' : '☆') + '</button>' +
          indicatorsHtml +
          pathCopyHtml +
          '<button type="button" tabindex="-1" class="icon-btn" data-action="edit" aria-label="編集">&#9998;</button>' +
          '<button type="button" tabindex="-1" class="icon-btn" data-action="delete" aria-label="削除">&#10005;</button>' +
        '</div></div>' +
      '</article>'
    );
  }
```

両ファイルで同一内容。`bookmarks-filesync.html`も同様に適用する(`:2824-3008`相当)。

- [ ] **Step 9: JS統合 — `renderFavoriteCard`系2関数+ディスパッチャ → 単一`renderFavoriteCard()`**

両ファイル(bookmarks.html `:1936-1975`相当)、変更前:

```js
  function renderFavoriteCardComfy(b) {
    var iconHtml = resolveIconHtml(b);
    return (
      '<article class="card card-favorite" data-fav-id="' + escapeHtml(b.id) + '">' +
        '<div class="card-stub"><span class="stub-grip" aria-hidden="true">&#8942;</span></div>' +
        '<div class="card-body">' +
          '<div class="card-top">' +
            '<div class="seal">' + iconHtml + '</div>' +
            '<div class="card-title-wrap">' +
              '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer">' + escapeHtml(b.title) + '</a></h3>' +
            '</div>' +
          '</div>' +
          '<div class="card-footer">' +
            '<div class="card-actions">' +
              '<button type="button" class="btn-quiet" data-action="unpin">ピン解除</button>' +
            '</div>' +
          '</div>' +
        '</div>' +
      '</article>'
    );
  }

  function renderFavoriteCardList(b) {
    var iconHtml = resolveIconHtml(b);
    return (
      '<article class="card card-favorite card-favorite--list" data-fav-id="' + escapeHtml(b.id) + '">' +
        '<span class="fav-grip" aria-hidden="true">&#8942;&#8942;</span>' +
        '<div class="seal">' + iconHtml + '</div>' +
        '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer">' + escapeHtml(b.title) + '</a></h3>' +
        '<span class="row-group-chip">' + escapeHtml(b.group || '未分類') + '</span>' +
        '<div class="card-footer"><div class="card-actions">' +
          '<button type="button" class="icon-btn" data-action="unpin" aria-label="ピン解除">&#9733;</button>' +
        '</div></div>' +
      '</article>'
    );
  }

  function renderFavoriteCard(b) {
    if (state.densityMode === 'list') return renderFavoriteCardList(b);
    return renderFavoriteCardComfy(b);
  }
```

変更後(`renderFavoriteCardComfy`・ディスパッチャを削除し、`renderFavoriteCardList`の本体を`renderFavoriteCard`という名前で残す。Task 3でこの関数に`copy-login-id`/`copy-login-password`/`edit`ボタンを追加するため、現時点ではまだ`unpin`のみ):

```js
  function renderFavoriteCard(b) {
    var iconHtml = resolveIconHtml(b);
    return (
      '<article class="card card-favorite card-favorite--list" data-fav-id="' + escapeHtml(b.id) + '">' +
        '<span class="fav-grip" aria-hidden="true">&#8942;&#8942;</span>' +
        '<div class="seal">' + iconHtml + '</div>' +
        '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer">' + escapeHtml(b.title) + '</a></h3>' +
        '<span class="row-group-chip">' + escapeHtml(b.group || '未分類') + '</span>' +
        '<div class="card-footer"><div class="card-actions">' +
          '<button type="button" class="icon-btn" data-action="unpin" aria-label="ピン解除">&#9733;</button>' +
        '</div></div>' +
      '</article>'
    );
  }
```

`renderFavoritesSection()`内で`renderFavoriteCard`を呼んでいる箇所(`items.map(renderFavoriteCard)`)は名前が変わらないため無変更。

- [ ] **Step 10: JS修正 — `renderFavoritesSection()`内の`gridModifier`をハードコード化**

両ファイルの`renderFavoritesSection()`内、変更前:

```js
    var items = pinned.slice().sort(compareFavoriteOrder);
    var gridModifier = state.densityMode === 'list' ? ' group-grid--list' : '';
    container.innerHTML =
      '<section class="favorites-section">' +
        '<div class="group-section-header"><h2 class="group-section-title">お気に入り</h2><span class="group-section-count">' + items.length + ' 件</span></div>' +
        '<div class="group-grid' + gridModifier + '">' + items.map(renderFavoriteCard).join('') +
        '</div>' +
      '</section>';
```

変更後:

```js
    var items = pinned.slice().sort(compareFavoriteOrder);
    container.innerHTML =
      '<section class="favorites-section">' +
        '<div class="group-section-header"><h2 class="group-section-title">お気に入り</h2><span class="group-section-count">' + items.length + ' 件</span></div>' +
        '<div class="group-grid group-grid--list">' + items.map(renderFavoriteCard).join('') +
        '</div>' +
      '</section>';
```

- [ ] **Step 11: JS修正 — `renderSections()`内の`gridModifier`をハードコード化**

両ファイルの`renderSections()`内(bookmarks.html `:2051-2053`相当)、変更前:

```js
          var items = sortItems(filtered.filter(function (b) { return (b.group || null) === key; }));
          var gridModifier = state.densityMode === 'compact' ? ' group-grid--compact' : state.densityMode === 'list' ? ' group-grid--list' : '';
          var body = items.length
            ? '<div class="group-grid' + gridModifier + '" data-group-grid="' + escapeHtml(key || '') + '">' +
```

変更後:

```js
          var items = sortItems(filtered.filter(function (b) { return (b.group || null) === key; }));
          var body = items.length
            ? '<div class="group-grid group-grid--list" data-group-grid="' + escapeHtml(key || '') + '">' +
```

- [ ] **Step 12: 残存確認 — grepで密度モード関連の削除漏れが無いことを確認する**

Run: `grep -n "densityMode\|DENSITY_MODE_KEY\|card--compact\|group-grid--compact\|card-favorite \.card-stub\|renderCardComfy\|renderCardCompact\|renderFavoriteCardComfy" bookmarks.html bookmarks-filesync.html`

Expected: 出力無し(0件)。1件でも出力されたら削除漏れなので該当箇所を修正する。

Run: `grep -n "card--list\|group-grid--list\|card-favorite--list\|SORT_MODE_KEY\|loadSortMode\|toggle-sort-mode-btn" bookmarks.html bookmarks-filesync.html`

Expected: いずれも両ファイルに出力がある(`SORT_MODE_KEY`/`loadSortMode`/`toggle-sort-mode-btn`は`bookmarks-filesync.html`には元から存在しないため、このファイルでは0件が正しい)。維持すべきものが消えていないことを確認する。

- [ ] **Step 13: Node.js検証 — `renderCard`/`renderFavoriteCard`が例外無く動作し、密度分岐が残っていないことを確認する**

スクラッチディレクトリに`verify-task1.js`として以下を保存し実行する:

```js
'use strict';
var fs = require('fs');
var assert = require('assert');

function extractFunction(src, name) {
  var m = new RegExp('function\\s+' + name + '\\s*\\(').exec(src);
  assert.ok(m, 'function not found: ' + name);
  var start = m.index;
  var braceStart = src.indexOf('{', start);
  var depth = 0;
  for (var i = braceStart; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) return src.slice(start, i + 1); }
  }
  throw new Error('unbalanced braces for: ' + name);
}

['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');

  var renderCardSrc = extractFunction(src, 'renderCard');
  var renderFavoriteCardSrc = extractFunction(src, 'renderFavoriteCard');
  assert.ok(renderCardSrc.indexOf('state.densityMode') === -1, file + ': renderCard still branches on state.densityMode');
  assert.ok(renderFavoriteCardSrc.indexOf('state.densityMode') === -1, file + ': renderFavoriteCard still branches on state.densityMode');
  assert.ok(renderCardSrc.indexOf("'card--list'") === -1 && renderCardSrc.indexOf('card card--list') !== -1, file + ': renderCard should hardcode card--list');

  var resolveIconHtmlSrc = extractFunction(src, 'resolveIconHtml');
  var resolveOpenHrefSrc = extractFunction(src, 'resolveOpenHref');
  var escapeHtmlSrc = extractFunction(src, 'escapeHtml');

  var wrapperSrc = [escapeHtmlSrc, resolveOpenHrefSrc, resolveIconHtmlSrc, renderCardSrc, renderFavoriteCardSrc].join('\n');
  var factory = new Function('state', wrapperSrc + '\nreturn { renderCard: renderCard, renderFavoriteCard: renderFavoriteCard };');
  var mod = factory({ selectMode: false, tag: null, editingId: null });

  var sample = { id: 'b1', url: 'https://example.com/', title: 'Example', scheme: 'https', domain: 'example.com', tags: [], group: null, loginId: '', loginPassword: '', memo: '', pinned: false };

  var cardHtml = mod.renderCard(sample);
  assert.ok(cardHtml.indexOf('data-card-id="b1"') !== -1, file + ': renderCard output missing data-card-id');
  assert.ok(cardHtml.indexOf('class="card card--list"') !== -1, file + ': renderCard output missing card--list class');

  var favHtml = mod.renderFavoriteCard(sample);
  assert.ok(favHtml.indexOf('data-fav-id="b1"') !== -1, file + ': renderFavoriteCard output missing data-fav-id');
  assert.ok(favHtml.indexOf('card-favorite--list') !== -1, file + ': renderFavoriteCard output missing card-favorite--list class');

  console.log(file + ': OK');
});
```

Run: `node verify-task1.js`(スクラッチディレクトリで実行)
Expected: 両ファイルで`OK`が出力され、exit code 0。

- [ ] **Step 14: Node.js検証 — `renderEditCard()`の基本CSSセレクタ依存が無傷であることを確認する**

Run: `grep -n "class=\"card\"" bookmarks.html bookmarks-filesync.html | grep -c "renderEditCard\|card-stub\"></div>"`の代わりに、実際に`renderEditCard`関数本体を抽出して出力を確認する簡易チェックを行う:

```js
// verify-task1-editcard.js
'use strict';
var fs = require('fs');
var assert = require('assert');
function extractFunction(src, name) {
  var m = new RegExp('function\\s+' + name + '\\s*\\(').exec(src);
  assert.ok(m, 'function not found: ' + name);
  var start = m.index, braceStart = src.indexOf('{', start), depth = 0;
  for (var i = braceStart; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) return src.slice(start, i + 1); }
  }
  throw new Error('unbalanced braces');
}
['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');
  var editSrc = extractFunction(src, 'renderEditCard');
  assert.ok(editSrc.indexOf("'<article class=\"card\"") !== -1, file + ': renderEditCard no longer uses base .card class as before');
  assert.ok(editSrc.indexOf('card--list') === -1, file + ': renderEditCard unexpectedly references card--list');
  console.log(file + ': renderEditCard unchanged OK');
});
```

Run: `node verify-task1-editcard.js`
Expected: 両ファイルで`unchanged OK`が出力される。

- [ ] **Step 15: Commit**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "$(cat <<'EOF'
feat: 密度モードを廃止しリスト表示に一本化(R1)

ゆったり/コンパクトモードのレンダー関数・トグルUI・CSS・state・localStorageキーを削除し、
renderCard()/renderFavoriteCard()をリスト表示専用の単一関数に統合した。

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01621j21ZvaFTbxSaonKKkWX
EOF
)"
```

---

### Task 2: リスト行へのドラッグ用グリップ追加(R2)

**Files:**
- Modify: `bookmarks.html`
- Modify: `bookmarks-filesync.html`

**Interfaces:**
- Consumes: Task 1が生成した`renderCard(b)`(`class="card card--list"`、`checkboxHtml`から始まる本体)。
- Produces: `renderCard(b)`の先頭に`.card-stub`グリップを追加したマークアップ。Task 3/4はこの関数の別の箇所(フッターボタン群・タイトル付近)を編集するため、この追加がそれらと競合しないことを確認する。

- [ ] **Step 1: CSS追加 — `.card--list .card-stub`のサイズ調整**

`.card--list { ... }`ブロック(Task 1で維持された、bookmarks.html `:1059-1078`相当)の末尾に、以下の1行を追加する:

```css
  .card--list .card-stub { width: 14px; }
```

両ファイルに追加する。

- [ ] **Step 2: JS修正 — `renderCard(b)`にグリップ要素を追加**

Task 1で確定した`renderCard(b)`の変更前(両ファイル共通):

```js
  function renderCard(b) {
    var iconHtml = resolveIconHtml(b);
    var checkboxHtml = state.selectMode
      ? '<input type="checkbox" tabindex="-1" class="card-select" data-action="select" aria-label="このブックマークを選択"' + (state.selectedIds.has(b.id) ? ' checked' : '') + '>'
      : '';
```

変更後(`grip`変数を追加し、`<article>`直下の最初の子要素として`.card-stub`を挿入):

```js
  function renderCard(b) {
    var iconHtml = resolveIconHtml(b);
    var grip = !state.selectMode ? '<span class="stub-grip" aria-hidden="true">&#8942;</span>' : '';
    var checkboxHtml = state.selectMode
      ? '<input type="checkbox" tabindex="-1" class="card-select" data-action="select" aria-label="このブックマークを選択"' + (state.selectedIds.has(b.id) ? ' checked' : '') + '>'
      : '';
```

続けて、`return (...)`内の`<article ...>`開始直後(`checkboxHtml +`の前)に`'<div class="card-stub">' + grip + '</div>' +`を挿入する。変更前:

```js
    return (
      '<article class="card card--list" data-card-id="' + escapeHtml(b.id) + '" tabindex="-1">' +
        checkboxHtml +
```

変更後:

```js
    return (
      '<article class="card card--list" data-card-id="' + escapeHtml(b.id) + '" tabindex="-1">' +
        '<div class="card-stub">' + grip + '</div>' +
        checkboxHtml +
```

両ファイルに同一の変更を適用する。

- [ ] **Step 3: Node.js検証 — グリップがドラッグ可否(`selectMode`)に応じて正しく出力されることを確認する**

スクラッチディレクトリに`verify-task2.js`として保存:

```js
'use strict';
var fs = require('fs');
var assert = require('assert');
function extractFunction(src, name) {
  var m = new RegExp('function\\s+' + name + '\\s*\\(').exec(src);
  assert.ok(m, 'function not found: ' + name);
  var start = m.index, braceStart = src.indexOf('{', start), depth = 0;
  for (var i = braceStart; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) return src.slice(start, i + 1); }
  }
  throw new Error('unbalanced braces');
}
['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');
  var renderCardSrc = extractFunction(src, 'renderCard');
  var resolveIconHtmlSrc = extractFunction(src, 'resolveIconHtml');
  var resolveOpenHrefSrc = extractFunction(src, 'resolveOpenHref');
  var escapeHtmlSrc = extractFunction(src, 'escapeHtml');
  var factory = new Function('state', [escapeHtmlSrc, resolveOpenHrefSrc, resolveIconHtmlSrc, renderCardSrc].join('\n') + '\nreturn renderCard;');

  var sample = { id: 'b1', url: 'https://example.com/', title: 'Example', scheme: 'https', domain: 'example.com', tags: [], group: null, loginId: '', loginPassword: '', memo: '', pinned: false };

  var renderCardNormal = factory({ selectMode: false, tag: null, editingId: null, selectedIds: new Set() });
  var htmlNormal = renderCardNormal(sample);
  assert.ok(htmlNormal.indexOf('class="card-stub"') !== -1, file + ': missing .card-stub wrapper');
  assert.ok(htmlNormal.indexOf('stub-grip') !== -1, file + ': grip glyph missing when selectMode is false');

  var renderCardSelecting = factory({ selectMode: true, tag: null, editingId: null, selectedIds: new Set() });
  var htmlSelecting = renderCardSelecting(sample);
  assert.ok(htmlSelecting.indexOf('stub-grip') === -1, file + ': grip glyph should be empty while selectMode is true');

  console.log(file + ': OK');
});
```

Run: `node verify-task2.js`
Expected: 両ファイルで`OK`。

- [ ] **Step 4: Commit**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "$(cat <<'EOF'
feat: リスト行にドラッグ用グリップを追加(R2)

並び替え・グループ移動機構自体は既に密度非依存で動作していたため、
掴みやすい視覚的な領域(.card-stub)を追加するのみ。

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01621j21ZvaFTbxSaonKKkWX
EOF
)"
```

---

### Task 3: お気に入り行へのコピー/編集機能追加(R3)

**Files:**
- Modify: `bookmarks.html`
- Modify: `bookmarks-filesync.html`

**Interfaces:**
- Consumes: Task 1が生成した`renderFavoriteCard(b)`、既存の`copyToClipboard(text, btn)`(シグネチャ不変)。
- Produces: `renderFavoriteCard(b)`に`copy-login-id`/`copy-login-password`/`edit`ボタンを追加したマークアップ、`renderFavoritesSection()`内のクリックバインドループにこれら3アクションのハンドラを追加。

- [ ] **Step 1: JS修正 — `renderFavoriteCard(b)`に`.ind-btn`(ID/パスワード)ボタンを追加**

両ファイルの`renderFavoriteCard(b)`、変更前:

```js
  function renderFavoriteCard(b) {
    var iconHtml = resolveIconHtml(b);
    return (
      '<article class="card card-favorite card-favorite--list" data-fav-id="' + escapeHtml(b.id) + '">' +
        '<span class="fav-grip" aria-hidden="true">&#8942;&#8942;</span>' +
        '<div class="seal">' + iconHtml + '</div>' +
        '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer">' + escapeHtml(b.title) + '</a></h3>' +
        '<span class="row-group-chip">' + escapeHtml(b.group || '未分類') + '</span>' +
        '<div class="card-footer"><div class="card-actions">' +
          '<button type="button" class="icon-btn" data-action="unpin" aria-label="ピン解除">&#9733;</button>' +
        '</div></div>' +
      '</article>'
    );
  }
```

変更後(`idFilled`/`pwFilled`に基づく`.ind-btn`表現を通常カードのリスト行と同じパターンで追加し、既存の`unpin`ボタンの後に`indicatorsHtml`、その後に`edit`ボタンを配置):

```js
  function renderFavoriteCard(b) {
    var iconHtml = resolveIconHtml(b);
    var idFilled = !!b.loginId;
    var pwFilled = !!b.loginPassword;

    var indicatorsHtml =
      (idFilled
        ? '<button type="button" class="ind-btn filled" data-action="copy-login-id" aria-label="IDをコピー"><svg><use href="#ic-key"></use></svg></button>'
        : '<span class="ind-btn empty" aria-hidden="true"><svg><use href="#ic-key"></use></svg></span>') +
      (pwFilled
        ? '<button type="button" class="ind-btn filled" data-action="copy-login-password" aria-label="パスワードをコピー"><svg><use href="#ic-lock"></use></svg></button>'
        : '<span class="ind-btn empty" aria-hidden="true"><svg><use href="#ic-lock"></use></svg></span>');

    return (
      '<article class="card card-favorite card-favorite--list" data-fav-id="' + escapeHtml(b.id) + '">' +
        '<span class="fav-grip" aria-hidden="true">&#8942;&#8942;</span>' +
        '<div class="seal">' + iconHtml + '</div>' +
        '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer">' + escapeHtml(b.title) + '</a></h3>' +
        '<span class="row-group-chip">' + escapeHtml(b.group || '未分類') + '</span>' +
        '<div class="card-footer"><div class="card-actions">' +
          '<button type="button" class="icon-btn" data-action="unpin" aria-label="ピン解除">&#9733;</button>' +
          indicatorsHtml +
          '<button type="button" class="icon-btn" data-action="edit" aria-label="編集">&#9998;</button>' +
        '</div></div>' +
      '</article>'
    );
  }
```

両ファイルに同一の変更を適用する。`.ind-btn`/`.icon-btn`のCSSはTask 1以前から既存のため追加不要。

- [ ] **Step 2: JS修正 — `renderFavoritesSection()`のクリックバインドループに3アクションを追加**

`bookmarks.html`の`renderFavoritesSection()`内、変更前:

```js
      var unpinBtn = el.querySelector('[data-action="unpin"]');
      if (unpinBtn) unpinBtn.addEventListener('click', function () {
        b.pinned = false;
        b.favoriteOrder = null;
        saveData(bookmarks);
        render();
      });
    });

    bindFavoriteDragEvents(container);
```

変更後(`copy-login-id`/`copy-login-password`/`edit`のハンドラを追加。`edit`は通常カードの`editBtn`ハンドラと同じ2行をそのまま複製する):

```js
      var unpinBtn = el.querySelector('[data-action="unpin"]');
      if (unpinBtn) unpinBtn.addEventListener('click', function () {
        b.pinned = false;
        b.favoriteOrder = null;
        saveData(bookmarks);
        render();
      });

      var copyIdBtn = el.querySelector('[data-action="copy-login-id"]');
      if (copyIdBtn) copyIdBtn.addEventListener('click', function () { copyToClipboard(b.loginId, copyIdBtn); });

      var copyPwBtn = el.querySelector('[data-action="copy-login-password"]');
      if (copyPwBtn) copyPwBtn.addEventListener('click', function () { copyToClipboard(b.loginPassword, copyPwBtn); });

      var editBtn = el.querySelector('[data-action="edit"]');
      if (editBtn) editBtn.addEventListener('click', function () { state.editingId = b.id; render(); });
    });

    bindFavoriteDragEvents(container);
```

`bookmarks-filesync.html`の`renderFavoritesSection()`(タイトルリンクのアクセス記録ブロックが無い点のみ`bookmarks.html`と異なる、CLAUDE.md記載の既存差異)にも同一のブロックを、既存の`unpinBtn`ハンドラの直後・`bindFavoriteDragEvents(container);`の直前に追加する。

- [ ] **Step 3: Node.js検証 — 3アクションがマークアップ・バインドの両方に存在することを確認する**

スクラッチディレクトリに`verify-task3.js`として保存:

```js
'use strict';
var fs = require('fs');
var assert = require('assert');
function extractFunction(src, name) {
  var m = new RegExp('function\\s+' + name + '\\s*\\(').exec(src);
  assert.ok(m, 'function not found: ' + name);
  var start = m.index, braceStart = src.indexOf('{', start), depth = 0;
  for (var i = braceStart; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) return src.slice(start, i + 1); }
  }
  throw new Error('unbalanced braces');
}
['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');

  var renderFavoriteCardSrc = extractFunction(src, 'renderFavoriteCard');
  var resolveIconHtmlSrc = extractFunction(src, 'resolveIconHtml');
  var resolveOpenHrefSrc = extractFunction(src, 'resolveOpenHref');
  var escapeHtmlSrc = extractFunction(src, 'escapeHtml');
  var factory = new Function([escapeHtmlSrc, resolveOpenHrefSrc, resolveIconHtmlSrc, renderFavoriteCardSrc].join('\n') + '\nreturn renderFavoriteCard;');
  var renderFavoriteCard = factory();

  var filled = { id: 'f1', url: 'https://example.com/', title: 'Example', scheme: 'https', domain: 'example.com', group: null, loginId: 'user1', loginPassword: 'pw1', pinned: true };
  var htmlFilled = renderFavoriteCard(filled);
  assert.ok(htmlFilled.indexOf('data-action="copy-login-id"') !== -1, file + ': copy-login-id button missing');
  assert.ok(htmlFilled.indexOf('data-action="copy-login-password"') !== -1, file + ': copy-login-password button missing');
  assert.ok(htmlFilled.indexOf('data-action="edit"') !== -1, file + ': edit button missing');
  assert.ok(htmlFilled.indexOf('ind-btn filled') !== -1, file + ': filled indicator missing when loginId/loginPassword present');

  var empty = { id: 'f2', url: 'https://example.com/2', title: 'Example2', scheme: 'https', domain: 'example.com', group: null, loginId: '', loginPassword: '', pinned: true };
  var htmlEmpty = renderFavoriteCard(empty);
  assert.ok(htmlEmpty.indexOf('ind-btn empty') !== -1, file + ': empty indicator missing when loginId/loginPassword absent');

  var favSectionSrc = extractFunction(src, 'renderFavoritesSection');
  assert.ok(favSectionSrc.indexOf('copy-login-id') !== -1, file + ': renderFavoritesSection does not bind copy-login-id');
  assert.ok(favSectionSrc.indexOf('copy-login-password') !== -1, file + ': renderFavoritesSection does not bind copy-login-password');
  assert.ok(favSectionSrc.indexOf('state.editingId = b.id') !== -1, file + ': renderFavoritesSection does not set state.editingId on edit');

  console.log(file + ': OK');
});
```

Run: `node verify-task3.js`
Expected: 両ファイルで`OK`。

- [ ] **Step 4: Commit**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "$(cat <<'EOF'
feat: お気に入り行にID/パスワードコピー・編集ボタンを追加(R3)

編集はstate.editingIdを設定するのみで、お気に入りセクション自体は変化しない
(編集フォームは該当ブックマークが属する通常のグループセクション側に開く、既存のrender()機構どおり)。

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01621j21ZvaFTbxSaonKKkWX
EOF
)"
```

---

### Task 4: リスト行へのタグ表記追加(R6)

**Files:**
- Modify: `bookmarks.html`
- Modify: `bookmarks-filesync.html`

**Interfaces:**
- Consumes: Task 2完了後の`renderCard(b)`(`.card-stub`グリップ追加済み)。既存の`tagBtns`クリックバインドループ(`renderSections()`内、`card.querySelectorAll('[data-action="filter-tag"]')`)は密度モードと無関係に既に存在し、本タスクでは変更不要(新しいタグボタンを自動的に拾う)。
- Produces: `renderCard(b)`にタグチップ(`.card-tags` > `.tag-stamp`)を追加したマークアップ。

- [ ] **Step 1: CSS追加 — `.card--list`内でのタグチップの詰め**

`.card--list { ... }`ブロック(Task 1で維持、Task 2で`.card-stub`行を追加済み)の末尾に、以下を追加する:

```css
  .card--list .card-tags { margin: 0; gap: 4px; }
  .card--list .tag-stamp { transform: none; }
```

両ファイルに追加する(`.card-tags`のデフォルト`margin: 10px 0 4px`・`.tag-stamp`の手書き風`transform: rotate(...)`は、縦に余白を取らない1行レイアウトでは不要なため打ち消す)。

- [ ] **Step 2: JS修正 — `renderCard(b)`にタグチップを追加**

両ファイルの`renderCard(b)`、変更前:

```js
  function renderCard(b) {
    var iconHtml = resolveIconHtml(b);
    var grip = !state.selectMode ? '<span class="stub-grip" aria-hidden="true">&#8942;</span>' : '';
    var checkboxHtml = state.selectMode
```

変更後(`tagsHtml`を追加、既存の`renderCardComfy()`と同一のタグボタン生成ロジックを再利用):

```js
  function renderCard(b) {
    var iconHtml = resolveIconHtml(b);
    var grip = !state.selectMode ? '<span class="stub-grip" aria-hidden="true">&#8942;</span>' : '';
    var tagsHtml = (b.tags || []).map(function (t) {
      return '<button type="button" tabindex="-1" class="tag-stamp clickable' + (state.tag === t ? ' active' : '') + '" data-action="filter-tag" data-tag="' + escapeHtml(t) + '">#' + escapeHtml(t) + '</button>';
    }).join('');
    var checkboxHtml = state.selectMode
```

続けて、タイトル要素の直後・`memoPreviewHtml`の前に`(tagsHtml ? '<div class="card-tags">' + tagsHtml + '</div>' : '') +`を挿入する。変更前:

```js
        '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer" tabindex="-1">' + escapeHtml(b.title) + '</a></h3>' +
        memoPreviewHtml +
```

変更後:

```js
        '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer" tabindex="-1">' + escapeHtml(b.title) + '</a></h3>' +
        (tagsHtml ? '<div class="card-tags">' + tagsHtml + '</div>' : '') +
        memoPreviewHtml +
```

両ファイルに同一の変更を適用する。既存の`tagBtns`バインドループ(`renderSections()`内)は変更不要 — `card.querySelectorAll('[data-action="filter-tag"]')`は密度モードと無関係にすべてのカードに対して実行されるため、新しく出力されたタグボタンも自動的にクリック可能になる。

- [ ] **Step 3: Node.js検証 — タグが無ければ非表示、あればクリック可能な状態で出力されることを確認する**

スクラッチディレクトリに`verify-task4.js`として保存:

```js
'use strict';
var fs = require('fs');
var assert = require('assert');
function extractFunction(src, name) {
  var m = new RegExp('function\\s+' + name + '\\s*\\(').exec(src);
  assert.ok(m, 'function not found: ' + name);
  var start = m.index, braceStart = src.indexOf('{', start), depth = 0;
  for (var i = braceStart; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) return src.slice(start, i + 1); }
  }
  throw new Error('unbalanced braces');
}
['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');
  var renderCardSrc = extractFunction(src, 'renderCard');
  var resolveIconHtmlSrc = extractFunction(src, 'resolveIconHtml');
  var resolveOpenHrefSrc = extractFunction(src, 'resolveOpenHref');
  var escapeHtmlSrc = extractFunction(src, 'escapeHtml');
  var factory = new Function('state', [escapeHtmlSrc, resolveOpenHrefSrc, resolveIconHtmlSrc, renderCardSrc].join('\n') + '\nreturn renderCard;');

  var withTags = { id: 'b1', url: 'https://example.com/', title: 'Example', scheme: 'https', domain: 'example.com', tags: ['foo', 'bar'], group: null, loginId: '', loginPassword: '', memo: '', pinned: false };
  var renderCardFn = factory({ selectMode: false, tag: null, editingId: null, selectedIds: new Set() });
  var htmlWithTags = renderCardFn(withTags);
  assert.ok(htmlWithTags.indexOf('class="card-tags"') !== -1, file + ': card-tags container missing when tags present');
  assert.ok(htmlWithTags.indexOf('data-tag="foo"') !== -1, file + ': tag button for "foo" missing');
  assert.ok(htmlWithTags.indexOf('data-action="filter-tag"') !== -1, file + ': filter-tag action missing');

  var withoutTags = { id: 'b2', url: 'https://example.com/2', title: 'Example2', scheme: 'https', domain: 'example.com', tags: [], group: null, loginId: '', loginPassword: '', memo: '', pinned: false };
  var htmlWithoutTags = renderCardFn(withoutTags);
  assert.ok(htmlWithoutTags.indexOf('class="card-tags"') === -1, file + ': card-tags container should be absent when no tags');

  console.log(file + ': OK');
});
```

Run: `node verify-task4.js`
Expected: 両ファイルで`OK`。

- [ ] **Step 4: Commit**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "$(cat <<'EOF'
feat: リスト行にタグ表記を追加(R6)

既存のtagBtnsクリックバインド(密度モード非依存)がそのまま機能するため、
JS側のイベント割り当て追加は不要。

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01621j21ZvaFTbxSaonKKkWX
EOF
)"
```

---

### Task 5: 検索対象へのメモ追加・フィルタ選択時のスクロールリセット(R4+R5)

**Files:**
- Modify: `bookmarks.html`
- Modify: `bookmarks-filesync.html`

**Interfaces:**
- Consumes: 既存の`getSearchTagFiltered()`、タグナビ/グループナビ/`#clear-all-filters-btn`のクリックハンドラ。本タスクはTask 1〜4と独立(`renderCard`/`renderFavoriteCard`を触らない)。
- Produces: `getSearchTagFiltered()`の検索対象文字列にメモを追加。3つのクリックハンドラに`window.scrollTo({ top: 0, behavior: 'auto' })`を追加。

- [ ] **Step 1: JS修正 — `getSearchTagFiltered()`の検索対象にメモを追加**

両ファイルの`getSearchTagFiltered()`、変更前:

```js
  function getSearchTagFiltered() {
    var q = state.search.trim().toLowerCase();
    return bookmarks.filter(function (b) {
      if (state.tag && (b.tags || []).indexOf(state.tag) === -1) return false;
      if (state.group !== null && (b.group || '') !== state.group) return false;
      if (q) {
        var hay = (b.title + ' ' + b.url + ' ' + (b.tags || []).join(' ')).toLowerCase();
        if (hay.indexOf(q) === -1) return false;
      }
      return true;
    });
  }
```

変更後:

```js
  function getSearchTagFiltered() {
    var q = state.search.trim().toLowerCase();
    return bookmarks.filter(function (b) {
      if (state.tag && (b.tags || []).indexOf(state.tag) === -1) return false;
      if (state.group !== null && (b.group || '') !== state.group) return false;
      if (q) {
        var hay = (b.title + ' ' + b.url + ' ' + (b.tags || []).join(' ') + ' ' + (b.memo || '')).toLowerCase();
        if (hay.indexOf(q) === -1) return false;
      }
      return true;
    });
  }
```

両ファイルに同一の変更を適用する。

- [ ] **Step 2: JS修正 — タグナビ・グループナビ・フィルタ全解除でスクロールを先頭へ**

両ファイルのタグナビclickハンドラ(`renderTagNav()`内)、変更前:

```js
    nav.querySelectorAll('button').forEach(function (btn) {
      btn.addEventListener('click', function () {
        var val = btn.getAttribute('data-value');
        state.tag = (state.tag === val) ? null : val;
        render();
      });
    });
  }

  function navItem(label, value, isActive, count) {
```

変更後:

```js
    nav.querySelectorAll('button').forEach(function (btn) {
      btn.addEventListener('click', function () {
        var val = btn.getAttribute('data-value');
        state.tag = (state.tag === val) ? null : val;
        render();
        window.scrollTo({ top: 0, behavior: 'auto' });
      });
    });
  }

  function navItem(label, value, isActive, count) {
```

グループナビclickハンドラ(`renderGroupNav()`内)、変更前:

```js
    nav.querySelectorAll('button').forEach(function (btn) {
      btn.addEventListener('click', function () {
        var val = btn.getAttribute('data-value');
        state.group = (state.group === val) ? null : val;
        render();
      });
    });
  }
```

変更後:

```js
    nav.querySelectorAll('button').forEach(function (btn) {
      btn.addEventListener('click', function () {
        var val = btn.getAttribute('data-value');
        state.group = (state.group === val) ? null : val;
        render();
        window.scrollTo({ top: 0, behavior: 'auto' });
      });
    });
  }
```

`#clear-all-filters-btn`のclickハンドラ、変更前:

```js
  document.getElementById('clear-all-filters-btn').addEventListener('click', function () {
    state.tag = null;
    state.group = null;
    render();
  });
```

変更後:

```js
  document.getElementById('clear-all-filters-btn').addEventListener('click', function () {
    state.tag = null;
    state.group = null;
    render();
    window.scrollTo({ top: 0, behavior: 'auto' });
  });
```

3箇所とも両ファイルに同一の変更を適用する。

- [ ] **Step 3: Node.js検証 — 検索対象メモ追加とスクロール呼び出しの両方を確認する**

スクラッチディレクトリに`verify-task5.js`として保存:

```js
'use strict';
var fs = require('fs');
var assert = require('assert');
function extractFunction(src, name) {
  var m = new RegExp('function\\s+' + name + '\\s*\\(').exec(src);
  assert.ok(m, 'function not found: ' + name);
  var start = m.index, braceStart = src.indexOf('{', start), depth = 0;
  for (var i = braceStart; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) return src.slice(start, i + 1); }
  }
  throw new Error('unbalanced braces');
}
['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');

  // R4: メモを検索対象文字列から実行時に検証する(state/bookmarksを差し込んで実際にフィルタする)
  var searchSrc = extractFunction(src, 'getSearchTagFiltered');
  var factory = new Function('state', 'bookmarks', searchSrc + '\nreturn getSearchTagFiltered();');

  var sampleBookmarks = [
    { id: 'b1', title: 'Foo', url: 'https://a.example/', tags: [], memo: 'uniquememotext', group: null },
    { id: 'b2', title: 'Bar', url: 'https://b.example/', tags: [], memo: '', group: null }
  ];
  var resultHit = factory({ search: 'uniquememotext', tag: null, group: null }, sampleBookmarks);
  assert.strictEqual(resultHit.length, 1, file + ': memo search should match exactly one bookmark');
  assert.strictEqual(resultHit[0].id, 'b1', file + ': memo search matched the wrong bookmark');

  var resultTitleStillWorks = factory({ search: 'Bar', tag: null, group: null }, sampleBookmarks);
  assert.strictEqual(resultTitleStillWorks.length, 1, file + ': existing title search regressed');
  assert.strictEqual(resultTitleStillWorks[0].id, 'b2', file + ': existing title search matched the wrong bookmark');

  // R5: スクロール呼び出しがハンドラ内に存在することをソース上で確認する
  assert.ok((src.match(/window\.scrollTo\(\{ top: 0, behavior: 'auto' \}\)/g) || []).length >= 3, file + ': expected at least 3 window.scrollTo calls (tag nav, group nav, clear-all-filters)');

  console.log(file + ': OK');
});
```

Run: `node verify-task5.js`
Expected: 両ファイルで`OK`。

- [ ] **Step 4: Commit**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "$(cat <<'EOF'
feat: 検索対象にメモを追加し、フィルタ選択時にスクロールを先頭へ戻す(R4,R5)

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01621j21ZvaFTbxSaonKKkWX
EOF
)"
```

---

### Task 6: キーボードショートカットの最小化(R7)

**Files:**
- Modify: `bookmarks.html`
- Modify: `bookmarks-filesync.html`

**Interfaces:**
- Consumes: 既存の`handleGlobalShortcut(e)`。`handleCardShortcut()`・`closeTopmostLayer()`・トップレベル`keydown`リスナーは無変更。
- Produces: `n`/`t`/`g`/`?`/`x`キーのcaseを削除した`handleGlobalShortcut(e)`。更新済みのヘルプパネル`<dl>`。

- [ ] **Step 1: JS修正 — `handleGlobalShortcut(e)`から5キーを削除**

両ファイルの`handleGlobalShortcut(e)`、変更前:

```js
  function handleGlobalShortcut(e) {
    switch (e.key) {
      case '/':
        e.preventDefault(); // フォーカス移動後に '/' が入力されてしまうのを防ぐ
        document.getElementById('search-input').focus();
        return true;
      case 'n':
        e.preventDefault();
        clickHeaderButton('toggle-add-btn');
        return true;
      case 't':
        e.preventDefault();
        switchSidebarTab('tag');
        var firstTagBtn = document.querySelector('#tag-nav button');
        if (firstTagBtn) firstTagBtn.focus();
        return true;
      case 'g':
        e.preventDefault();
        switchSidebarTab('group');
        var firstGroupBtn = document.querySelector('#group-nav button');
        if (firstGroupBtn) firstGroupBtn.focus();
        return true;
      case '?':
        e.preventDefault();
        toggleHelp();
        return true;
      case 'x':
        e.preventDefault();
        document.getElementById('clear-all-filters-btn').click();
        return true;
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
      case 'Home':
        e.preventDefault();
        var firstCard = getNavigableCards()[0];
        if (firstCard) firstCard.focus();
        return true;
      case 'End':
        e.preventDefault();
        var navCards = getNavigableCards();
        if (navCards.length) navCards[navCards.length - 1].focus();
        return true;
    }
    return false;
  }
```

変更後(`/`・矢印キー・Home・Endのみ残す):

```js
  function handleGlobalShortcut(e) {
    switch (e.key) {
      case '/':
        e.preventDefault(); // フォーカス移動後に '/' が入力されてしまうのを防ぐ
        document.getElementById('search-input').focus();
        return true;
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
      case 'Home':
        e.preventDefault();
        var firstCard = getNavigableCards()[0];
        if (firstCard) firstCard.focus();
        return true;
      case 'End':
        e.preventDefault();
        var navCards = getNavigableCards();
        if (navCards.length) navCards[navCards.length - 1].focus();
        return true;
    }
    return false;
  }
```

両ファイルに同一の変更を適用する。`handleCardShortcut()`・`closeTopmostLayer()`・トップレベル`keydown`リスナーは無変更。

- [ ] **Step 2: マークアップ修正 — ヘルプパネルの説明一覧を更新**

両ファイルのヘルプパネル`<dl class="help-grid">`、変更前:

```html
  <dl class="help-grid">
    <dt><kbd>/</kbd></dt><dd>検索ボックスへフォーカス</dd>
    <dt><kbd>n</kbd></dt><dd>追加パネルの開閉</dd>
    <dt><kbd>?</kbd></dt><dd>このヘルプの開閉</dd>
    <dt><kbd>x</kbd></dt><dd>タグ・グループの絞り込みをすべて解除</dd>
    <dt><kbd>Esc</kbd></dt><dd>検索欄にフォーカス中かつ入力中なら検索をクリア。それ以外はヘルプ &rarr; グループ管理 &rarr; タグ管理 &rarr; フォーム &rarr; 選択モードの順に閉じ、何も無ければフォーカスを外す</dd>
    <dt><kbd>&#8593;</kbd> <kbd>&#8595;</kbd></dt><dd>グループ間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
    <dt><kbd>&#8592;</kbd> <kbd>&#8594;</kbd></dt><dd>同じグループ内でカード間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
    <dt><kbd>Home</kbd></dt><dd>一覧の先頭カードへフォーカス</dd>
    <dt><kbd>End</kbd></dt><dd>一覧の末尾カードへフォーカス</dd>
    <dt><kbd>Enter</kbd></dt><dd>フォーカス中のカードのURLを開く</dd>
    <dt><kbd>e</kbd></dt><dd>フォーカス中のカードを編集</dd>
    <dt><kbd>Delete</kbd></dt><dd>フォーカス中のカードを削除</dd>
    <dt><kbd>t</kbd></dt><dd>タグタブに切り替えてタグ絞り込みナビへフォーカス</dd>
    <dt><kbd>g</kbd></dt><dd>グループタブに切り替えてグループ絞り込みナビへフォーカス</dd>
  </dl>
```

変更後(`n`/`?`/`x`/`t`/`g`の行を削除):

```html
  <dl class="help-grid">
    <dt><kbd>/</kbd></dt><dd>検索ボックスへフォーカス</dd>
    <dt><kbd>Esc</kbd></dt><dd>検索欄にフォーカス中かつ入力中なら検索をクリア。それ以外はヘルプ &rarr; グループ管理 &rarr; タグ管理 &rarr; フォーム &rarr; 選択モードの順に閉じ、何も無ければフォーカスを外す</dd>
    <dt><kbd>&#8593;</kbd> <kbd>&#8595;</kbd></dt><dd>グループ間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
    <dt><kbd>&#8592;</kbd> <kbd>&#8594;</kbd></dt><dd>同じグループ内でカード間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
    <dt><kbd>Home</kbd></dt><dd>一覧の先頭カードへフォーカス</dd>
    <dt><kbd>End</kbd></dt><dd>一覧の末尾カードへフォーカス</dd>
    <dt><kbd>Enter</kbd></dt><dd>フォーカス中のカードのURLを開く</dd>
    <dt><kbd>e</kbd></dt><dd>フォーカス中のカードを編集</dd>
    <dt><kbd>Delete</kbd></dt><dd>フォーカス中のカードを削除</dd>
  </dl>
```

両ファイルに同一の変更を適用する(filesyncは`bookmarks.html`から+100行の位置)。

- [ ] **Step 3: Node.js検証 — 削除対象キーのcaseが残っていないこと、ヘルプ一覧に削除対象キーの表記が残っていないことを確認する**

スクラッチディレクトリに`verify-task6.js`として保存:

```js
'use strict';
var fs = require('fs');
var assert = require('assert');
function extractFunction(src, name) {
  var m = new RegExp('function\\s+' + name + '\\s*\\(').exec(src);
  assert.ok(m, 'function not found: ' + name);
  var start = m.index, braceStart = src.indexOf('{', start), depth = 0;
  for (var i = braceStart; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) return src.slice(start, i + 1); }
  }
  throw new Error('unbalanced braces');
}
['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');

  var shortcutSrc = extractFunction(src, 'handleGlobalShortcut');
  ["case 'n':", "case 't':", "case 'g':", "case '?':", "case 'x':"].forEach(function (c) {
    assert.ok(shortcutSrc.indexOf(c) === -1, file + ': handleGlobalShortcut still has ' + c);
  });
  ["case '/':", "case 'Home':", "case 'End':", "case 'ArrowDown':"].forEach(function (c) {
    assert.ok(shortcutSrc.indexOf(c) !== -1, file + ': handleGlobalShortcut lost ' + c);
  });

  // Enter/e/Deleteはhandleglobalshortcutの対象外(handleCardShortcut側)なので無変更を確認する
  var cardShortcutSrc = extractFunction(src, 'handleCardShortcut');
  ["case 'Enter':", "case 'e':", "case 'Delete':"].forEach(function (c) {
    assert.ok(cardShortcutSrc.indexOf(c) !== -1, file + ': handleCardShortcut unexpectedly lost ' + c);
  });

  var helpMatch = /<dl class="help-grid">[\s\S]*?<\/dl>/.exec(src);
  assert.ok(helpMatch, file + ': help-grid dl not found');
  var helpHtml = helpMatch[0];
  assert.ok(helpHtml.indexOf('<kbd>n</kbd>') === -1, file + ': help panel still documents n');
  assert.ok(helpHtml.indexOf('<kbd>?</kbd>') === -1, file + ': help panel still documents ?');
  assert.ok(helpHtml.indexOf('<kbd>x</kbd>') === -1, file + ': help panel still documents x');
  assert.ok(helpHtml.indexOf('<kbd>t</kbd>') === -1, file + ': help panel still documents t');
  assert.ok(helpHtml.indexOf('<kbd>g</kbd>') === -1, file + ': help panel still documents g');
  assert.ok(helpHtml.indexOf('<kbd>/</kbd>') !== -1, file + ': help panel lost /');
  assert.ok(helpHtml.indexOf('<kbd>e</kbd>') !== -1, file + ': help panel lost e (card edit, unrelated to this task)');

  console.log(file + ': OK');
});
```

Run: `node verify-task6.js`
Expected: 両ファイルで`OK`。

- [ ] **Step 4: Commit**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "$(cat <<'EOF'
feat: キーボードショートカットを最小化(R7)

n(新規追加)/t・g(タブ切替)/?(ヘルプ)/x(フィルタ全解除)を削除。
対応するボタンのクリックで引き続き操作可能。検索(/)・矢印キー/Home/End・
カード操作(Enter/e/Delete)・Escapeチェーンは無変更。

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01621j21ZvaFTbxSaonKKkWX
EOF
)"
```
