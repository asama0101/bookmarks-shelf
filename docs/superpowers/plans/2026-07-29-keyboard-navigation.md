# キーボードのみでの操作対応 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **重要:** このプロジェクトには自動テストランナーが存在しない。各タスクのRED/GREENは`tdd-gates`の`browser-manual-e2e`プロファイル（`~/.claude/skills/tdd-gates/references/profiles/browser-manual-e2e.md`）に従い、claude-in-chromeで実ブラウザを操作してDOM状態をJS評価で確認する。

**Goal:** bookmark-shelf（`bookmarks.html` / `bookmarks-filesync.html`）にグローバルショートカットキーとカード一覧のroving tabindexを実装し、マウスを使わずに検索・追加・編集・削除・複数選択操作を完結できるようにする。

**Architecture:** 既存の「`document.addEventListener('click', ...)` + `data-action`属性 + `closest()`」委譲パターンを踏襲し、対になる`keydown`委譲リスナーを1本追加する。カード一覧には`.card[tabindex]`属性を「ロービング対象カードの目印」として使うroving tabindex機構を導入し、`render()`の全再生成に対応するため`focusedCardId`という一時変数でフォーカス位置を保持する。

**Tech Stack:** Vanilla ES5-style JS（IIFE）、素のDOM API。ビルド・パッケージマネージャ・依存追加は無し（既存方針を維持）。

## Global Constraints

- 両ファイル（`bookmarks.html` / `bookmarks-filesync.html`）へ同一パターンを適用する。スクリプト部は+301行、CSS部は+75/+76行、body部は+100行のオフセットで既存コードと一致することを確認済み（差異はfilesync固有のR4のみ）。
- 自動テストランナーは存在しない。全シナリオはclaude-in-chromeによる手動E2Eで検証する（`browser-manual-e2e`プロファイル）。
- `bookmarks-filesync.html`の「ファイル接続済み状態」はOSネイティブのファイルピッカーに依存するためブラウザ自動化から直接は作れない。ユーザーが事前に一度ファイルを開いて永続許可を与えておけば`boot()`が自動復元し、以降は自動検証できる（`bookmarks-filesync.html:1043-1047`の`queryPermission() === 'granted'`経路）。この準備が無い場合、接続済み状態が前提のシナリオは`bookmarks.html`側の検証結果とファイル内容の`diff`一致確認に代替する。
- 対象spec: `docs/superpowers/specs/2026-07-29-keyboard-navigation-design.md`（CP-A PASS済み、要件ID: R1〜R4）。

## 共通テストデータ・アサーション（全タスク共通）

`bookmarks.html`はサニタイズを通さずlocalStorageを素通しするため、以下でシード可能:

```js
localStorage.setItem('bookmarks_data', JSON.stringify([
  { id: 'k1', url: 'https://example.com/a', title: 'Alpha', tags: ['work'], group: '仕事', groupOrder: 0,
    loginId: 'u@example.com', loginPassword: 'pw', memo: null, scheme: 'https', domain: 'example.com', order: 0 },
  { id: 'k2', url: 'https://example.com/b', title: 'Bravo', tags: [], group: '仕事', groupOrder: 0,
    loginId: null, loginPassword: null, memo: null, scheme: 'https', domain: 'example.com', order: 1 },
  { id: 'k3', url: 'https://example.com/c', title: 'Charlie', tags: ['ref'], group: null, groupOrder: 0,
    loginId: null, loginPassword: null, memo: null, scheme: 'https', domain: 'example.com', order: 2 }
])); location.reload();
```

DOM順は「仕事区画（k1, k2）→ 未分類区画（k3）」。頻用アサーション式（以下`MAP`と呼ぶ）:

```js
Array.from(document.querySelectorAll('#group-sections .card'))
  .map(function (c) { return c.getAttribute('data-card-id') + ':' + c.getAttribute('tabindex'); }).join(',')
```

各タスクのDone条件に共通で含めるもの:
1. `bookmarks.html`でのシナリオ全件GREEN
2. `bookmarks-filesync.html`への同一差分適用と該当リージョンの`diff`一致確認
3. filesyncの未接続状態シナリオ（該当タスクにある場合）GREEN
4. DevToolsコンソールにエラーが出ていないこと
5. `progress.md`にRED/GREENの実行結果（評価式と返り値）を証拠として追記

