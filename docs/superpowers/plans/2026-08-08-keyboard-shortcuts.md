# ショートカットキーの見直し Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `s`（カード一覧へフォーカス）ショートカットを廃止し、代わりに矢印キーを未フォーカス状態でも常に使えるようにする。さらに `Home`/`End` キーで一覧の先頭/末尾カードへ直接フォーカスできるようにする（`bookmarks.html` / `bookmarks-filesync.html` の両方）。

**Architecture:** `handleGlobalShortcut()` から `case 's':` を削除し、代わりに矢印キー4種（カードに既にフォーカス中なら `handleCardShortcut` 側の既存の移動処理にフォールスルーさせ、未フォーカスなら一覧の起点カードへフォーカスを移す）と `Home`/`End`（常に先頭/末尾カードへフォーカス）の分岐を追加する。`handleCardShortcut()` 自体は変更しない（`c`/`p`/`Delete`/`Backspace`/`e`/`Enter`/矢印キーは既存のまま）。ヘルプパネルの説明も合わせて更新する。

**Tech Stack:** 素のHTML/CSS/JS（ビルドなし、フレームワークなし）。自動テストは存在しないため、ブラウザでの手動確認（またはブラウザが使えない場合の静的確認）で検証する。

## Global Constraints

- 対象ファイルは `bookmarks.html` と `bookmarks-filesync.html` の2つのみ。
- `/`・`n`・`t`・`g`・`?`・`Esc`・`Enter`・`e`・`c`・`p`・`Delete`/`Backspace` の割り当て・挙動は一切変更しない。`handleCardShortcut()` 関数の内容そのものも変更しない。
- カード内の「コピー」「削除」ボタン自体のクリックハンドラ・`tabindex="-1"` 属性は変更しない（これらのボタンはTab移動対象外であり、`c`/`p`/`Delete`/`Backspace` ショートカットがキーボードから実行する唯一の手段であるため、廃止しない）。
- `Escape` キーの階層的クローズ処理（`closeTopmostLayer()`）、モーダル表示中のグローバルショートカット無効化ガードは変更しない。

---

### Task 1: `bookmarks.html` のショートカットキーを変更する

**Files:**
- Modify: `bookmarks.html:2577-2581`（`case 's':` を削除し、矢印キー4種と `Home`/`End` の分岐を追加）
- Modify: `bookmarks.html:1075`, `bookmarks.html:1078-1079`（ヘルプパネルの更新）

**Interfaces:**
- Consumes: 既存の `getNavigableCards()`（`bookmarks.html:2658` 付近、`Array.prototype.slice.call(document.querySelectorAll('#group-sections .card[tabindex]'))` を返す）
- Produces: なし（既存関数の内部ロジック変更のみ。Task 2ではこれと同一の変更を `bookmarks-filesync.html` にも適用する）

- [ ] **Step 1: `handleGlobalShortcut()` の `case 's':` を削除し、矢印キーと `Home`/`End` の分岐を追加する**

`bookmarks.html:2551-2584` は現在:

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
      case 's':
        e.preventDefault();
        var focusTarget = document.querySelector('#group-sections .card[tabindex="0"]');
        if (focusTarget) focusTarget.focus();
        return true;
    }
    return false;
  }
```

次のように変更する（`case 's':` の3行を削除し、矢印キー4種と `Home`/`End` の分岐を `case '?':` の後に追加する）:

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

`getNavigableCards()` はこの時点でまだ関数宣言としては後方（`bookmarks.html` 内でさらに下の行）に書かれているが、`function` 宣言は巻き上げられるため、この位置から呼び出しても問題ない。

- [ ] **Step 2: ヘルプパネルを更新する**

`bookmarks.html:1070-1084` は現在:

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

次のように変更する（矢印キーの説明文を更新し、`Home`/`End` の行を追加、`s` の行を削除。それ以外の行の順序・内容は変更しない）:

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
    <dt><kbd>Delete</kbd></dt><dd>フォーカス中のカードを削除</dd>
    <dt><kbd>t</kbd></dt><dd>タグタブに切り替えてタグ絞り込みナビへフォーカス</dd>
    <dt><kbd>g</kbd></dt><dd>グループタブに切り替えてグループ絞り込みナビへフォーカス</dd>
    <dt><kbd>c</kbd></dt><dd>フォーカス中のカードのログインIDをコピー</dd>
    <dt><kbd>p</kbd></dt><dd>フォーカス中のカードのパスワードをコピー</dd>
  </dl>
```

