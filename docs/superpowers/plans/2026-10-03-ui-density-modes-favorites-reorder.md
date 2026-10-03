# 情報密度モード切替・お気に入り並び替え Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `bookmarks.html`・`bookmarks-filesync.html`に3段階の情報密度モード（ゆったり／コンパクト／リスト）と、お気に入りセクション専用のドラッグ並び替え（`favoriteOrder`）を追加する。

**Architecture:** 両ファイルとも単一IIFE内のレンダー関数（`renderCard`/`renderFavoriteCard`）をモード別のディスパッチャに変える。密度モードはCSS修飾クラス（`card--compact`/`card--list`/`group-grid--compact`/`group-grid--list`）＋レンダー内容の分岐で表現し、既存のキーボードナビゲーション（`getNavigableCards()`）・イベント委譲（`data-card-id`ベースのループ）が依存する`class="card"`・`data-card-id`・`tabindex`属性は全モードで維持する。お気に入りの並び替えは既存のカード/グループドラッグ処理（`bindDragEvents`/`bindGroupSectionDragEvents`）を一切流用せず、汎用ユーティリティ`bindDragReorder()`を使って独立した`bindFavoriteDragEvents()`を新設する。

**Tech Stack:** Vanilla ES5-style JS（IIFE）、インラインCSS。ビルド・依存ゼロ。自動テストランナーは存在しない。

**Spec:** `docs/superpowers/specs/2026-10-02-ui-density-modes-favorites-reorder-design.md`

## Global Constraints

- 対象ファイルは`bookmarks.html`・`bookmarks-filesync.html`の両方。両ファイルは大半のCSS/レンダリングロジックを重複実装しており、本機能は書き込み頻度を増やさない表示系機能のためファイル限定は行わない（前タスクの「よく使う順」のような`bookmarks.html`限定はしない）。
- 密度モードは`class="card"`・`data-card-id`・`tabindex`属性を全モードで維持し、修飾クラスの追加とレンダー内容の分岐だけで実装する（`getNavigableCards()`等の既存キーボードナビゲーション機構を壊さないため）。
- 既存のドラッグ&ドロップ無効化条件（`bookmarks.html`のみ: `state.sortMode === 'frequency'`のとき）は本タスクで変更しない。密度モード自体はドラッグ可否に関与しない。
- `favoriteOrder`マイグレーションの判定条件は必ず`pinned === true`を併せて絞り込む（`typeof b.favoriteOrder !== 'number'`単独では未ピン留めレコードに対して毎回条件が真になり続け、ユーザーの並び替えを上書きし続けるバグになる）。
- お気に入りセクションのドラッグ処理は、既存のカード/グループドラッグ処理（`bindDragEvents`/`bindGroupSectionDragEvents`、いずれも`data-card-id`ベース）を流用せず独立実装とする。ただし両者が内部で使っている汎用プリミティブ`bindDragReorder(el, key, opts)`はお気に入り側でも再利用してよい（カード固有の処理ではない）。
- 自動テストランナーが存在しないため（`CLAUDE.md`明記）、各タスクの検証は「対象関数をファイルから抽出し、Node.jsで最小限のスタブ（`document`/`localStorage`等）とともに実行して期待どおりの出力・副作用を確認する」方式で行う（前タスクでユーザー承認済みの代替検証手段）。検証スクリプトは`.superpowers/sdd/<ワークスペース>/scratch/`配下に置き、リポジトリにはコミットしない（プロジェクトに自動テストスイートという概念自体が無いため）。claude-in-chromeが利用可能なセッションでは、追加でブラウザ目視確認を行ってよい（必須ではない）。
- CSS変数は既存の`:root`定義（`--ivory`/`--card`/`--ink`/`--ink-soft`/`--amber`/`--amber-ink`/`--wine`/`--sage`/`--shadow`/`--header-h`/`--font-display`/`--font-body`/`--font-mono`）をそのまま使う。新規パレットは追加しない。
- CIステージ整備（CP-E）はスキップする。本プロジェクトにCI/ビルドパイプラインが存在しないため（前タスクのplanと同じ扱い）。

## Review Focus

- **favoriteOrderマイグレーションの非冪等性**: 未ピン留めレコードが1件でも存在する状態で繰り返しロードしても、2回目以降は`needsFavoriteOrderMigration`が偽になり再計算されないこと（Task 1のテストで検証）。
- **値が無い項目のインジケーターがクリック不可であること**: コンパクト/リストモードでID・パスワードが無いブックマークをレンダーした結果に`data-action="copy-login-id"`/`data-action="copy-login-password"`が一切含まれないこと（存在すれば既存のイベント割り当てループが誤ってクリック可能なボタンとして拾ってしまう）（Task 3のテストで検証）。
- **リストモードでもタイトルクリックのアクセス記録が機能すること**: `renderCardList()`の出力に`.card-title a`セレクタで拾える`<a>`要素が含まれ、既存の`renderSections()`内イベント割り当てループ（`card.querySelector('.card-title a')`）がそのまま機能すること（Task 4のテストで検証）。
- **密度モードはドラッグ可否に影響しないこと**: `bookmarks.html`の`renderSections()`内`bindDragEvents`/`bindGroupSectionDragEvents`の呼び出し条件式に`state.densityMode`への参照が一切追加されていないこと（Task 3のテストで検証、ソースの静的grep確認）。
- **お気に入りの並び替えが元のグループ内`order`を変更しないこと**: `moveFavorite()`を実行した前後で、対象外の非ピン留めブックマークは当然として、並び替えた当人の`order`・`group`フィールドも不変であり、`favoriteOrder`のみが更新されること（Task 5のテストで検証）。

---

### Task 1: favoriteOrderデータモデル拡張とマイグレーション（R1）

**Files:**
- Modify: `bookmarks.html:1204-1233`（`loadData()`）、`bookmarks.html:1353-1384`（`sanitizeBookmark()`）、`bookmarks.html:1398-1406`直後（新規`getNextFavoriteOrder()`）、`bookmarks.html:1852-1860`（`renderSections()`内ピン留めトグル）、`bookmarks.html:1780-1785`（`renderFavoritesSection()`内unpin）、`bookmarks.html:2499-2540`（追加フォーム送信ハンドラ）
- Modify: `bookmarks-filesync.html:1656-1688`（`sanitizeBookmark()`）、`bookmarks-filesync.html:1363-1382`（`readBookmarksFromHandle()`）、`bookmarks-filesync.html:1692-1700`直後（新規`assignFavoriteOrder()`）、`bookmarks-filesync.html:2852-2885`（JSONインポート）、`bookmarks-filesync.html:2129-2137`（`renderSections()`内ピン留めトグル）、`bookmarks-filesync.html:2058-2063`（`renderFavoritesSection()`内unpin）、`bookmarks-filesync.html:2760-2801`（追加フォーム送信ハンドラ）