---

### Task 1: グローバルショートカット基盤（`/` `n` `s` + 入力欄ガード + R4のdisabledガード）

**充足要件:** R1の`/` `n` `s`と入力欄ガード、R4全体

**Files:**
- Modify: `bookmarks.html:1726`（`});` と `render();` の間に挿入）
- Modify: `bookmarks-filesync.html:2027`（`});` と `boot();` の間に挿入）

**Interfaces:**
- Produces: `isTextInputElement(el)`、`clickHeaderButton(id)`、`document`への`keydown`委譲リスナー（Task 2, 4, 5がこの`switch`に分岐を追加する）
- Consumes: なし

- [ ] **Step 1: RED確認（実装前ベースライン）**

`file:///home/asama/bookmarks-shelf/bookmarks.html`を開き、共通テストデータをシードしてリロード後、claude-in-chromeで以下を実行し失敗（無反応）を確認する。

- `document.activeElement.blur()` → `/`押下 → `document.activeElement.id` を評価 → 期待するRED: `""`（bodyのまま）
- `n`押下 → `document.getElementById('add-panel').hidden` を評価 → 期待するRED: `true`（変化なし）
- `s`押下 → `document.getElementById('selection-bar').hidden` を評価 → 期待するRED: `true`（変化なし）

`file:///home/asama/bookmarks-shelf/bookmarks-filesync.html`をIndexedDBクリア済み（未接続）状態で開き、`document.getElementById('toggle-add-btn').disabled`が`true`であることを前提確認する。

- [ ] **Step 2: 実装**

両ファイルの該当箇所に以下を追加する（コード内容は同一）。

```js
  /* ---------- キーボード操作 ---------- */

  // 入力中のタイピングをショートカットが奪わないようにするための判定。
  // select も対象に含めるのは、並べ替えセレクトで文字キーによる項目ジャンプが効かなくなるのを避けるため。
  function isTextInputElement(el) {
    if (!el) return false;
    var tag = el.tagName;
    return tag === 'INPUT' || tag === 'TEXTAREA' || tag === 'SELECT';
  }

  // ヘッダーのボタンはfilesync版ではファイル未接続時にdisabledになる。
  // ショートカットもクリックと同じく無反応にするため、disabledを明示的に確認してから発火する。
  function clickHeaderButton(id) {
    var btn = document.getElementById(id);
    if (!btn || btn.disabled) return;
    btn.click();
  }

  document.addEventListener('keydown', function (e) {
    if (e.ctrlKey || e.metaKey || e.altKey) return; // ブラウザ標準のショートカットを妨げない
    if (isTextInputElement(e.target)) return;

    switch (e.key) {
      case '/':
        e.preventDefault(); // フォーカス移動後に '/' が入力されてしまうのを防ぐ
        document.getElementById('search-input').focus();
        return;
      case 'n':
        e.preventDefault();
        clickHeaderButton('toggle-add-btn');
        return;
      case 's':
        e.preventDefault();
        clickHeaderButton('toggle-select-btn');
        return;
    }
  });
```

- [ ] **Step 3: GREEN確認**

同じ操作を再実行し、以下を確認する。

- `/`押下後: `document.activeElement.id === "search-input"`、`value === ""`
- `n`押下後: `add-panel.hidden === false`、`document.activeElement.id === "new-url"`。再度`n`で`hidden === true`
- `s`押下後: `selection-bar.hidden === false`、`toggle-select-btn.textContent === "選択終了"`。再度`s`で`hidden === true`、`textContent === "選択"`
- 入力欄ガード: `#search-input`にフォーカスした状態で`n`,`s`,`/`を打鍵 → `search-input.value === "ns/"`、`add-panel.hidden === true`、`selection-bar.hidden === true`
- filesync未接続状態: `n`押下→`add-panel.hidden === true`のまま、`s`押下→`selection-bar.hidden === true`のまま、`/`押下→`document.activeElement.id === "search-input"`（リスナー自体は生きていることの対照確認）
- （接続済み状態を用意できる場合）接続後に`n`を押すと`add-panel.hidden === false`になること

