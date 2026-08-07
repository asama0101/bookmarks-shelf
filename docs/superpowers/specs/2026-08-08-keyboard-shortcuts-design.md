# ショートカットキーの見直し

- 日付: 2026-08-08
- 対象ファイル: `bookmarks.html`, `bookmarks-filesync.html`

## 背景

現状のショートカットキーには2つの不満がある:

1. 割り当てが覚えづらい・直感的でない（例: `s` が「カード一覧へフォーカス」を意味するのは連想しにくい）。
2. `e`（編集）・`c`（ログインIDコピー）・`p`（パスワードコピー）・`Delete`/`Backspace`（削除）・矢印キー（カード間移動）・`Enter`（開く）は、いずれもカードに既にフォーカスしている状態でないと使えない。フォーカスを合わせるには `s` を押すか、マウスでカードをクリックするか、Tabで辿り着く必要があり、毎回の手間になっている。

## 要件

- `s`（カード一覧へフォーカス）を廃止する。代わりに矢印キー（↑↓←→）を、カードにフォーカスしていない状態でも常に使えるようにする。フォーカス済みの場合は従来通りカード間を移動し、未フォーカスの場合は一覧の起点カードへフォーカスを移す。
- `Home`/`End` キーを新設し、フォーカス状態に関わらず常に一覧の先頭/末尾カードへフォーカスできるようにする。
- `c`（ログインIDコピー）・`p`（パスワードコピー）・`Delete`/`Backspace`（削除）のショートカットを廃止する。これらの操作はカード内に既存の「コピー」「削除」ボタンがあるため、機能自体は失われない。
- `/`（検索）・`n`（追加）・`t`（タグタブ）・`g`（グループタブ）・`?`（ヘルプ）・`Esc`（階層的に閉じる）・`Enter`（開く）・`e`（編集）の割り当ては変更しない。
- ヘルプパネル（`?` キーで開く一覧）を上記の変更に合わせて更新する。

## 設計

### `handleGlobalShortcut()` の変更

`bookmarks.html:2551-2584` の `switch` 文を次のように変更する。

- `case 's':` の分岐を削除する。
- 代わりに、矢印キー4種と `Home`/`End` の分岐を追加する。

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

`getNavigableCards()` は既存関数（`bookmarks.html:2658` 付近）で、そのまま利用できる。`handleGlobalShortcut()` は既存の `keydown` リスナー内で `handleCardShortcut` より先に呼ばれるため、矢印キーの分岐が `false` を返した場合（＝既にカードにフォーカスしている場合）は、そのまま既存の `handleCardShortcut` 側の矢印キー処理（`moveCardFocus`）に自然にフォールスルーする。

### `handleCardShortcut()` の変更

`bookmarks.html:2587-2629` の `switch` 文から、`case 'c':`・`case 'p':`・`case 'Delete': case 'Backspace':` の3つの分岐を削除する。矢印キー・`Enter`・`e` の分岐は変更しない。

```js
  function handleCardShortcut(e, card, cardId) {
    switch (e.key) {
      case 'ArrowDown':
      case 'ArrowRight':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocus(card, 1);
        return;
      case 'ArrowUp':
      case 'ArrowLeft':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocus(card, -1);
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
    }
  }
```

`copyToClipboard` の呼び出し元がカード内ボタンのクリックハンドラのみになるが、そのクリックハンドラ自体（`bookmarks.html` 内、カード描画時にコピー/削除ボタンへ `addEventListener('click', ...)` する既存コード）は変更しない。

### ヘルプパネルの変更

`bookmarks.html:1070-1084` の `<dl class="help-grid">` を次のように変更する。