**Interfaces:**
- Produces: `bookmark.favoriteOrder`（`number | null`）フィールド。`getNextFavoriteOrder()`（両ファイル、戻り値`number`）。`assignFavoriteOrder(list)`（`bookmarks-filesync.html`のみ、副作用で`list`内のピン留め済み要素に`favoriteOrder`を連番で付与）。
- Consumes: 既存の`bookmarks`配列（モジュールスコープ）、`saveData()`、`render()`。

- [ ] **Step 1: `bookmarks.html`の`sanitizeBookmark()`を修正**

`bookmarks.html:1375-1382`の戻り値オブジェクトを以下に置き換える（`accessCount`行の末尾にカンマを追加し、新フィールドを足す）。

```js
      return {
        id: makeId(),
        url: url,
        title: title,
        tags: tags,
        group: group,
        groupOrder: typeof raw.groupOrder === 'number' ? raw.groupOrder : undefined,
        loginId: loginId,
        loginPassword: loginPassword,
        memo: memo,
        scheme: scheme,
        domain: deriveDomain(url, scheme),
        order: 0,
        pinned: typeof raw.pinned === 'boolean' ? raw.pinned : false,
        lastAccessed: typeof raw.lastAccessed === 'number' ? raw.lastAccessed : null,
        accessCount: typeof raw.accessCount === 'number' && raw.accessCount >= 0 ? Math.floor(raw.accessCount) : 0,
        favoriteOrder: typeof raw.favoriteOrder === 'number' ? raw.favoriteOrder : null
      };
```

- [ ] **Step 2: `bookmarks-filesync.html`の`sanitizeBookmark()`に同じ変更を加える**

`bookmarks-filesync.html:1678-1685`の戻り値オブジェクトを、Step 1と同一の内容に置き換える（コード本文はStep 1と完全に同一）。

- [ ] **Step 3: 両ファイルに`getNextFavoriteOrder()`を追加**

`bookmarks.html:1406`（`getGroupOrderForName()`終了直後）に挿入:

```js
    // お気に入り内の新規ピン留めは常に末尾(既存最大値+1)に配置する。ピン留めが1件も無ければ0。
    function getNextFavoriteOrder() {
      var max = -1;
      bookmarks.forEach(function (b) { if (b.pinned && typeof b.favoriteOrder === 'number' && b.favoriteOrder > max) max = b.favoriteOrder; });
      return max + 1;
    }
```

`bookmarks-filesync.html:1710`（`getGroupOrderForName()`終了直後）に同一のコードを挿入する。

- [ ] **Step 4: `bookmarks.html`の`loadData()`にfavoriteOrderマイグレーションを追加**

`bookmarks.html:1218-1226`の`needsFavFieldsMigration`ブロック直後、`return parsed;`の手前（1228-1232行目）を以下に置き換える。

```js
        // お気に入り並び替え機能の導入前に保存されたデータには favoriteOrder が無いため、ピン留め済みのものだけ
        // 現在の order(手動並び順)を基準に連番を割り当てる。pinned===false のものは対象外(favoriteOrder は null のまま)。
        // 判定条件に pinned===true を必ず併せること — typeof チェックだけだと未ピン留めレコードが永久に
        // 条件を満たし続け、毎回のロードでドラッグ並び替えた値を上書きしてしまう(design doc R1参照)。
        var needsFavoriteOrderMigration = parsed.some(function (b) { return b.pinned === true && typeof b.favoriteOrder !== 'number'; });
        if (needsFavoriteOrderMigration) {
          var pinnedByOrder = parsed.filter(function (b) { return b.pinned === true; })
            .sort(function (a, b) { return (a.order || 0) - (b.order || 0); });
          pinnedByOrder.forEach(function (b, i) { b.favoriteOrder = i; });
        }

        if (needsOrderMigration || needsGroupOrderMigration || needsFavFieldsMigration || needsFavoriteOrderMigration) localStorage.setItem(STORAGE_KEY, JSON.stringify(parsed));
        return parsed;
```

- [ ] **Step 5: `bookmarks-filesync.html`に`assignFavoriteOrder()`を追加**

`bookmarks-filesync.html:1700`（`assignGroupOrder()`終了直後）に挿入:

```js
    // favoriteOrder が1件でも欠けているピン留め済みレコードがあれば、現在の order(手動並び順)を基準に
    // ピン留め済みのものだけ連番で補完する。pinned===false のものは対象外(favoriteOrder は null のまま)。
    // assignGroupOrder() と同様、後方互換のための一度きりの補完。以後は各操作が明示的に維持する。
    function assignFavoriteOrder(list) {
      var pinnedByOrder = list.filter(function (b) { return b.pinned === true; })
        .sort(function (a, b) { return (a.order || 0) - (b.order || 0); });
      pinnedByOrder.forEach(function (b, i) { b.favoriteOrder = i; });
    }
```

- [ ] **Step 6: `bookmarks-filesync.html`の`readBookmarksFromHandle()`にマイグレーション呼び出しを追加**

`bookmarks-filesync.html:1377-1380`を以下に置き換える。

```js
        var sanitized = parsed.map(sanitizeBookmark).filter(Boolean);
        sanitized.forEach(function (b, i) { b.order = i; });
        if (sanitized.some(function (b) { return typeof b.groupOrder !== 'number'; })) assignGroupOrder(sanitized);
        if (sanitized.some(function (b) { return b.pinned === true && typeof b.favoriteOrder !== 'number'; })) assignFavoriteOrder(sanitized);
        return { data: sanitized, skipped: parsed.length - sanitized.length };
```

- [ ] **Step 7: `bookmarks-filesync.html`のJSONインポートハンドラにも同じ呼び出しを追加**

`bookmarks-filesync.html:2869-2871`を以下に置き換える。

```js
        var sanitized = parsed.map(sanitizeBookmark).filter(Boolean);
        sanitized.forEach(function (b, i) { b.order = i; });
        if (sanitized.some(function (b) { return typeof b.groupOrder !== 'number'; })) assignGroupOrder(sanitized);
        if (sanitized.some(function (b) { return b.pinned === true && typeof b.favoriteOrder !== 'number'; })) assignFavoriteOrder(sanitized);
```

- [ ] **Step 8: 新規ブックマーク追加フォームの送信ハンドラに`favoriteOrder: null`を追加**

`bookmarks.html:2527`の`accessCount: 0`行を次の2行に置き換える。

```js
        accessCount: 0,
        favoriteOrder: null
```

`bookmarks-filesync.html:2788`の`accessCount: 0`行にも同一の変更を加える。

- [ ] **Step 9: ピン留めトグル処理を更新（カード側・お気に入り側、両ファイル）**

`bookmarks.html:1856-1860`（`renderSections()`内）を以下に置き換える。

```js
        if (pinBtn) pinBtn.addEventListener('click', function () {
          b.pinned = !b.pinned;
          b.favoriteOrder = b.pinned ? getNextFavoriteOrder() : null;
          saveData(bookmarks);
          render();
        });
```