- [ ] **Step 4: bookmarks-filesync.htmlへの同一差分適用と一致確認**

```bash
diff <(sed -n '<A>,<B>p' bookmarks.html) <(sed -n '<C>,<D>p' bookmarks-filesync.html)
```
（行番号は実装後の実測値に置き換え、空出力になることを確認）

- [ ] **Step 5: Commit**

```bash
git add bookmarks.html bookmarks-filesync.html docs/superpowers/plans/2026-07-29-keyboard-navigation.md
git commit -m "feat: add global keyboard shortcuts (/, n, s) with text-input guard"
```

---

### Task 2: ヘルプパネル（`?`）と `Escape` の段階クローズ

**充足要件:** R1の`?`と`Escape`

**Files:**
- Modify: `bookmarks.html:159`直後（CSS挿入）、`bookmarks.html:731`直後（HTML挿入）、キーボードセクション内（JS追加）
- Modify: `bookmarks-filesync.html:234`直後、`831`直後、キーボードセクション内

**Interfaces:**
- Produces: `#help-panel`要素、`helpPanel`変数、`toggleHelp()`、`closeTopmostLayer()`
- Consumes: Task 1（`keydown`リスナー）、Task 3（`.card[tabindex="0"]`によるフォーカス戻し）、既存`addPanel`・`state.editingId`/`state.editingGroup`/`state.selectMode`

**注記:** 本タスクはTask 3の後に実装する（Task 3が提供する`.card[tabindex="0"]`セレクタに`closeTopmostLayer()`が依存するため）。実装順序は本ファイル末尾「実装順序」を参照。

- [ ] **Step 1: RED確認**

- `document.getElementById('help-panel')`を評価 → 期待するRED: `null`
- `n`押下で追加パネルを開いた状態から`Escape`押下 → `add-panel.hidden`を評価 → 期待するRED: `false`（閉じない。Escape自体が未実装のため）

- [ ] **Step 2: 実装（CSS）**

`bookmarks.html:159`（`.add-panel[hidden] { display: none; }`）の直後に追加:

```css
  .help-panel {
    background: var(--card);
    border-bottom: 1px solid var(--sage);
    padding: 18px 24px;
  }
  .help-panel[hidden] { display: none; }

  .help-panel h2 {
    font-family: var(--font-display);
    font-size: 0.95rem;
    letter-spacing: 0.06em;
    margin: 0 0 10px;
  }

  .help-grid {
    display: grid;
    grid-template-columns: max-content 1fr;
    gap: 6px 16px;
    max-width: 640px;
    margin: 0;
    font-size: 0.85rem;
  }

  .help-grid dt { margin: 0; }
  .help-grid dd { margin: 0; color: var(--ink-soft); }

  .help-grid kbd {
    display: inline-block;
    padding: 1px 6px;
    border: 1px solid var(--sage);
    border-bottom-width: 2px;
    border-radius: 4px;
    background: var(--ivory);
    font-family: var(--font-mono);
    font-size: 0.78rem;
  }
```

- [ ] **Step 3: 実装（HTML）**

`bookmarks.html:731`（add-panelの`</section>`）の直後に追加:

```html
<section class="help-panel" id="help-panel" hidden>
  <h2>キーボードショートカット</h2>
  <dl class="help-grid">
    <dt><kbd>/</kbd></dt><dd>検索ボックスへフォーカス</dd>
    <dt><kbd>n</kbd></dt><dd>追加パネルの開閉</dd>
    <dt><kbd>s</kbd></dt><dd>選択モードの切り替え</dd>
    <dt><kbd>?</kbd></dt><dd>このヘルプの開閉</dd>
    <dt><kbd>Esc</kbd></dt><dd>ヘルプ &rarr; フォーム &rarr; 選択モードの順に閉じる</dd>
    <dt><kbd>&#8593;</kbd> <kbd>&#8595;</kbd></dt><dd>カード間のフォーカス移動</dd>
    <dt><kbd>Enter</kbd></dt><dd>フォーカス中のカードのURLを開く</dd>
    <dt><kbd>e</kbd></dt><dd>フォーカス中のカードを編集</dd>
    <dt><kbd>Delete</kbd></dt><dd>フォーカス中のカードを削除</dd>
    <dt><kbd>Space</kbd></dt><dd>選択モード中: フォーカス中のカードの選択を切り替え</dd>
  </dl>
</section>
```

