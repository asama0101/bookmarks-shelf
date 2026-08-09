# 矢印キーのグループ間移動対応 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** カードにフォーカスしている状態での矢印キーの役割を、↑↓=グループ間移動、←→=グループ内カード移動、に変更する。

**Architecture:** 両ファイルの `moveCardFocus(card, direction)` を、グループ区画内限定の `moveCardFocusWithinGroup()` と、区画をまたいで循環移動する `moveCardFocusToGroup()` の2関数に分割する。呼び出し元の `handleCardShortcut()` の矢印キー分岐をArrowUp/Down/Left/Rightそれぞれ独立させ、新関数へ振り分ける。ヘルプパネルの矢印キー説明も1行から2行に分ける。

**Tech Stack:** Vanilla JS (ES5-style IIFE)、ビルドなし。自動テストは存在しないため、ブラウザでの手動確認が検証手段。

## Global Constraints

- 対象は `bookmarks.html` と `bookmarks-filesync.html` の両方。該当ロジックは完全同一のため、同じ変更を両方に適用する(spec: `docs/superpowers/specs/2026-08-09-arrow-key-group-navigation-design.md`)。
- 変更対象はカードグリッド(`#group-sections`)にフォーカスがある場面の矢印キーのみ。サイドバーnav・モーダル・フォーム内・Home/Endの挙動は変更しない。
- ↑↓の移動先は区画の先頭カード固定。表示中カード0件の区画はスキップ。最初/最後の区画では表示順で循環する。カードを持つ区画が1つしかない場合は何もしない。
- ←→は現在の区画内のみで移動。区画の端では循環しない(何も起きない)。
- 自動テストが存在しないプロジェクトのため、各タスクの検証はブラウザでの手動確認で行う。

---

### Task 1: `bookmarks.html` の矢印キー処理を分割する

**Files:**
- Modify: `bookmarks.html:2680-2686`(`moveCardFocus` 定義を置き換え)
- Modify: `bookmarks.html:2588-2598`(`handleCardShortcut` の矢印キー分岐)

**Interfaces:**
- Consumes: 既存の `getNavigableCards()`(`bookmarks.html:2662-2664`、変更しない)
- Produces: `moveCardFocusWithinGroup(card, direction)`、`moveCardFocusToGroup(card, direction)` — Task 2(filesync側)は同名・同シグネチャで移植するため、ここで確定した関数名・実装をそのまま踏襲すること。

- [ ] **Step 1: `moveCardFocus` を2つの新関数に置き換える**

`bookmarks.html:2680-2686` の以下のコードを:

```js
  // 表示中の全カードをDOM順の1本のリストとして扱い、前後のカードへフォーカスを移す(区画をまたぐ移動も含む)。
  // tabindexの張り替えとfocusedCardIdの更新はfocusinハンドラ側で行われる。
  function moveCardFocus(card, direction) {
    var cards = getNavigableCards();
    var next = cards[cards.indexOf(card) + direction];
    if (next) next.focus();
  }
```

次のコードに置き換える:

```js
  // 同じグループ区画内のカードのみを対象に、DOM順で前後移動する(区画をまたがない。端では何も起きない)。
  // tabindexの張り替えとfocusedCardIdの更新はfocusinハンドラ側で行われる。
  function moveCardFocusWithinGroup(card, direction) {
    var section = card.closest('.group-section');
    if (!section) return;
    var cards = Array.prototype.slice.call(section.querySelectorAll('.card[tabindex]'));
    var next = cards[cards.indexOf(card) + direction];
    if (next) next.focus();
  }

  // 表示中のグループ区画(表示順)を1つ進め/戻し、移動先区画の先頭カードへフォーカスする。
  // 表示中カードが0件の区画はスキップし、最初/最後の区画では反対側の端へ循環する。
  // カードを持つ区画が1つしかない場合は何も起きない。
  function moveCardFocusToGroup(card, direction) {
    var currentSection = card.closest('.group-section');
    if (!currentSection) return;
    var sections = Array.prototype.slice.call(document.querySelectorAll('#group-sections .group-section'))
      .filter(function (s) { return s.querySelector('.card[tabindex]'); });
    if (sections.length <= 1) return;
    var idx = sections.indexOf(currentSection);
    if (idx === -1) return;
    var targetIdx = (idx + direction + sections.length) % sections.length;
    var targetCard = sections[targetIdx].querySelector('.card[tabindex]');
    if (targetCard) targetCard.focus();
  }
```