`bookmarks.html:1780-1785`（`renderFavoritesSection()`内のunpin）を以下に置き換える。

```js
        var unpinBtn = el.querySelector('[data-action="unpin"]');
        if (unpinBtn) unpinBtn.addEventListener('click', function () {
          b.pinned = false;
          b.favoriteOrder = null;
          saveData(bookmarks);
          render();
        });
```

`bookmarks-filesync.html:2129-2137`・`bookmarks-filesync.html:2058-2063`にも同一の変更を加える（コード本文は上記と完全に同一）。

- [ ] **Step 10: 検証スクリプトを作成し実行**

`.superpowers/sdd/<ワークスペース>/scratch/verify-task1.js`を作成する（`sdd-workspace`スクリプトが出力するワークスペースパス配下）。

```js
// bookmarks.html から loadData() のマイグレーションロジック部分を抽出して検証する。
// 実ファイルからsedで該当関数を切り出し、localStorageスタブとともにevalする。
var fs = require('fs');
var src = fs.readFileSync('bookmarks.html', 'utf8');

function extractFn(name) {
  var startIdx = src.indexOf('function ' + name + '(');
  if (startIdx === -1) throw new Error(name + ' not found');
  var braceDepth = 0, i = src.indexOf('{', startIdx), start = i;
  for (; i < src.length; i++) {
    if (src[i] === '{') braceDepth++;
    else if (src[i] === '}') { braceDepth--; if (braceDepth === 0) { i++; break; } }
  }
  return src.slice(startIdx, i);
}

var STORAGE_KEY = 'bookmarks_data';
global.localStorage = (function () {
  var store = {};
  return {
    getItem: function (k) { return Object.prototype.hasOwnProperty.call(store, k) ? store[k] : null; },
    setItem: function (k, v) { store[k] = String(v); }
  };
})();

function assignGroupOrder(list) { list.forEach(function (b) { b.groupOrder = b.groupOrder || 0; }); }

eval(extractFn('loadData').replace('function loadData', 'var loadData = function'));

// ケース1: pinned=true だが favoriteOrder 欠損 → 連番が付与される
var dataset1 = [
  { id: 'a', order: 2, pinned: true },
  { id: 'b', order: 0, pinned: true },
  { id: 'c', order: 1, pinned: false }
];
localStorage.setItem(STORAGE_KEY, JSON.stringify(dataset1));
var result1 = loadData();
var byId1 = {}; result1.forEach(function (b) { byId1[b.id] = b; });
console.assert(byId1.b.favoriteOrder === 0, 'order=0のbが先頭(favoriteOrder=0)になるはず: got ' + byId1.b.favoriteOrder);
console.assert(byId1.a.favoriteOrder === 1, 'order=2のaが2番目(favoriteOrder=1)になるはず: got ' + byId1.a.favoriteOrder);
console.assert(byId1.c.favoriteOrder === null, '非ピン留めcはfavoriteOrderがnullのままのはず: got ' + byId1.c.favoriteOrder);

// ケース2(冪等性): 1回目のロード結果を再度ロードしても favoriteOrder が変わらない(再計算されない)
localStorage.setItem(STORAGE_KEY, JSON.stringify(result1));
var result2 = loadData();
var byId2 = {}; result2.forEach(function (b) { byId2[b.id] = b; });
console.assert(byId2.b.favoriteOrder === 0 && byId2.a.favoriteOrder === 1, '2回目のロードでfavoriteOrderが変わらないはず(冪等性): got b=' + byId2.b.favoriteOrder + ' a=' + byId2.a.favoriteOrder);

// ケース3: ユーザーが並び替えた後の値(favoriteOrderが逆転)を再ロードしても上書きされない
var dataset3 = [
  { id: 'a', order: 2, pinned: true, favoriteOrder: 5 },
  { id: 'b', order: 0, pinned: true, favoriteOrder: 2 }
];
localStorage.setItem(STORAGE_KEY, JSON.stringify(dataset3));
var result3 = loadData();
var byId3 = {}; result3.forEach(function (b) { byId3[b.id] = b; });
console.assert(byId3.a.favoriteOrder === 5 && byId3.b.favoriteOrder === 2, 'ユーザーが設定したfavoriteOrderは上書きされないはず: got a=' + byId3.a.favoriteOrder + ' b=' + byId3.b.favoriteOrder);

console.log('OK: Task 1 migration verification passed');
```

Run: `cd /home/asama/bookmarks-shelf && node .superpowers/sdd/<ワークスペース>/scratch/verify-task1.js`

Expected: `OK: Task 1 migration verification passed`が出力され、`console.assert`の失敗（`Assertion failed`のstderr出力）が1件も無いこと。

- [ ] **Step 11: コミット**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "feat: favoriteOrderフィールドとマイグレーションを追加(お気に入り並び替えの基盤)"
```

---

### Task 2: 密度モードトグルUI（R2）

**Files:**
- Modify: `bookmarks.html:1181-1200`（`SORT_MODE_KEY`/`state`付近）、`bookmarks.html:1014-1029`（ヘッダーHTML）、`bookmarks.html:2485-2489`付近（JSハンドラ）、`bookmarks.html:1495-1500`付近（`render()`内）、CSS（`:root`直後またはボタン群の近く）
- Modify: `bookmarks-filesync.html:1096-1104`（ヘッダーHTML）、filesync側の`state`初期化部分、同様のJSハンドラ・render()同期・CSS

**Interfaces:**
- Produces: `state.densityMode`（`'comfy' | 'compact' | 'list'`）。`loadDensityMode()`（両ファイル、戻り値`string`）。`DENSITY_MODE_KEY`定数。
- Consumes: なし（新規の独立した状態）。Task 3・4が`state.densityMode`を読む。

- [ ] **Step 1: `bookmarks.html`に`DENSITY_MODE_KEY`・`loadDensityMode()`・`state.densityMode`を追加**

`bookmarks.html:1181`（`SORT_MODE_KEY`定義の直前）に挿入:

```js
    var DENSITY_MODE_KEY = 'bookmarks_density_mode';

    function loadDensityMode() {
      try {
        var v = localStorage.getItem(DENSITY_MODE_KEY);
        return (v === 'comfy' || v === 'compact' || v === 'list') ? v : 'compact';
      } catch (e) { return 'compact'; }
    }
```

`bookmarks.html:1193-1200`の`state`オブジェクトリテラルの`sortMode: loadSortMode()`行の直後に`,`を追加し、次の行を足す。

```js
      sortMode: loadSortMode(),
      densityMode: loadDensityMode()