- [ ] **Step 4: 実装（JS）**

キーボードセクションに追加（`clickHeaderButton`の後）:

```js
  var helpPanel = document.getElementById('help-panel');

  function toggleHelp() {
    helpPanel.hidden = !helpPanel.hidden;
  }

  // Escapeは「手前にあるもの」から1段階ずつ閉じる: ヘルプ → 各種フォーム(編集・追加・グループ名リネーム) → 選択モード。
  // フォームは3種をまとめて1段階として扱う(同時に開いていても1回のEscapeで閉じ切る)。
  function closeTopmostLayer() {
    if (!helpPanel.hidden) {
      helpPanel.hidden = true;
      return;
    }

    var closedEditingCard = state.editingId !== null;
    var closedForm = false;
    if (state.editingId !== null) { state.editingId = null; closedForm = true; }
    if (state.editingGroup !== null) { state.editingGroup = null; closedForm = true; }
    if (!addPanel.hidden) { addPanel.hidden = true; closedForm = true; }
    if (closedForm) {
      render();
      // 編集フォームを閉じるとフォーカスが行き場を失うため、Tabの停止位置になっているカードへ戻す
      if (closedEditingCard) {
        var target = document.querySelector('#group-sections .card[tabindex="0"]');
        if (target) target.focus();
      }
      return;
    }

    if (state.selectMode) {
      state.selectMode = false;
      state.selectedIds.clear();
      render();
    }
  }
```

`keydown`リスナーの先頭（Task 1の`if (e.ctrlKey ...) return;`の直後）に`Escape`の分岐を追加し、`isTextInputElement`ガードより前に置く（Escapeは入力欄でも常に有効にするため）:

```js
    if (e.key === 'Escape') {
      closeTopmostLayer();
      return;
    }
```

さらに`switch`に`?`の分岐を追加:

```js
      case '?':
        e.preventDefault();
        toggleHelp();
        return;
```

- [ ] **Step 5: GREEN確認**

- `?`押下 → `help-panel.hidden === false`。再度`?` → `true`
- `?`で開いた状態から`Escape` → `help-panel.hidden === true`
- `n`で追加パネルを開いた状態（`document.activeElement.id === "new-url"`）から`Escape` → `add-panel.hidden === true`（入力欄フォーカス中でも効くことの証拠）
- k1の編集ボタンクリック（`.edit-form`が存在）→`Escape` → `.edit-form`が無くなり、`document.querySelectorAll('#group-sections .card').length === 3`
- グループ「仕事」の鉛筆ボタンクリック（`.group-rename-form`が存在）→`Escape` → `.group-rename-form`が無くなる
- `s`で選択モードON → `Escape` → `selection-bar.hidden === true`
- 段階クローズ: `s`→`?`の順で開く → `Escape`1回目で`[help-panel.hidden, selection-bar.hidden] === [true, false]` → `Escape`2回目で`[true, true]`

- [ ] **Step 6: bookmarks-filesync.htmlへの同一差分適用と一致確認**

（Task 1と同様の`diff`確認）

- [ ] **Step 7: Commit**

```bash
git commit -m "feat: add help panel (?) and staged Escape close"
```

---

### Task 3: roving tabindex基盤とフォーカス保持

**充足要件:** R2のうちtabindex構造（カードルート・カード内要素の封じ込め）、R3全体

**Files:**
- Modify: `bookmarks.html:503`直後（CSS）、`renderSections`（`bookmarks.html:1170`直後・`1250`直後）、`renderCard`内9箇所（`bookmarks.html:1474,1480,1488,1492,1493,1501,1508,1517,1518`）、キーボードセクション
- Modify: `bookmarks-filesync.html`の対応箇所（+301/+75/+76オフセット）