変更前:
```html
  <dl class="help-grid">
    <dt><kbd>/</kbd></dt><dd>検索ボックスへフォーカス</dd>
    <dt><kbd>n</kbd></dt><dd>追加パネルの開閉</dd>
    <dt><kbd>?</kbd></dt><dd>このヘルプの開閉</dd>
    <dt><kbd>Esc</kbd></dt><dd>ヘルプ &rarr; グループ管理 &rarr; タグ管理 &rarr; フォーム &rarr; 選択モードの順に閉じる</dd>
    <dt><kbd>&#8593;</kbd> <kbd>&#8595;</kbd> <kbd>&#8592;</kbd> <kbd>&#8594;</kbd></dt><dd>カード間のフォーカス移動</dd>
    <dt><kbd>Enter</kbd></dt><dd>フォーカス中のカードのURLを開く</dd>
    <dt><kbd>e</kbd></dt><dd>フォーカス中のカードを編集</dd>
    <dt><kbd>Delete</kbd></dt><dd>フォーカス中のカードを削除</dd>
    <dt><kbd>s</kbd></dt><dd>カード一覧へフォーカス</dd>
    <dt><kbd>t</kbd></dt><dd>タグタブに切り替えてタグ絞り込みナビへフォーカス</dd>
    <dt><kbd>g</kbd></dt><dd>グループタブに切り替えてグループ絞り込みナビへフォーカス</dd>
    <dt><kbd>c</kbd></dt><dd>フォーカス中のカードのログインIDをコピー</dd>
    <dt><kbd>p</kbd></dt><dd>フォーカス中のカードのパスワードをコピー</dd>
  </dl>
```

変更後:
```html
  <dl class="help-grid">
    <dt><kbd>/</kbd></dt><dd>検索ボックスへフォーカス</dd>
    <dt><kbd>n</kbd></dt><dd>追加パネルの開閉</dd>
    <dt><kbd>?</kbd></dt><dd>このヘルプの開閉</dd>
    <dt><kbd>Esc</kbd></dt><dd>ヘルプ &rarr; グループ管理 &rarr; タグ管理 &rarr; フォーム &rarr; 選択モードの順に閉じる</dd>
    <dt><kbd>&#8593;</kbd> <kbd>&#8595;</kbd> <kbd>&#8592;</kbd> <kbd>&#8594;</kbd></dt><dd>カード間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
    <dt><kbd>Home</kbd></dt><dd>一覧の先頭カードへフォーカス</dd>
    <dt><kbd>End</kbd></dt><dd>一覧の末尾カードへフォーカス</dd>
    <dt><kbd>Enter</kbd></dt><dd>フォーカス中のカードのURLを開く</dd>
    <dt><kbd>e</kbd></dt><dd>フォーカス中のカードを編集</dd>
    <dt><kbd>t</kbd></dt><dd>タグタブに切り替えてタグ絞り込みナビへフォーカス</dd>
    <dt><kbd>g</kbd></dt><dd>グループタブに切り替えてグループ絞り込みナビへフォーカス</dd>
  </dl>
```

## 変更しないもの

- `/`・`n`・`t`・`g`・`?`・`Esc`・`Enter`・`e` の割り当て・挙動
- カード内の「コピー」「削除」ボタン自体のクリックハンドラ・挙動
- `Escape` キーの階層的クローズ処理（`closeTopmostLayer()`）
- モーダル表示中にグローバルショートカットを無効化するガード

## 検証方法

自動テストは存在しないプロジェクトのため、ブラウザでの手動確認のみ（`bookmarks.html` と `bookmarks-filesync.html` の両方で実施）:

- ページ読み込み直後（どのカードにもフォーカスしていない状態）で `↓` または `→` を押し、一覧の起点カードにフォーカスが移ることを確認する。続けて矢印キーを押すと、カード間をこれまで通り移動できることを確認する。
- 同様に未フォーカス状態で `Home` を押すと一覧の先頭カードへ、`End` を押すと末尾カードへ、それぞれフォーカスが移ることを確認する。カードにフォーカスしている状態からでも `Home`/`End` が機能することを確認する。
- `s` キーを押しても何も起きないことを確認する。
- カードにフォーカスした状態で `c`・`p`・`Delete`・`Backspace` を押しても何も起きないことを確認する。
- カード内の「コピー」（ログインID・パスワード）・「削除」ボタンをクリックすると、これまで通り動作することを確認する。
- `?` キーでヘルプパネルを開き、上記の変更が正しく反映されていることを確認する。
- `/`・`n`・`t`・`g`・`Esc`・`Enter`・`e` が、これまで通り動作することを確認する（回帰確認）。