```

- [ ] **Step 2: `bookmarks-filesync.html`に同じ定数・関数・stateプロパティを追加**

`bookmarks-filesync.html`の`state`初期化部分（`sortMode`が存在しないため`sidebarTab: 'group'`等の末尾付近）に、Step 1と同一の`DENSITY_MODE_KEY`・`loadDensityMode()`を追加し、`state`オブジェクトリテラルに`densityMode: loadDensityMode()`を追加する（カンマの付け方は既存の最終プロパティに合わせる）。

- [ ] **Step 3: `bookmarks.html`のヘッダーに密度トグルUIを追加**

`bookmarks.html:1023`（`<button class="btn btn-ghost" id="toggle-sort-mode-btn" type="button">よく使う順で表示</button>`）の直後に挿入:

```html
      <div class="density-toggle" id="density-toggle" role="group" aria-label="表示密度">
        <button type="button" class="density-btn" data-density="comfy">ゆったり</button>
        <button type="button" class="density-btn" data-density="compact">コンパクト</button>
        <button type="button" class="density-btn" data-density="list">リスト</button>
      </div>
```

- [ ] **Step 4: `bookmarks-filesync.html`のヘッダーに同じUIを追加**

`bookmarks-filesync.html:1102`（`<button class="btn btn-ghost" id="import-btn" type="button" disabled>インポート</button>`）の直後、`<input type="file" id="import-file" ...>`の手前に挿入:

```html
      <div class="density-toggle" id="density-toggle" role="group" aria-label="表示密度">
        <button type="button" class="density-btn" data-density="comfy">ゆったり</button>
        <button type="button" class="density-btn" data-density="compact">コンパクト</button>
        <button type="button" class="density-btn" data-density="list">リスト</button>
      </div>
```

- [ ] **Step 5: 両ファイルにクリックハンドラを追加**

`bookmarks.html:2489`（`#toggle-sort-mode-btn`のクリックハンドラの直後）に挿入:

```js
    document.getElementById('density-toggle').addEventListener('click', function (e) {
      var btn = e.target.closest('[data-density]');
      if (!btn) return;
      state.densityMode = btn.getAttribute('data-density');
      try { localStorage.setItem(DENSITY_MODE_KEY, state.densityMode); } catch (err) {}
      render();
    });
```

`bookmarks-filesync.html`の対応する位置（`#toggle-select-btn`等、既存ヘッダーボタンのクリックハンドラが並ぶ箇所の近く）に同一のコードを追加する。

- [ ] **Step 6: `render()`にアクティブ状態の同期を追加**

`bookmarks.html:1495`（`sortModeBtn.classList.toggle(...)`の行の直後）に挿入:

```js
      document.querySelectorAll('#density-toggle [data-density]').forEach(function (btn) {
        btn.classList.toggle('active', btn.getAttribute('data-density') === state.densityMode);
      });
```

`bookmarks-filesync.html:1788`（`selectBtn.classList.toggle(...)`の行の直後、`syncHeaderHeight()`の手前）に同一のコードを追加する。

- [ ] **Step 7: CSSを追加**

両ファイルの`.btn-ghost`ルール群の直後に挿入:

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

- [ ] **Step 8: 検証スクリプトを作成し実行**

`.superpowers/sdd/<ワークスペース>/scratch/verify-task2.js`:

```js
var fs = require('fs');
['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');

  console.assert(/var DENSITY_MODE_KEY = 'bookmarks_density_mode';/.test(src), file + ': DENSITY_MODE_KEY定義が無い');
  console.assert(/function loadDensityMode\(\)/.test(src), file + ': loadDensityMode()が無い');
  console.assert(/densityMode: loadDensityMode\(\)/.test(src), file + ': state.densityModeの初期化が無い');
  console.assert(/id="density-toggle"/.test(src), file + ': density-toggleのHTMLが無い');
  console.assert(/data-density="comfy"/.test(src) && /data-density="compact"/.test(src) && /data-density="list"/.test(src), file + ': 3モード分のボタンが揃っていない');

  // loadDensityMode()単体の動作確認(不正値・欠損時にcompactへフォールバックするか)
  var startIdx = src.indexOf('function loadDensityMode(');
  var i = src.indexOf('{', startIdx), depth = 0;
  for (; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) { i++; break; } }
  }
  var fnSrc = src.slice(startIdx, i).replace('function loadDensityMode', 'var loadDensityMode = function');

  [null, '', 'bogus', 'comfy', 'compact', 'list'].forEach(function (stored) {
    global.localStorage = { getItem: function () { return stored; } };
    eval(fnSrc);
    var result = loadDensityMode();
    var expected = (stored === 'comfy' || stored === 'compact' || stored === 'list') ? stored : 'compact';
    console.assert(result === expected, file + ': loadDensityMode(' + JSON.stringify(stored) + ') => ' + result + ', expected ' + expected);
  });
});
console.log('OK: Task 2 verification passed');
```

Run: `cd /home/asama/bookmarks-shelf && node .superpowers/sdd/<ワークスペース>/scratch/verify-task2.js`

Expected: `OK: Task 2 verification passed`、assertion失敗なし。