**Interfaces:**
- Produces: `focusedCardId`（モジュールスコープ変数）、`getNavigableCards()`、`applyRovingTabindex()`、`#group-sections`の`focusin`委譲リスナー、`.card[tabindex]`規約、`.card:focus`可視スタイル
- Consumes: なし（Task 1/2と独立に実装可能）

**注記:** `renderEditCard`（`bookmarks.html:1526-1554`）は変更しない。「`tabindex`属性を持つカードだけがロービング対象」という規約により、編集中カードは自動的に除外される。

- [ ] **Step 1: RED確認**

- シード投入・リロード後、`MAP`を評価 → 期待するRED: `"k1:null,k2:null,k3:null"`
- `document.querySelector('[data-card-id="k1"]').focus()`を評価 → フォーカスが入らない（`document.activeElement.tagName === "BODY"`のまま）

- [ ] **Step 2: 実装（CSS）**

`bookmarks.html:503`（`.card.drop-after`）の直後:

```css
  .card:focus { outline: 3px solid var(--amber); outline-offset: 2px; }
```

- [ ] **Step 3: 実装（renderCard内のtabindex付与、9箇所）**

```js
    // タグボタン
      return '<button type="button" tabindex="-1" class="tag-stamp clickable' + (state.tag === t ? ' active' : '') + '" data-action="filter-tag" data-tag="' + escapeHtml(t) + '">#' + escapeHtml(t) + '</button>';

    // 選択チェックボックス
      ? '<input type="checkbox" tabindex="-1" class="card-select" data-action="select" aria-label="このブックマークを選択"' + (state.selectedIds.has(b.id) ? ' checked' : '') + '>'

    // IDコピー
            '<button type="button" tabindex="-1" class="btn-quiet" data-action="copy-login-id">コピー</button></span>'

    // パスワード表示切替 / PWコピー
            '<button type="button" tabindex="-1" class="btn-quiet" data-action="toggle-reveal">表示</button>' +
            '<button type="button" tabindex="-1" class="btn-quiet" data-action="copy-login-password">コピー</button></span>'

    // カードルート(ロービング対象の目印を兼ねる)
      '<article class="card" data-card-id="' + escapeHtml(b.id) + '" tabindex="-1">' +

    // タイトルリンク(spec更新済み: Tabで一覧を抜けるために-1が必要。URLはEnterで開く)
              '<h3 class="card-title"><a href="' + escapeHtml(b.url) + '" target="_blank" rel="noopener noreferrer" tabindex="-1">' + escapeHtml(b.title) + '</a></h3>' +

    // 編集 / 削除
              '<button type="button" tabindex="-1" class="btn-quiet" data-action="edit">編集</button>' +
              '<button type="button" tabindex="-1" class="btn-danger" data-action="delete">削除</button>' +
```

- [ ] **Step 4: 実装（renderSections、フォーカス保持）**

`bookmarks.html:1170`（`var container = ...`の直後）:

```js
    // render()でDOMを作り直すとフォーカスが失われるため、カード自身にフォーカスがあった場合だけ復元する。
    // カード内のボタンにフォーカスがある場合(編集ボタンのクリック等)は、その後の遷移を妨げないよう復元しない。
    var active = document.activeElement;
    var hadCardFocus = !!(active && active.classList && active.classList.contains('card') && container.contains(active));
```

`bookmarks.html:1250`（`bookmarks.forEach`の閉じ`});`の直後、`1251`の`}`の前）:

```js
    var rovingTarget = applyRovingTabindex();
    if (hadCardFocus && rovingTarget) rovingTarget.focus();
```

- [ ] **Step 5: 実装（キーボードセクション）**

