# タグ/グループナビの矢印キー移動と検索欄Escクリア

- 日付: 2026-08-08
- 対象ファイル: `bookmarks.html`, `bookmarks-filesync.html`
- 前提: `docs/superpowers/specs/2026-08-08-keyboard-shortcuts-design.md`（ショートカットキー見直し）の続き。同じブランチ内で、その実装完了・最終レビュー通過後に追加要望として着手する。

## 背景

前回のショートカットキー見直しで、矢印キーを「カードにフォーカスしていない間は一覧に入る」よう常時有効化した。その最終レビューで、`t`/`g` でタグ/グループナビにフォーカスした直後に矢印キーを押すと、ナビ内を移動するのではなくカード一覧にフォーカスが飛んでしまう（ナビ自体は矢印キーを処理しないため）という軽微な指摘があった。この指摘を受け、タグ/グループナビ内でも矢印キーでボタン間を移動できるようにする。

また、検索欄にフォーカスしている間にEscキーで検索をクリアできるようにしたいという要望があった。現状、検索欄にフォーカスしていてもEscを押すと（テキスト入力中でもEscはグローバルに先に処理されるため）`closeTopmostLayer()` が呼ばれ、ヘルプ→グループ管理モーダル→タグ管理モーダル→フォーム→選択モードの順に閉じる処理が走る。検索欄の内容そのものをクリアする手段は既存の×ボタンのクリックのみだった。

## 要件

- フォーカスが `#tag-nav` または `#group-nav` 内のボタンにある間、矢印キー（↓→で次、↑←で前）でそのリスト内のボタン間を移動できる。リストの端では何もしない（カードの移動と同じ挙動）。
- 検索欄（`#search-input`）にフォーカスがあり、かつ検索文字列が入力されている状態でEscキーを押すと、既存の×ボタン（`#search-clear-btn`）と同じ処理（検索をクリアし、検索欄にフォーカスを残す）を行う。
- 検索欄にフォーカスがあっても検索文字列が空の場合は、これまで通り `closeTopmostLayer()` の通常のチェーンにフォールスルーする。
- ヘルプパネルのEscの説明文を、検索クリアが最優先で行われることが分かるように更新する。

## 設計

### タグ/グループナビの矢印キー移動

新規ヘルパー関数 `moveNavFocus(btn, direction)` を追加する。`btn.closest('ul')` で親の `<ul>`（`#tag-nav` または `#group-nav`）を取得し、その中のボタン列で次/前を計算する（`moveCardFocus` と同じ「無ければ何もしない」規約）。

```js
  // タグ/グループナビ内でのボタン間移動(次/前が無ければ何もしない。moveCardFocusと同じ規約)
  function moveNavFocus(btn, direction) {
    var list = btn.closest('ul');
    var buttons = Array.prototype.slice.call(list.querySelectorAll('button'));
    var next = buttons[buttons.indexOf(btn) + direction];
    if (next) next.focus();
  }
```

`handleGlobalShortcut()` の矢印キー分岐の先頭に、タグ/グループナビ内のボタンかどうかのチェックを追加する（既存の「カードにフォーカス中ならreturn false」のチェックより先に判定する）。

変更前（`bookmarks.html:2578-2587`）:
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

変更後:
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

モーダル表示中は既存のガード（`if (!groupManageOverlay.hidden) return;` 等、`handleGlobalShortcut` 呼び出しより前にある）でそもそも `handleGlobalShortcut` 自体が呼ばれないため、追加のガードは不要。

### 検索欄でのEscクリア

既存の×ボタンのクリック処理を独立した関数 `clearSearch()` に切り出し、×ボタンのクリックハンドラとEscハンドラの両方から呼ぶ。

変更前（`bookmarks.html:2398-2417`）:
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

変更後:
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

`document.addEventListener('keydown', ...)` 内のEscape分岐を変更する。

変更前（`bookmarks.html:2650-2653`）:
```js
    if (e.key === 'Escape') {
      closeTopmostLayer();
      return;
    }
```

変更後:
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

`searchInput` はこの時点（ファイル内でこの `keydown` リスナーより手前）で既に宣言・代入済みの変数なので、そのまま参照できる。

### ヘルプパネルの更新

`Esc` の説明行を、検索クリアが最優先で行われることが分かるように更新する。

変更前:
```html
    <dt><kbd>Esc</kbd></dt><dd>ヘルプ &rarr; グループ管理 &rarr; タグ管理 &rarr; フォーム &rarr; 選択モードの順に閉じる</dd>
```

変更後:
```html
    <dt><kbd>Esc</kbd></dt><dd>検索欄にフォーカス中かつ入力中なら検索をクリア。それ以外はヘルプ &rarr; グループ管理 &rarr; タグ管理 &rarr; フォーム &rarr; 選択モードの順に閉じる</dd>
```

## 変更しないもの

- `closeTopmostLayer()` 自体のロジック・優先順位
- `moveCardFocus()`・`getNavigableCards()` などカード側の既存ロジック
- `handleCardShortcut()` 関数
- タグ/グループナビの `click` ハンドラ・フィルタ切り替えロジック
- `/`・`n`・`t`・`g`・`?`・`Home`・`End`・`Enter`・`e`・`c`・`p`・`Delete`/`Backspace` の割り当て

## 検証方法

自動テストは存在しないプロジェクトのため、ブラウザでの手動確認のみ（`bookmarks.html` と `bookmarks-filesync.html` の両方で実施）:

- `t` でタグタブに切り替え、`↓` でタグナビ内を次のタグへ移動できることを確認する。`↑` で前へ戻れることを確認する。リストの端で矢印キーを押しても何も起きない（カード一覧に飛ばない）ことを確認する。
- `g` でグループタブに切り替え、同様にグループナビ内を矢印キーで移動できることを確認する。
- 検索欄に何か入力した状態でフォーカスしたまま Esc を押すと、検索がクリアされ、検索欄にフォーカスが残ることを確認する（一覧の絞り込みも解除されることを確認する）。
- 検索欄が空の状態でフォーカスしたまま Esc を押すと、これまで通り `closeTopmostLayer()` の挙動（ヘルプが開いていれば閉じる、等）になることを確認する。
- `?` キーでヘルプパネルを開き、Escの説明文が更新されていることを確認する。
- カード一覧での矢印キー移動・`Home`/`End`・その他既存のショートカットが、これまで通り動作することを確認する（回帰確認）。