- [ ] **Step 2: `handleCardShortcut()` の矢印キー分岐を書き換える**

`bookmarks.html:2588-2598` の以下のコードを:

```js
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
```

次のコードに置き換える:

```js
      case 'ArrowRight':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocusWithinGroup(card, 1);
        return;
      case 'ArrowLeft':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocusWithinGroup(card, -1);
        return;
      case 'ArrowDown':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocusToGroup(card, 1);
        return;
      case 'ArrowUp':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocusToGroup(card, -1);
        return;
```

- [ ] **Step 3: `moveCardFocus` の呼び出しが他に残っていないことを確認する**

Run: `grep -n "moveCardFocus(" bookmarks.html`
Expected: `moveCardFocusWithinGroup(` と `moveCardFocusToGroup(` の呼び出し・定義のみがヒットし、`moveCardFocus(card,` という古い呼び出しは残っていない。

- [ ] **Step 4: コミット**

```bash
git add bookmarks.html
git commit -m "$(cat <<'EOF'
feat: bookmarks.htmlの矢印キーをグループ間/グループ内移動に分割

↑↓はグループ区画をまたいで先頭カードへ、←→は同じ区画内でカードを
前後移動するように変更。moveCardFocus()をmoveCardFocusWithinGroup()と
moveCardFocusToGroup()に分割した。
EOF
)"
```

---

### Task 2: `bookmarks-filesync.html` に同じ変更を適用する

**Files:**
- Modify: `bookmarks-filesync.html:2982-2988`(`moveCardFocus` 定義を置き換え)
- Modify: `bookmarks-filesync.html:2890-2900`(`handleCardShortcut` の矢印キー分岐)

**Interfaces:**
- Consumes: 既存の `getNavigableCards()`(filesync側、変更しない)
- Produces: Task 1と同名・同シグネチャの `moveCardFocusWithinGroup(card, direction)`、`moveCardFocusToGroup(card, direction)`(filesync側の関数として独立に定義。2ファイルは共通モジュールを持たないため、コードは重複するが実装内容はTask 1と完全に一致させる)。

- [ ] **Step 1: `moveCardFocus` を2つの新関数に置き換える**

`bookmarks-filesync.html:2982-2988` の以下のコードを:

```js
  // 表示中の全カードをDOM順の1本のリストとして扱い、前後のカードへフォーカスを移す(区画をまたぐ移動も含む)。
  // tabindexの張り替えとfocusedCardIdの更新はfocusinハンドラ側で行われる。
  function moveCardFocus(card, direction) {
    var cards = getNavigableCards();
    var next = cards[cards.indexOf(card) + direction];
    if (next) next.focus();
  }
```

次のコードに置き換える(Task 1と同一内容):

```js
  // 同じグループ区画内のカードのみを対象に、DOM順で前後移動する(区画をまたがない。端では何も起きない)。
  // tabindexの張り替えとfocusedCardIdの更新はfocusinハンドラ側で行われる。
  function moveCardFocusWithinGroup(card, direction) {
    var section = card.closest('.group-section');
    if (!section) return;
    var cards = Array.prototype.slice.call(section.querySelectorAll('.card[tabindex]'));
    var next = cards[cards.indexOf(card) + direction];
    if (next) next.focus();
  }

  // 表示中のグループ区画(表示順)を1つ進め/戻し、移動先区画の先頭カードへフォーカスする。
  // 表示中カードが0件の区画はスキップし、最初/最後の区画では反対側の端へ循環する。
  // カードを持つ区画が1つしかない場合は何も起きない。
  function moveCardFocusToGroup(card, direction) {
    var currentSection = card.closest('.group-section');
    if (!currentSection) return;
    var sections = Array.prototype.slice.call(document.querySelectorAll('#group-sections .group-section'))
      .filter(function (s) { return s.querySelector('.card[tabindex]'); });
    if (sections.length <= 1) return;
    var idx = sections.indexOf(currentSection);
    if (idx === -1) return;
    var targetIdx = (idx + direction + sections.length) % sections.length;
    var targetCard = sections[targetIdx].querySelector('.card[tabindex]');
    if (targetCard) targetCard.focus();
  }
```