```js
  // キーボード操作でフォーカス中のカードID。render()がDOMを作り直すため、stateではなくここで一時保持する。
  var focusedCardId = null;

  // ロービングの対象は tabindex 属性を持つカードのみ。編集フォームに差し替わったカード(renderEditCard)は
  // 属性を持たないため自動的に除外され、フォーム内inputへ自然にTab遷移できる。
  function getNavigableCards() {
    return Array.prototype.slice.call(document.querySelectorAll('#group-sections .card[tabindex]'));
  }

  // 再描画のたびにTabの停止位置(tabindex="0")を1枚だけに絞り直す。対象は直前までフォーカスしていたカード、
  // それが検索・フィルタ・削除で消えている場合と未フォーカスの場合は表示中の先頭カード。
  function applyRovingTabindex() {
    var cards = getNavigableCards();
    if (!cards.length) return null; // 表示中カードが0件のときはどのカードにもtabindex="0"を置かない
    var target = null;
    if (focusedCardId !== null) {
      target = cards.filter(function (c) { return c.getAttribute('data-card-id') === focusedCardId; })[0] || null;
    }
    if (!target) target = cards[0];
    cards.forEach(function (c) { c.setAttribute('tabindex', c === target ? '0' : '-1'); });
    return target;
  }

  // クリック・Tab・矢印キーのいずれの経路でカードがフォーカスされてもIDを覚え、Tabの停止位置も張り替える
  document.getElementById('group-sections').addEventListener('focusin', function (e) {
    var card = e.target.closest('#group-sections .card[tabindex]');
    if (!card) return;
    focusedCardId = card.getAttribute('data-card-id');
    getNavigableCards().forEach(function (c) { c.setAttribute('tabindex', c === card ? '0' : '-1'); });
  });
```

- [ ] **Step 6: GREEN確認**

- 初期tabindex: シード投入後リロード → `MAP === "k1:0,k2:-1,k3:-1"`
- カード内フォーカス可能要素の封じ込め: `Array.from(document.querySelectorAll('#group-sections .card a, #group-sections .card button, #group-sections .card input')).every(el => el.getAttribute('tabindex') === '-1')` → `true`（選択モードON時も同様）
- Tabで一覧を抜ける: `[data-card-id="k1"]`にfocus → `Tab`押下 → `!!(document.activeElement.closest && document.activeElement.closest('#group-sections .card'))` → `false`
- フォーカス保持とフォールバック: k2をクリック→`MAP === "k1:-1,k2:0,k3:-1"` → `#work`タグクリックで絞り込み（k2が消える）→`MAP`の先頭が`"k1:0"` → 絞り込み解除で`MAP === "k1:-1,k2:0,k3:-1"`に戻る
- 編集中カードの除外: k1編集ボタンクリック → `document.querySelector('[data-card-id="k1"]').hasAttribute('tabindex') === false`、ロービング対象カード数`=== 2`、Tabで編集フォーム内inputを順に辿れる
- 表示中0件: 検索に`zzz`と入力 → ロービング対象カード数`=== 0`、コンソールエラーなし

- [ ] **Step 7: bookmarks-filesync.htmlへの同一差分適用と一致確認**

- [ ] **Step 8: Commit**

```bash
git commit -m "feat: add roving tabindex and focus persistence for card list"
```

---

### Task 4: カード内ショートカット（`↑` `↓` `Enter` `e` `Space`）

**充足要件:** R2の削除を除くキー操作

**Files:**
- Modify: `bookmarks.html`キーボードセクション（`keydown`の`switch`後に追記、関数追加）
- Modify: `bookmarks-filesync.html`の対応箇所

**Interfaces:**
- Produces: `moveCardFocus(card, direction)`、`keydown`リスナー内のカード分岐`switch`（Task 5がここに`Delete`/`Backspace`を追加）
- Consumes: Task 1（`keydown`リスナー）、Task 3（`getNavigableCards()`、`.card[tabindex]`、`focusin`ハンドラ）

- [ ] **Step 1: RED確認**

- `[data-card-id="k1"]`にfocus → `ArrowDown`押下 → `document.activeElement.getAttribute('data-card-id')` → 期待するRED: `"k1"`（変化なし）
- 同上で`Enter`押下 → URLが開かれない
- `e`押下 → `.edit-form`が現れない

- [ ] **Step 2: 実装**

キーボードセクションに関数追加（`applyRovingTabindex`の後）:

```js
  // 表示中の全カードをDOM順の1本のリストとして扱い、前後のカードへフォーカスを移す(区画をまたぐ移動も含む)。
  // tabindexの張り替えとfocusedCardIdの更新はfocusinハンドラ側で行われる。
  function moveCardFocus(card, direction) {
    var cards = getNavigableCards();
    var next = cards[cards.indexOf(card) + direction];
    if (next) next.focus();
  }
```