- [ ] **Step 3: `bookmarks.html` を手動確認する**

自動テストは存在しないため、ブラウザでの手動確認を行う（ヘッドレス環境でブラウザが使えない場合は、後述の代替確認で置き換える）。

1. `bookmarks.html` を開き、ページ読み込み直後（どのカードにもフォーカスしていない状態、例えばクリックで検索欄などを一度フォーカスしたあとページの何もない部分をクリックして解除するか、単純にロード直後）で `↓` キーを押す。一覧の起点カード（ロービングタブインデックス対象。多くの場合は先頭のカード）にフォーカスが移り、カードに視覚的なフォーカスリング（既存の `.card:focus` スタイル）が表示されることを確認する。
2. 続けて `↓`/`→`/`↑`/`←` を押し、カード間をこれまで通り移動できることを確認する。
3. 一度カード一覧の外（例えば検索欄）をクリックしてフォーカスを外し、`Home` キーを押す。一覧の先頭カードにフォーカスが移ることを確認する。同様に `End` キーで末尾カードにフォーカスが移ることを確認する。
4. カードにフォーカスしている状態から `Home`/`End` を押しても、それぞれ先頭/末尾カードへ移動することを確認する。
5. `s` キーを押しても何も起きないことを確認する。
6. カードにフォーカスした状態で `c`（ログインIDコピー）・`p`（パスワードコピー）・`Delete`（削除）・`e`（編集）・`Enter`（開く）が、これまで通り動作することを確認する（回帰確認）。
7. `?` キーでヘルプパネルを開き、`s` の行が無いこと、矢印キーの説明が更新されていること、`Home`/`End` の行が追加されていることを確認する。
8. `/`・`n`・`t`・`g`・`Esc` が、これまで通り動作することを確認する（回帰確認）。

ヘッドレス環境等でブラウザ（`DISPLAY`）が使えない場合は、代わりに次の静的確認を行い、その旨をレポートに明記する:
- `grep -n "case 's':" bookmarks.html` がヒットしないこと（`handleGlobalShortcut()` 内から削除されていること）
- `grep -n "case 'Home':\|case 'End':" bookmarks.html` がそれぞれヒットすること
- `grep -n "ArrowDown\|ArrowRight\|ArrowUp\|ArrowLeft" bookmarks.html` で、`handleGlobalShortcut()` と `handleCardShortcut()` の両方に矢印キーの分岐が存在することを確認する
- `grep -n "kbd>Home</kbd\|kbd>End</kbd\|kbd>s</kbd" bookmarks.html` で、ヘルプパネルに `Home`/`End` の行があり `s` の行が無いことを確認する
- `handleCardShortcut()` 関数の内容が変更されていないこと（`case 'c':`, `case 'p':`, `case 'Delete':`, `case 'Backspace':`, `case 'e':`, `case 'Enter':` がすべて残っていること）を確認する
- ブラウザでのインタラクティブ確認（上記1〜8、特に実際のフォーカス移動の視覚確認）は未実施であり、マージ前に人間による実ブラウザでの確認が必須である旨をレポートに明記する。

- [ ] **Step 4: Commit**

```bash
git add bookmarks.html
git commit -m "feat: rework keyboard shortcuts (drop s, always-on arrow keys, add Home/End) in bookmarks.html"
```

---

### Task 2: `bookmarks-filesync.html` に同一の変更を適用する

**Files:**
- Modify: `bookmarks-filesync.html:2881-2885`（`case 's':` を削除し、矢印キー4種と `Home`/`End` の分岐を追加、Task 1と同一内容）
- Modify: `bookmarks-filesync.html:1175`, `bookmarks-filesync.html:1178-1179`（ヘルプパネルの更新、Task 1と同一内容）