- [ ] **Step 9: コミット**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "feat: 情報密度モード(ゆったり/コンパクト/リスト)切替トグルを追加"
```

---

### Task 3: コンパクトモードのカード表示（R3）

**Files:**
- Modify: `bookmarks.html:2281-2341`（`renderCard`をリネーム＋新規関数追加）、`bookmarks.html:1789-1908`の`renderSections()`内グリッドクラス付与部分、CSS、SVGシンボル定義（既存の`<template id="icon-file">`付近）
- Modify: `bookmarks-filesync.html:2548-2608`（同上）、`bookmarks-filesync.html:2067-2175`の`renderSections()`内、CSS、SVGシンボル定義

**Interfaces:**
- Consumes: `state.densityMode`（Task 2で追加）。
- Produces: `renderCardComfy(b)`（既存`renderCard()`の中身をリネームしたもの）、`renderCardCompact(b)`（新規）。`renderCard(b)`は以後`state.densityMode`に応じたディスパッチャになる（Task 4でさらに`list`分岐を追加）。

- [ ] **Step 1: `bookmarks.html`の既存`renderCard`を`renderCardComfy`にリネームし、ディスパッチャを追加**

`bookmarks.html:2281`の`function renderCard(b) {`を`function renderCardComfy(b) {`に変更する（関数本体・終了`}`はそのまま）。その直後に以下を追加する。

```js
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

    function renderCard(b) {
      if (state.densityMode === 'compact') return renderCardCompact(b);
      return renderCardComfy(b);
    }
```

- [ ] **Step 2: `bookmarks-filesync.html`に同一の変更を加える**

`bookmarks-filesync.html:2548`の`function renderCard(b) {`を`function renderCardComfy(b) {`に変更し、Step 1と完全に同一の`renderCardCompact(b)`・`renderCard(b)`をその直後に追加する。

- [ ] **Step 3: SVGシンボル定義を追加**

両ファイルで、既存の`<template id="icon-file">`要素を`grep -n 'id="icon-file"'`で検索し、その直後（`</template>`群の終わりの直後）に挿入:

```html
<svg style="display:none" aria-hidden="true">
  <symbol id="ic-memo" viewBox="0 0 24 24"><path d="M6 3h9l5 5v13H6z" fill="none" stroke="currentColor" stroke-width="1.8"></path><path d="M15 3v5h5" fill="none" stroke="currentColor" stroke-width="1.8"></path></symbol>
  <symbol id="ic-key" viewBox="0 0 24 24"><circle cx="8" cy="15" r="4" fill="none" stroke="currentColor" stroke-width="1.8"></circle><path d="M11 12l9-9M17 6l2 2M14 9l2 2" fill="none" stroke="currentColor" stroke-width="1.8"></path></symbol>
  <symbol id="ic-lock" viewBox="0 0 24 24"><rect x="5" y="11" width="14" height="9" rx="1.5" fill="none" stroke="currentColor" stroke-width="1.8"></rect><path d="M8 11V8a4 4 0 018 0v3" fill="none" stroke="currentColor" stroke-width="1.8"></path></symbol>
</svg>
```

- [ ] **Step 4: `renderSections()`のグリッドにコンパクト修飾クラスを付与**

`bookmarks.html:1808-1812`（`var body = items.length ? '<div class="group-grid" data-group-grid="' + ...`の行）を以下に置き換える。

```js
            var gridModifier = state.densityMode === 'compact' ? ' group-grid--compact' : '';
            var body = items.length
              ? '<div class="group-grid' + gridModifier + '" data-group-grid="' + escapeHtml(key || '') + '">' +
                  items.map(function (b) { return b.id === state.editingId ? renderEditCard(b) : renderCard(b); }).join('') +
                '</div>'
              : '<div class="group-section-empty" data-group-grid="' + escapeHtml(key || '') + '">該当するブックマークがありません。ここにドラッグして移動できます。</div>';
```

`bookmarks-filesync.html:2091-2095`の同じ箇所にも同一の変更を加える。

- [ ] **Step 5: CSSを追加**

両ファイルの`.card-footer`ルール群の直後に挿入:

```css
    .card--compact .card-stub { width: 14px; }
    .card--compact .card-body { padding: 8px 9px; }
    .card--compact .seal { width: 18px; height: 18px; }
    .card--compact .seal img, .card--compact .seal svg { width: 11px; height: 11px; }
    .card--compact .card-title { font-size: 0.78rem; }
    .card--compact .card-footer { margin-top: 6px; gap: 2px; }
    .card--compact .card-actions { gap: 2px; }

    .group-grid--compact { grid-template-columns: repeat(auto-fill, minmax(170px, 1fr)); }

    .icon-btn {
      all: unset;
      box-sizing: border-box;
      cursor: pointer;
      width: 20px;
      height: 20px;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 4px;
      font-size: 0.85rem;
      color: var(--ink-soft);
    }
    .icon-btn.amber { color: var(--amber-ink); }
    .icon-btn:hover { background: rgba(27, 53, 56, 0.08); }

    .ind-btn {
      box-sizing: border-box;
      width: 20px;
      height: 20px;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 4px;
      position: relative;
      border: none;
      background: transparent;
      padding: 0;
    }
    .ind-btn svg { width: 12px; height: 12px; stroke: currentColor; fill: none; stroke-width: 1.6; }
    .ind-btn.filled { color: var(--amber-ink); cursor: pointer; }
    .ind-btn.empty { color: var(--sage); opacity: 0.55; cursor: default; pointer-events: none; }
    .ind-btn.empty::after {
      content: '';
      position: absolute;
      width: 14px;
      height: 1.3px;
      background: var(--sage);
      transform: rotate(-45deg);
    }
```

- [ ] **Step 6: 検証スクリプトを作成し実行**

`.superpowers/sdd/<ワークスペース>/scratch/verify-task3.js`:

```js
var fs = require('fs');

function extractFn(src, name) {
  var startIdx = src.indexOf('function ' + name + '(');
  if (startIdx === -1) throw new Error(name + ' not found');
  var i = src.indexOf('{', startIdx), depth = 0;
  for (; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) { i++; break; } }
  }
  return src.slice(startIdx, i);
}

['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');

  // 密度モードはドラッグ可否に影響しないことを静的に確認(Review Focus項目)
  if (file === 'bookmarks.html') {
    console.assert(/if \(!state\.selectMode && state\.sortMode !== 'frequency'\) bindDragEvents\(card, b\);/.test(src),
      file + ': bindDragEventsの呼び出し条件にdensityModeへの参照が混入していないか確認できない(文言が変わっている可能性)');
  }

  function escapeHtml(s) { return String(s).replace(/[&<>"']/g, function (c) { return { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]; }); }
  function resolveIconHtml() { return '<svg></svg>'; }
  function resolveOpenHref(b) { return b.url; }
  global.escapeHtml = escapeHtml;
  global.resolveIconHtml = resolveIconHtml;
  global.resolveOpenHref = resolveOpenHref;
  global.state = { selectMode: false, selectedIds: new Set(), tag: null, densityMode: 'compact' };

  eval(extractFn(src, 'renderCardCompact').replace('function renderCardCompact', 'var renderCardCompact = function'));

  var withCreds = renderCardCompact({ id: '1', title: 'Test', url: 'https://example.com', scheme: 'https', memo: 'メモ本文', loginId: 'user', loginPassword: 'pass', pinned: true });
  console.assert(withCreds.indexOf('data-card-id="1"') !== -1, file + ': data-card-id属性が維持されていない');
  console.assert(withCreds.indexOf('tabindex="-1"') !== -1, file + ': tabindex属性が維持されていない');
  console.assert(withCreds.indexOf('class="card card--compact"') !== -1, file + ': card--compact修飾クラスが付与されていない');
  console.assert(withCreds.indexOf('data-action="copy-login-id"') !== -1, file + ': IDありなのにcopy-login-idボタンが無い');
  console.assert(withCreds.indexOf('data-action="copy-login-password"') !== -1, file + ': パスワードありなのにcopy-login-passwordボタンが無い');
  console.assert(withCreds.indexOf('ind-btn filled') !== -1, file + ': メモありのfilled表示が無い');
  console.assert(withCreds.indexOf('<div class="card-tags">') === -1, file + ': タグ要素が出力されてしまっている(非表示のはず)');
  console.assert(/<p class="card-memo">/.test(withCreds) === false, file + ': メモ本文がそのまま出力されてしまっている(アイコンのみのはず)');

  var withoutCreds = renderCardCompact({ id: '2', title: 'NoCreds', url: 'https://example.com', scheme: 'https', pinned: false });
  console.assert(withoutCreds.indexOf('data-action="copy-login-id"') === -1, file + ': IDが無いのにcopy-login-idボタンが出力されている(クリック不可であるべき)');
  console.assert(withoutCreds.indexOf('data-action="copy-login-password"') === -1, file + ': パスワードが無いのにcopy-login-passwordボタンが出力されている(クリック不可であるべき)');
  console.assert(withoutCreds.indexOf('ind-btn empty') !== -1, file + ': 値が無い項目のempty表示が無い');
});
console.log('OK: Task 3 verification passed');
```

Run: `cd /home/asama/bookmarks-shelf && node .superpowers/sdd/<ワークスペース>/scratch/verify-task3.js`

Expected: `OK: Task 3 verification passed`、assertion失敗なし。

- [ ] **Step 7: コミット**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "feat: コンパクトモードのカード表示を追加(タグ/メモ/認証情報の非表示化とアイコン化)"
```

---

### Task 4: リストモードの行表示（R4）

**Files:**
- Modify: `bookmarks.html`（Task 3で追加した`renderCard`ディスパッチャ、`renderSections()`の`gridModifier`）、CSS
- Modify: `bookmarks-filesync.html`（同上）

**Interfaces:**
- Consumes: `state.densityMode`、Task 3の`renderCardCompact`/`renderCardComfy`。
- Produces: `renderCardList(b)`（新規）。`renderCard(b)`ディスパッチャに`list`分岐を追加。

- [ ] **Step 1: `bookmarks.html`に`renderCardList`を追加し、ディスパッチャを拡張**

Task 3で追加した`renderCardCompact(b)`関数の直後に挿入:

```js
    function renderCardList(b) {
      var iconHtml = resolveIconHtml(b);
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

直後の`renderCard(b)`ディスパッチャを以下に置き換える。

```js
    function renderCard(b) {
      if (state.densityMode === 'compact') return renderCardCompact(b);
      if (state.densityMode === 'list') return renderCardList(b);
      return renderCardComfy(b);
    }
```

- [ ] **Step 2: `bookmarks-filesync.html`に同一の変更を加える**

Step 1と完全に同一の`renderCardList(b)`・`renderCard(b)`を、`bookmarks-filesync.html`のTask 3で追加した`renderCardCompact(b)`の直後に反映する。

- [ ] **Step 3: `renderSections()`の`gridModifier`にlist分岐を追加**

`bookmarks.html`のTask 3 Step 4で追加した行を以下に置き換える。

```js
            var gridModifier = state.densityMode === 'compact' ? ' group-grid--compact' : state.densityMode === 'list' ? ' group-grid--list' : '';
```

`bookmarks-filesync.html`の同じ行にも同一の変更を加える。

- [ ] **Step 4: CSSを追加**

両ファイルの、Task 3で追加した`.ind-btn.empty::after`ルールの直後に挿入:

```css
    .card--list {
      align-items: center;
      gap: 8px;
      padding: 6px 8px;
      border-radius: 0;
      border-bottom: 1px solid var(--sage);
      box-shadow: none;
      background: transparent;
    }
    .card--list .seal { width: 16px; height: 16px; }
    .card--list .seal img, .card--list .seal svg { width: 10px; height: 10px; }
    .card--list .card-title {
      font-size: 0.82rem;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
      flex: 1 1 auto;
      min-width: 0;
    }
    .card--list .card-footer { margin-top: 0; flex: 0 0 auto; }

    .row-memo-preview {
      font-size: 0.72rem;
      font-style: italic;
      color: var(--ink-soft);
      background: rgba(227, 167, 63, 0.14);
      border-radius: 10px;
      padding: 2px 9px;
      margin-left: 14px;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
      max-width: 220px;
      flex: 0 1 auto;
    }
    .row-memo-preview svg { width: 10px; height: 10px; stroke: currentColor; fill: none; stroke-width: 1.8; vertical-align: -1px; margin-right: 3px; }

    .group-grid--list { display: flex; flex-direction: column; gap: 0; }
```

- [ ] **Step 5: 検証スクリプトを作成し実行**

`.superpowers/sdd/<ワークスペース>/scratch/verify-task4.js`:

```js
var fs = require('fs');

function extractFn(src, name) {
  var startIdx = src.indexOf('function ' + name + '(');
  if (startIdx === -1) throw new Error(name + ' not found');
  var i = src.indexOf('{', startIdx), depth = 0;
  for (; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) { i++; break; } }
  }
  return src.slice(startIdx, i);
}