`keydown`リスナー末尾（Task 1の`switch`の閉じ括弧の後）に追記:

```js
    var card = e.target.closest('#group-sections .card[tabindex]');
    if (!card) return;
    var cardId = card.getAttribute('data-card-id');

    switch (e.key) {
      case 'ArrowDown':
      case 'ArrowUp':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocus(card, e.key === 'ArrowDown' ? 1 : -1);
        return;
      case 'Enter':
        e.preventDefault();
        // クリック時と完全に同じ経路(target=_blank rel=noopener)で開くため、既存のリンクを踏む
        var link = card.querySelector('.card-title a');
        if (link) link.click();
        return;
      case 'e':
        e.preventDefault();
        state.editingId = cardId;
        render();
        // render()内のフォーカス復元より後に呼ぶことで、編集フォームの先頭inputへフォーカスを渡す
        var urlInput = document.getElementById('edit-url');
        if (urlInput) urlInput.focus();
        return;
      case ' ':
        if (!state.selectMode) return; // 選択モード外ではSpaceの既定動作(スクロール)を残す
        e.preventDefault();
        if (state.selectedIds.has(cardId)) state.selectedIds.delete(cardId);
        else state.selectedIds.add(cardId);
        render();
        return;
    }
```

- [ ] **Step 3: GREEN確認**

- `↓`移動（区画またぎ含む）: k1→`ArrowDown`→`"k2"`→`ArrowDown`→`"k3"`（`MAP === "k1:-1,k2:-1,k3:0"`）→末尾で`ArrowDown`しても`"k3"`のまま
- `↑`移動: k3→`ArrowUp`×2→`"k1"`→先頭でさらに`ArrowUp`しても`"k1"`のまま
- `Enter`: クリックを`ev.preventDefault()`して`href`を捕捉するシムを仕込み、k1にfocus→`Enter`→捕捉したURLが`"https://example.com/a"`
- `e`: k2にfocus→`e`押下→`.edit-form`が存在、`document.activeElement.id === "edit-url"`。続けて`Escape`で閉じ、`document.activeElement.getAttribute('data-card-id') === "k2"`に戻ることも確認（Task 2との結合確認）
- `Space`（選択モード中）: `s`押下→k1にfocus→`Space`→`selection-count.textContent === "1 件選択中"`、チェック済みチェックボックス数`=== 1`、`document.activeElement.getAttribute('data-card-id') === "k1"`（render()を跨いでフォーカスが残ることの実証）→再度`Space`で`"0 件選択中"`
- `Space`（選択モード外）: k1にfocus→`Space`→`selection-bar.hidden === true`のまま、チェックボックス数`=== 0`（既定のスクロール動作を妨げない）

- [ ] **Step 4: bookmarks-filesync.htmlへの同一差分適用と一致確認**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat: add card-level shortcuts (arrows, Enter, e, Space)"
```

---

### Task 5: `Delete` / `Backspace` による削除とフォーカス受け渡し

**充足要件:** R2の削除行（削除後のフォーカス移動先ルールを含む）

**Files:**
- Modify: `bookmarks.html:1593-1599`（`handleDelete`に戻り値追加）、キーボードセクション（関数追加+`switch`に分岐追加）
- Modify: `bookmarks-filesync.html:1894-1900`、対応するキーボードセクション

**Interfaces:**
- Produces: `deleteCardWithFocusHandoff(card)`、`handleDelete(id)`の戻り値（`boolean`）
- Consumes: Task 3（`getNavigableCards()`、`focusedCardId`）、Task 4（カード分岐`switch`）、既存`handleDelete`

- [ ] **Step 1: RED確認**

`window.confirm = function () { return true; };`を評価してから、k2にfocus→`Delete`押下→`JSON.parse(localStorage.getItem('bookmarks_data')).length`を評価 → 期待するRED: `3`（削除されない）

- [ ] **Step 2: 実装**

`handleDelete`に戻り値を追加（`bookmarks.html:1593-1599`）:

```js
  // 削除が実際に行われたかを返す(キーボード操作側が、確認ダイアログのキャンセル時にフォーカス移動先を巻き戻すため)
  function handleDelete(id) {
    if (!confirm('このブックマークを削除しますか?')) return false;
    bookmarks = bookmarks.filter(function (b) { return b.id !== id; });
    state.selectedIds.delete(id);
    saveData(bookmarks);
    render();
    return true;
  }