**Interfaces:**
- Consumes: 既存の `getNavigableCards()`（`bookmarks-filesync.html:2962` 付近、Task 1と同一シグネチャ）
- Produces: なし（本プラン最終タスク）

- [ ] **Step 1: `handleGlobalShortcut()` の `case 's':` を削除し、矢印キーと `Home`/`End` の分岐を追加する**

`bookmarks-filesync.html:2855-2888` は現在:

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
      case 's':
        e.preventDefault();
        var focusTarget = document.querySelector('#group-sections .card[tabindex="0"]');
        if (focusTarget) focusTarget.focus();
        return true;
    }
    return false;
  }
```

Task 1 Step 1 と全く同じ内容に変更する:

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

- [ ] **Step 2: ヘルプパネルを更新する**

`bookmarks-filesync.html:1170-1184` は現在:

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

Task 1 Step 2 と全く同じ内容に変更する:

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
    <dt><kbd>Delete</kbd></dt><dd>フォーカス中のカードを削除</dd>
    <dt><kbd>t</kbd></dt><dd>タグタブに切り替えてタグ絞り込みナビへフォーカス</dd>
    <dt><kbd>g</kbd></dt><dd>グループタブに切り替えてグループ絞り込みナビへフォーカス</dd>
    <dt><kbd>c</kbd></dt><dd>フォーカス中のカードのログインIDをコピー</dd>
    <dt><kbd>p</kbd></dt><dd>フォーカス中のカードのパスワードをコピー</dd>
  </dl>
```

- [ ] **Step 3: `bookmarks-filesync.html` を手動確認する**

`bookmarks-filesync.html` は Chrome または Edge で開くこと（File System Access API 非対応ブラウザでは「未対応」画面が出るため、確認自体ができない）。

1. Chrome/Edge で `bookmarks-filesync.html` を開き、ファイルを接続する（既存のブックマークJSONでも、空の `[]` を保存した新規ファイルでもよい。空の場合はカードが無いため、事前に「+追加」から1〜2件追加しておく）。
2. Task 1 Step 3 の 1〜8 と同じ確認項目をすべて実施する。

ヘッドレス環境等でブラウザが使えない場合は、Task 1 Step 3 と同様の静的確認（`grep -n "case 's':"` がヒットしないこと、`Home`/`End`/矢印キーの分岐が存在すること、ヘルプパネルの更新内容、`handleCardShortcut()` が変更されていないこと）を `bookmarks-filesync.html` に対して行う。

未実施の項目（実ブラウザでのインタラクティブ確認）をレポートに明記する。

- [ ] **Step 4: 両ファイルの対称性を確認する**

次のコマンドで、両ファイルの `handleGlobalShortcut()` 内の変更箇所とヘルプパネルの変更箇所が一致することを確認する:

```bash
for pattern in "case 'Home':" "case 'End':" "kbd>Home</kbd" "kbd>End</kbd" "一覧へ入る"; do
  a=$(grep -c -- "$pattern" bookmarks.html)
  b=$(grep -c -- "$pattern" bookmarks-filesync.html)
  if [ "$a" != "$b" ]; then echo "MISMATCH: $pattern (bookmarks.html=$a, bookmarks-filesync.html=$b)"; fi
done
echo "checked"
grep -c "case 's':" bookmarks.html bookmarks-filesync.html   # 両方とも0であることを期待
grep -c "kbd>s</kbd" bookmarks.html bookmarks-filesync.html  # 両方とも0であることを期待
```

`checked` の前に `MISMATCH` が出ず、`case 's':` と `kbd>s</kbd` のカウントが両ファイルとも `0` であることを確認する。

- [ ] **Step 5: Commit**

```bash
git add bookmarks-filesync.html
git commit -m "feat: rework keyboard shortcuts (drop s, always-on arrow keys, add Home/End) in bookmarks-filesync.html"
```