- [ ] **Step 2: `handleCardShortcut()` の矢印キー分岐を書き換える**

`bookmarks-filesync.html:2890-2900` の以下のコードを:

```js
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
```

次のコードに置き換える(Task 1と同一内容):

```js
      case 'ArrowRight':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocusWithinGroup(card, 1);
        return;
      case 'ArrowLeft':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocusWithinGroup(card, -1);
        return;
      case 'ArrowDown':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocusToGroup(card, 1);
        return;
      case 'ArrowUp':
        e.preventDefault(); // 一覧内の移動中にページがスクロールしないようにする
        moveCardFocusToGroup(card, -1);
        return;
```

- [ ] **Step 3: `moveCardFocus` の呼び出しが他に残っていないことを確認する**

Run: `grep -n "moveCardFocus(" bookmarks-filesync.html`
Expected: `moveCardFocusWithinGroup(` と `moveCardFocusToGroup(` の呼び出し・定義のみがヒットし、`moveCardFocus(card,` という古い呼び出しは残っていない。

- [ ] **Step 4: コミット**

```bash
git add bookmarks-filesync.html
git commit -m "$(cat <<'EOF'
feat: bookmarks-filesync.htmlの矢印キーをグループ間/グループ内移動に分割

bookmarks.htmlと同じ変更をfilesync版にも適用。
moveCardFocus()をmoveCardFocusWithinGroup()とmoveCardFocusToGroup()に
分割した。
EOF
)"
```

---

### Task 3: ヘルプパネルの説明文を更新する

**Files:**
- Modify: `bookmarks.html:1074`
- Modify: `bookmarks-filesync.html:1174`

**Interfaces:**
- Consumes: なし
- Produces: なし(表示文言のみの変更)

- [ ] **Step 1: `bookmarks.html` のヘルプパネル矢印キー行を分割する**

`bookmarks.html:1074` の以下の行を:

```html
    <dt><kbd>&#8593;</kbd> <kbd>&#8595;</kbd> <kbd>&#8592;</kbd> <kbd>&#8594;</kbd></dt><dd>カード間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
```

次の2行に置き換える:

```html
    <dt><kbd>&#8593;</kbd> <kbd>&#8595;</kbd></dt><dd>グループ間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
    <dt><kbd>&#8592;</kbd> <kbd>&#8594;</kbd></dt><dd>同じグループ内でカード間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
```

- [ ] **Step 2: `bookmarks-filesync.html` のヘルプパネル矢印キー行を分割する**

`bookmarks-filesync.html:1174` に対して、Step 1と同一の置き換えを行う。

- [ ] **Step 3: コミット**

```bash
git add bookmarks.html bookmarks-filesync.html
git commit -m "$(cat <<'EOF'
docs: ヘルプパネルの矢印キー説明をグループ間/グループ内移動に更新

矢印キーの役割変更(Task 1, 2)に合わせて、ヘルプパネルの説明文言を
↑↓とその他←→の2行に分割した。
EOF
)"
```

---

### Task 4: ブラウザでの手動検証

**Files:**
- なし(検証のみ。修正が必要になった場合はTask 1〜3の該当箇所に戻って直す)

**Interfaces:**
- Consumes: Task 1〜3で完成した両ファイル
- Produces: なし