```

キーボードセクションに関数追加（`moveCardFocus`の後）:

```js
  // 削除後のフォーカス移動先は「次のカード」、無ければ「前のカード」、一覧が空になる場合はどこにもフォーカスしない。
  // handleDelete()がrender()を内部で呼ぶため、移動先IDは削除の前に確定させておく必要がある。
  function deleteCardWithFocusHandoff(card) {
    var cards = getNavigableCards();
    var pos = cards.indexOf(card);
    var neighbor = cards[pos + 1] || cards[pos - 1] || null;
    var previousFocusedCardId = focusedCardId;

    focusedCardId = neighbor ? neighbor.getAttribute('data-card-id') : null;
    if (!handleDelete(card.getAttribute('data-card-id'))) {
      focusedCardId = previousFocusedCardId; // 確認ダイアログでキャンセルされたら元に戻す
      return;
    }
    // confirm()の表示でフォーカスが外れるブラウザ挙動に左右されないよう、削除確定後に明示的に移す
    var restored = document.querySelector('#group-sections .card[tabindex="0"]');
    if (restored) restored.focus();
  }
```

Task 4のカード`switch`に分岐追加:

```js
      case 'Delete':
      case 'Backspace':
        e.preventDefault(); // 一部環境でBackspaceが「戻る」に割り当たるのを防ぐ
        deleteCardWithFocusHandoff(card);
        return;
```

- [ ] **Step 3: GREEN確認**

（confirm()はページ上で`window.confirm = function () { return true/false; };`と差し替えて制御する）

- 削除後は次のカードへ: `window.confirm=true`化→k2にfocus→`Delete`→localStorageのid一覧が`"k1,k3"`、`document.activeElement.getAttribute('data-card-id') === "k3"`、`MAP === "k1:-1,k3:0"`
- 末尾カード削除は前のカードへ: シード再投入→k3にfocus→`Backspace`→id一覧`"k1,k2"`、フォーカス`"k2"`
- 一覧が空になる場合: 1件だけのシードでk1を削除→カード数`0`、`document.activeElement === document.body`、コンソールエラーなし。続けて`ArrowDown`/`ArrowUp`を押しても何も起きない
- キャンセル時: `window.confirm=false`化→k2にfocus→`Delete`→データ件数`3`のまま、フォーカスが`"k2"`のまま、`MAP === "k1:-1,k2:0,k3:-1"`（`focusedCardId`巻き戻しの実証）
- マウス操作の非退行: k1の削除ボタンをクリック→localStorageのid一覧が`"k2,k3"`

- [ ] **Step 4: bookmarks-filesync.htmlへの同一差分適用と一致確認**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat: add Delete/Backspace card removal with focus handoff"
```

---

## 実装順序

```
Task 1 (グローバル /,n,s + 入力ガード + R4)
Task 3 (roving tabindex + フォーカス保持)   ← Task1と独立、並行実装可
Task 2 (ヘルプ ? + Escape 段階クローズ)      ← Task3の .card[tabindex="0"] に依存
Task 4 (↑↓ Enter e Space)                   ← Task1のリスナー + Task3のセレクタが必要
Task 5 (Delete/Backspace + フォーカス受け渡し) ← Task4の直後
```

推奨実行順: **Task 1 → Task 3 → Task 2 → Task 4 → Task 5**。

## 完了後（受け入れ確認後にdoc-updaterへ委任）

全タスク完了後、`CLAUDE.md`のArchitecture節に第6のレイヤー「キーボード操作 — `keydown`委譲リスナー、`.card[tabindex]`によるロービング対象の表現、`focusedCardId`による再描画跨ぎのフォーカス保持」を追記する。