['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');

  function escapeHtml(s) { return String(s).replace(/[&<>"']/g, function (c) { return { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]; }); }
  global.escapeHtml = escapeHtml;
  global.resolveIconHtml = function () { return '<svg></svg>'; };
  global.resolveOpenHref = function (b) { return b.url; };
  global.state = { densityMode: 'list' };

  eval(extractFn(src, 'renderCardList').replace('function renderCardList', 'var renderCardList = function'));

  var withMemo = renderCardList({ id: '1', title: 'Test', url: 'https://example.com', scheme: 'https', memo: 'これはメモです', pinned: false });
  console.assert(withMemo.indexOf('data-card-id="1"') !== -1, file + ': data-card-id属性が維持されていない');
  console.assert(withMemo.indexOf('class="card card--list"') !== -1, file + ': card--list修飾クラスが付与されていない');
  console.assert(/<h3 class="card-title"><a [^>]*>/.test(withMemo), file + ': .card-title > a のセレクタ互換が崩れている(既存のタイトルクリックイベント割り当てが機能しなくなる)');
  console.assert(withMemo.indexOf('これはメモです') !== -1, file + ': メモ本文がリストモードの行に表示されていない');
  console.assert(withMemo.indexOf('row-memo-preview') !== -1, file + ': メモプレビューのチップ要素が無い');

  var noMemo = renderCardList({ id: '2', title: 'NoMemo', url: 'https://example.com', scheme: 'https', pinned: false });
  console.assert(noMemo.indexOf('row-memo-preview') === -1, file + ': メモが無いのにプレビューチップが出力されている');

  var withGroupGridList = src.indexOf("state.densityMode === 'list' ? ' group-grid--list'") !== -1;
  console.assert(withGroupGridList, file + ': renderSections()のgridModifierにlist分岐が追加されていない');
});
console.log('OK: Task 4 verification passed');
```

Run: `cd /home/asama/bookmarks-shelf && node .superpowers/sdd/<ワークスペース>/scratch/verify-task4.js`

Expected: `OK: Task 4 verification passed`、assertion失敗なし。

- [ ] **Step 6: コミット**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "feat: リストモードの行表示を追加(グループセクション単位でグルーピング、メモのインラインプレビュー)"
```

---

### Task 5: お気に入りセクションの並び替え（R5）

**Files:**
- Modify: `bookmarks.html:1724-1743`（`renderFavoriteCard`をリネーム＋新規関数追加）、`bookmarks.html:1745-1787`（`renderFavoritesSection()`にドラッグバインド呼び出し追加）、`bookmarks.html`のドラッグ関連コード付近（`draggedCardId`宣言の近く、新規`draggedFavId`・`moveFavorite`・`bindFavoriteDragEvents`）、CSS
- Modify: `bookmarks-filesync.html`（同上、対応する行）

**Interfaces:**
- Consumes: `state.densityMode`、`bindDragReorder(el, key, opts)`（既存の汎用ユーティリティ）、Task 1の`favoriteOrder`フィールド。
- Produces: `renderFavoriteCardComfy(b)`（既存`renderFavoriteCard()`の中身をリネーム。ゆったり/コンパクト共通で使う）、`renderFavoriteCardList(b)`（新規）、`renderFavoriteCard(b)`ディスパッチャ、`compareFavoriteOrder(a, b)`（favoriteOrder比較の単一ソース、null末尾送り）、`moveFavorite(draggedId, targetId, after)`、`bindFavoriteDragEvents(container)`。

- [ ] **Step 1: `bookmarks.html`の既存`renderFavoriteCard`を`renderFavoriteCardComfy`にリネームし、ドラッグハンドルを追加**

`bookmarks.html:1724`の`function renderFavoriteCard(b) {`を`function renderFavoriteCardComfy(b) {`に変更したうえで、関数本体（1725-1742行目）を以下に置き換える（`.card-stub`を追加）。

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
```

その直後に、リストモード用の行と、ディスパッチャを追加する。

```js
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

- [ ] **Step 2: `bookmarks-filesync.html`に同一の変更を加える**

`bookmarks-filesync.html:2013`の`function renderFavoriteCard(b) {`を`function renderFavoriteCardComfy(b) {`に変更し、Step 1と完全に同一の内容（`renderFavoriteCardComfy`本体・`renderFavoriteCardList`・`renderFavoriteCard`ディスパッチャ）に置き換える。

- [ ] **Step 3: `renderFavoritesSection()`の並び順を`sortItems()`から`favoriteOrder`基準に変更**

**重要**: `renderFavoritesSection()`は現状`sortItems(pinned)`を呼んでおり、これは`state.sortMode`（手動順/よく使う順）に連動したソートである。spec R5の「お気に入りは常に`favoriteOrder`で固定表示し、`state.sortMode`の影響を受けない」を満たすには、この呼び出し自体を`favoriteOrder`基準のソートに置き換える必要がある（Task 1〜2段階ではまだ手を付けていない箇所）。

まず`bookmarks.html`で`getGroupOrderForName()`または`getNextFavoriteOrder()`の直後に、favoriteOrder比較の単一ソースとなる関数を追加する（Step4の`moveFavorite()`もこれを使うため、表示ソートとドラッグ並び替えで「nullは末尾に送る」ルールが2箇所に別々実装されて食い違うことを防ぐ）。

```js
    // favoriteOrder比較の単一ソース。null(マイグレーション未実施等の異常系)は末尾に送る。
    function compareFavoriteOrder(a, b) {
      var ao = a.favoriteOrder != null ? a.favoriteOrder : Infinity;
      var bo = b.favoriteOrder != null ? b.favoriteOrder : Infinity;
      return ao - bo;
    }
```

次に`bookmarks.html:1753`の`var items = sortItems(pinned);`を以下に置き換える。

```js
      // お気に入りは state.sortMode(手動順/よく使う順)の影響を受けず、常に favoriteOrder で固定表示する。
      var items = pinned.slice().sort(compareFavoriteOrder);
```

`bookmarks-filesync.html`にも同一の`compareFavoriteOrder()`関数を追加し、`renderFavoritesSection()`内の同じ行（`var items = sortItems(pinned);`）にも同一の置き換えを加える。

- [ ] **Step 4: `draggedFavId`変数・`moveFavorite()`・`bindFavoriteDragEvents()`を追加**

`bookmarks.html`で`draggedCardId`が宣言されている行を`grep -n 'var draggedCardId'`で特定し、その直後に挿入:

```js
    var draggedFavId = null; // お気に入りセクション専用のドラッグ追跡変数(通常カードのdraggedCardIdとは独立)

    // お気に入りセクション内の並び替え専用。favoriteOrderのみ更新し、元のグループ内のorder・groupには一切影響しない。
    // 並び順の基準はStep3で追加したcompareFavoriteOrder()を再利用する(表示ソートと並び替え計算を食い違わせないため)。
    function moveFavorite(draggedId, targetId, after) {
      if (!draggedId || draggedId === targetId) return;
      var dragged = bookmarks.find(function (x) { return x.id === draggedId; });
      var target = bookmarks.find(function (x) { return x.id === targetId; });
      if (!dragged || !target || !dragged.pinned || !target.pinned) return;

      var pinned = bookmarks.filter(function (x) { return x.pinned; }).sort(compareFavoriteOrder);
      pinned.splice(pinned.indexOf(dragged), 1);
      var toIdx = pinned.indexOf(target);
      if (after) toIdx += 1;
      pinned.splice(toIdx, 0, dragged);
      pinned.forEach(function (x, i) { x.favoriteOrder = i; });
      saveData(bookmarks);
      render();
    }

    // お気に入り専用のドラッグバインド。bindDragEvents/bindGroupSectionDragEventsとは独立した実装とし、
    // data-card-idベースの既存イベント割り当てループ・getNavigableCards()からは引き続き除外されたままにする。
    function bindFavoriteDragEvents(container) {
      container.querySelectorAll('[data-fav-id]').forEach(function (el) {
        var id = el.getAttribute('data-fav-id');
        el.setAttribute('draggable', 'true');
        bindDragReorder(el, id, {
          getDragged: function () { return draggedFavId; },
          setDragged: function (v) { draggedFavId = v; },
          stopPropagation: true,
          dropEffect: 'move',
          onDrop: function (draggedId, targetId, after) { moveFavorite(draggedId, targetId, after); }
        });
      });
    }
```

- [ ] **Step 5: `bookmarks-filesync.html`に同一の変更を加える**

`bookmarks-filesync.html`で`draggedCardId`が宣言されている行の直後に、Step 4と完全に同一のコードを追加する。

- [ ] **Step 6: `renderFavoritesSection()`からドラッグバインドを呼び出す**

`bookmarks.html:1787`（`renderFavoritesSection()`末尾、既存の`container.querySelectorAll('[data-fav-id]').forEach(...)`ブロックの閉じ`});`の直後）に追加:

```js
      bindFavoriteDragEvents(container);
```

`bookmarks-filesync.html`の`renderFavoritesSection()`の対応箇所にも同一の1行を追加する。

- [ ] **Step 7: CSSを追加**

両ファイルの、Task 4で追加した`.group-grid--list`ルールの直後に挿入:

```css
    .card-favorite .card-stub { width: 14px; }
    .card-favorite--list {
      align-items: center;
      gap: 8px;
      padding: 6px 8px;
      border-radius: 0;
      border-bottom: 1px solid var(--sage);
      box-shadow: none;
      background: transparent;
    }
    .card-favorite--list .seal { width: 16px; height: 16px; }
    .card-favorite--list .seal img, .card-favorite--list .seal svg { width: 10px; height: 10px; }
    .card-favorite--list .card-title {
      font-size: 0.82rem;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
      flex: 1 1 auto;
      min-width: 0;
    }
    .card-favorite--list .card-footer { margin-top: 0; flex: 0 0 auto; }
    .fav-grip { color: var(--amber); font-size: 0.8rem; cursor: grab; flex: 0 0 auto; }
    .row-group-chip {
      font-family: var(--font-mono);
      font-size: 0.68rem;
      color: var(--ink-soft);
      background: var(--ivory);
      border-radius: 3px;
      padding: 1px 6px;
      flex: 0 0 auto;
    }
```

- [ ] **Step 8: 検証スクリプトを作成し実行**

`.superpowers/sdd/<ワークスペース>/scratch/verify-task5.js`:

```js
var fs = require('fs');

function extractFn(src, name) {
  var startIdx = src.indexOf('function ' + name + '(');
  if (startIdx === -1) throw new Error(name + ' not found in source');
  var i = src.indexOf('{', startIdx), depth = 0;
  for (; i < src.length; i++) {
    if (src[i] === '{') depth++;
    else if (src[i] === '}') { depth--; if (depth === 0) { i++; break; } }
  }
  return src.slice(startIdx, i);
}

['bookmarks.html', 'bookmarks-filesync.html'].forEach(function (file) {
  var src = fs.readFileSync(file, 'utf8');

  global.bookmarks = [
    { id: 'a', pinned: true, favoriteOrder: 0, order: 5, group: 'work' },
    { id: 'b', pinned: true, favoriteOrder: 1, order: 2, group: 'home' },
    { id: 'c', pinned: true, favoriteOrder: 2, order: 9, group: null },
    { id: 'd', pinned: false, favoriteOrder: null, order: 0, group: 'work' }
  ];
  global.saveData = function () {};
  global.render = function () {};

  // moveFavorite()はcompareFavoriteOrder()に依存するため、先にスコープへ読み込む
  eval(extractFn(src, 'compareFavoriteOrder').replace('function compareFavoriteOrder', 'var compareFavoriteOrder = function'));
  eval(extractFn(src, 'moveFavorite').replace('function moveFavorite', 'var moveFavorite = function'));

  // a(favoriteOrder=0)をc(favoriteOrder=2)の後ろへ移動 → 並びは b, c, a になるはず
  moveFavorite('a', 'c', true);
  var byId = {}; global.bookmarks.forEach(function (x) { byId[x.id] = x; });
  console.assert(byId.b.favoriteOrder === 0, file + ': bが先頭(0)になっていない: got ' + byId.b.favoriteOrder);
  console.assert(byId.c.favoriteOrder === 1, file + ': cが2番目(1)になっていない: got ' + byId.c.favoriteOrder);
  console.assert(byId.a.favoriteOrder === 2, file + ': aが末尾(2)になっていない: got ' + byId.a.favoriteOrder);

  // 元のorder・groupは一切変更されていないはず(Review Focus項目)
  console.assert(byId.a.order === 5 && byId.a.group === 'work', file + ': aのorder/groupが変更されてしまっている');
  console.assert(byId.b.order === 2 && byId.b.group === 'home', file + ': bのorder/groupが変更されてしまっている');
  console.assert(byId.c.order === 9 && byId.c.group === null, file + ': cのorder/groupが変更されてしまっている');

  // 非ピン留めのdは一切触れられていないはず
  console.assert(byId.d.favoriteOrder === null && byId.d.order === 0, file + ': 非ピン留めdのfavoriteOrder/orderが変更されてしまっている');

  // 自分自身へのドロップは何もしない
  var beforeSelf = JSON.stringify(global.bookmarks);
  moveFavorite('a', 'a', true);
  console.assert(JSON.stringify(global.bookmarks) === beforeSelf, file + ': 自分自身へのドロップで状態が変化してしまっている');

  console.assert(src.indexOf('function bindFavoriteDragEvents(container)') !== -1, file + ': bindFavoriteDragEventsが定義されていない');
  console.assert(src.indexOf('var draggedFavId') !== -1, file + ': draggedFavId変数が定義されていない');
  console.assert(/bindFavoriteDragEvents\(container\);/.test(src), file + ': renderFavoritesSection()からbindFavoriteDragEventsが呼ばれていない');

  // renderFavoritesSection()がもはやsortItems()でなくcompareFavoriteOrder基準でソートしていることを確認
  var favSectionSrc = extractFn(src, 'renderFavoritesSection');
  console.assert(favSectionSrc.indexOf('sortItems(pinned)') === -1, file + ': renderFavoritesSection()が依然としてsortItems(pinned)を呼んでいる(state.sortModeの影響を受けてしまう)');
  console.assert(favSectionSrc.indexOf('compareFavoriteOrder') !== -1, file + ': renderFavoritesSection()の並び替えにcompareFavoriteOrderが使われていない');

  // compareFavoriteOrder()自体を実ソースから抽出して直接テストする(手書きの再実装ではなく、
  // moveFavorite()・renderFavoritesSection()の両方が実際に呼んでいる関数そのものを検証する)
  eval(extractFn(src, 'compareFavoriteOrder').replace('function compareFavoriteOrder', 'var compareFavoriteOrder = function'));
  var sortedPinned = [
    { id: 'x', favoriteOrder: 2 },
    { id: 'y', favoriteOrder: null },
    { id: 'z', favoriteOrder: 0 }
  ].slice().sort(compareFavoriteOrder);
  console.assert(sortedPinned.map(function (x) { return x.id; }).join(',') === 'z,x,y', file + ': compareFavoriteOrder()の順序またはnull末尾送りが期待通りでない: got ' + sortedPinned.map(function (x) { return x.id; }).join(','));

  // moveFavorite()が独自のソート式を持たず、同じcompareFavoriteOrderを再利用していることを確認
  // (nullの扱いが表示側とドラッグ側で食い違わないようにするため)
  var moveFavoriteSrc = extractFn(src, 'moveFavorite');
  console.assert(moveFavoriteSrc.indexOf('.sort(compareFavoriteOrder)') !== -1, file + ': moveFavorite()がcompareFavoriteOrderを再利用していない(独自のソート式を持っていると表示側とnullの扱いが食い違う可能性)');
});
console.log('OK: Task 5 verification passed');
```

Run: `cd /home/asama/bookmarks-shelf && node .superpowers/sdd/<ワークスペース>/scratch/verify-task5.js`

Expected: `OK: Task 5 verification passed`、assertion失敗なし。

- [ ] **Step 9: コミット**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "feat: お気に入りセクション内のドラッグ並び替えを追加(favoriteOrder、全密度モード対応)"
```