- [ ] **Step 1: `bookmarks.html` を開く**

Run: `xdg-open bookmarks.html`

- [ ] **Step 2: 複数グループ+未分類を含むデータを用意する**

まだ複数グループのデータがなければ、追加パネルから2〜3個のグループ(例: 「仕事」「趣味」)にそれぞれ2件以上のブックマークを追加し、グループを付けないブックマークも1件以上追加する(未分類区画の確認用)。

- [ ] **Step 3: ↓でグループ間を移動できることを確認する**

いずれかのグループのカードをクリックしてフォーカスし、↓キーを押す。次の区画(表示順で1つ後ろ、未分類は常に最後)の先頭カードにフォーカスが移ることを目視確認する。

Expected: フォーカスの枠線が次の区画の最初のカードに移る。

- [ ] **Step 4: ↑でグループ間を逆方向に移動できることを確認する**

Step 3でフォーカスしたカードから↑キーを押し、元の区画の先頭カードに戻ることを確認する。

Expected: フォーカスが1つ前の区画の先頭カードに移る。

- [ ] **Step 5: 最初/最後の区画での循環を確認する**

一覧の最初の区画の先頭カードにフォーカスした状態で↑を押すと、最後の区画(未分類)の先頭カードにフォーカスが移ることを確認する。逆に、最後の区画で↓を押すと最初の区画に戻ることを確認する。

Expected: 両方向とも反対側の端へ循環する。

- [ ] **Step 6: タグ絞り込みで空になった区画がスキップされることを確認する**

いずれかのグループの全ブックマークに共通しないタグで絞り込み、そのグループの区画が0件表示になる状態を作る。0件でない区画のカードにフォーカスし、↑↓でその0件区画を素通りして次にカードのある区画へ移動することを確認する。

Expected: 0件の区画にはフォーカスが止まらない。

- [ ] **Step 7: グループフィルタで1区画のみのとき↑↓が何もしないことを確認する**

サイドバーのグループタブから特定のグループを選んで絞り込み、1区画のみ表示させる。その区画内のカードにフォーカスし、↑↓を押しても何も起きない(フォーカスが動かない)ことを確認する。フィルタを解除する。

Expected: フォーカスが移動しない。

- [ ] **Step 8: ←→で同じグループ内のカード移動を確認する**

複数カードを持つグループ区画内のカードにフォーカスし、→を押すと同じ区画内の次のカードへ、←を押すと前のカードへ移動することを確認する。区画内の最初のカードで←、最後のカードで→を押しても他区画へは移動しない(何も起きない)ことを確認する。

Expected: フォーカスが区画内に留まる。

- [ ] **Step 9: 未フォーカス状態からの「一覧へ入る」動作が回帰していないことを確認する**

ページを再読み込みし、どのカードにもフォーカスしていない状態で↑↓←→のいずれかを押す。一覧の起点カード(ロービングtabindexの対象)にフォーカスが入ることを確認する(回帰確認)。

Expected: 従来通りフォーカスが一覧に入る。

- [ ] **Step 10: 他のショートカットの回帰確認**

カードにフォーカスした状態で `Home`/`End`/`c`/`p`/`Delete`/`e`/`Enter` がこれまで通り動作することを確認する。

Expected: すべて従来通り動作する。

- [ ] **Step 11: ヘルプパネルの表示を確認する**

`?`キーでヘルプパネルを開き、矢印キーの説明が「↑↓: グループ間のフォーカス移動」「←→: 同じグループ内でカード間のフォーカス移動」の2行に分かれて表示されることを確認する。

Expected: Task 3で書いた文言通りに表示される。

- [ ] **Step 12: `bookmarks-filesync.html` でStep 1〜11を繰り返す**

Run: `xdg-open bookmarks-filesync.html`

Chrome/Edgeで開き、ファイル読み込み(または新規作成)を行った上でStep 2〜11と同じ確認を行う。

Expected: `bookmarks.html` と同じ挙動になる。

---
