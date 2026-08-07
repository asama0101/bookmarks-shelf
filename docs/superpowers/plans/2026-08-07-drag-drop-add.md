# URLドラッグ&ドロップによる追加 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ブラウザの別タブ・アドレスバーからリンクをドラッグし、既存の「+追加」ボタンにドロップするだけで、タグ・グループ・ログイン情報・メモを付けない最小限のブックマークを即座に追加できるようにする（`bookmarks.html` / `bookmarks-filesync.html` の両方）。

**Architecture:** 新規UI要素は追加せず、既存の「+追加」ボタン（`#toggle-add-btn`）に `dragover`/`dragleave`/`drop` リスナーを追加する。新規の独立した共通関数 `quickAddBookmark(url)` と、`DataTransfer` からURL文字列を取り出す `extractUrlFromDrop(dataTransfer)` を軸に実装する。既存のカード並べ替え・グループ移動用ドラッグ&ドロップ（`bindDragReorder`, `bindGroupSectionDragEvents` 等）はDOM上重ならないため衝突しないが、カード並べ替え中の誤反応だけは `draggedCardId` を見て防ぐ。既存の「+追加」ボタンのクリックハンドラ・フルフォームは変更しない。

**Tech Stack:** 素のHTML/CSS/JS（ビルドなし、フレームワークなし）。HTML5 Drag and Drop API（`dataTransfer.getData('text/uri-list')` / `'text/plain'`）を使用する。自動テストは存在しないため、ブラウザでの手動確認（またはブラウザが使えない場合の静的確認）で検証する。

## Global Constraints

- 対象ファイルは `bookmarks.html` と `bookmarks-filesync.html` の2つのみ。
- 新規追加されるブックマークは常に `tags: []`, `group: null`, `loginId: null`, `loginPassword: null`, `memo: null`。タイトルのみ `deriveTentativeTitle()` で自動生成する。
- URLとして認識できないものがドロップされた場合は何もしない（エラー表示・フォールバックパネル表示は行わない）。
- 新規UI要素（新しいボタン等）は追加しない。既存の「+追加」ボタン（`#toggle-add-btn`）のみを使う。
- 既存の「+追加」ボタンのクリックハンドラ（フォームのトグル開閉）、既存の追加フォームのsubmitハンドラ、既存のカード並べ替え・グループ移動用ドラッグ&ドロップのロジックは一切変更しない。
- `bookmarks-filesync.html` では、ファイル未接続の間（`fileHandle` が `null` の間）はこの機能も無効化する（`dragover` と `drop` の両方に `fileHandle` ガードを入れる）。
- キーボードショートカットの割り当ては変更しない。

---

### Task 1: `bookmarks.html` に「+追加」ボタンへのURLドラッグ&ドロップを実装する

**Files:**
- Modify: `bookmarks.html:138`（CSS追加）
- Modify: `bookmarks.html:1002`（`title` 属性追加）
- Modify: `bookmarks.html:1331-1338`（`quickAddBookmark()` 追加）
- Modify: `bookmarks.html:2271`（`extractUrlFromDrop()` とドロップイベントリスナー追加）

**Interfaces:**
- Consumes: 既存の `classifyUrl(raw)`, `deriveDomain(url, scheme)`, `deriveTentativeTitle(url, scheme)`, `makeId()`, `saveData(data)`, `getGroupOrderForName(name)`, `render()`, `draggedCardId`（モジュール変数）
- Produces: `quickAddBookmark(url)`（成功時true、無効なURLならfalseを返す）, `extractUrlFromDrop(dataTransfer)` — Task 2ではこれらと同名・同シグネチャの関数を `bookmarks-filesync.html` にも定義する（`bookmarks-filesync.html` 側は `dragover`/`drop` リスナーに `fileHandle` ガードが追加で入る点がTask 1と異なる）

- [ ] **Step 1: CSSを追加する**

`bookmarks.html:134-138` は現在:

```css
  .btn-amber {
    background: var(--amber);
    color: var(--amber-ink);
  }
  .btn-amber:hover { background: #eeb857; }
```

この直後に次のCSSを追加する:

```css
  #toggle-add-btn.add-btn-drop-target { box-shadow: 0 0 0 3px var(--amber-ink); }
```

- [ ] **Step 2: 「+追加」ボタンに `title` 属性を追加する**

`bookmarks.html:1002` は現在:

```html
    <button class="btn btn-amber" id="toggle-add-btn" type="button">+ 追加</button>
```

次のように変更する:

```html
    <button class="btn btn-amber" id="toggle-add-btn" type="button" title="URLをドラッグ&amp;ドロップして追加">+ 追加</button>
```

- [ ] **Step 3: `quickAddBookmark()` を追加する**

`bookmarks.html:1331-1338` は現在:

```js
  function getGroupOrderForName(name) {
    if (!name) return 0;
    var existing = bookmarks.find(function (b) { return b.group === name; });
    if (existing) return existing.groupOrder;
    var max = -1;
    bookmarks.forEach(function (b) { if (b.group && b.groupOrder > max) max = b.groupOrder; });
    return max + 1;
  }
```

この直後に次の関数を追加する:

```js

  // 「+追加」ボタンへのURLドラッグ&ドロップで使う最小限のブックマーク作成処理。
  // タグ・グループ・ログイン情報・メモは付けず、タイトルのみ自動生成する(必要なら追加後に編集フォームで補足する)。
  function quickAddBookmark(url) {
    var scheme = classifyUrl(url);
    if (!scheme) return false;
    bookmarks.push({
      id: makeId(),
      url: url,
      title: deriveTentativeTitle(url, scheme),
      tags: [],
      group: null,
      groupOrder: getGroupOrderForName(null),
      loginId: null,
      loginPassword: null,
      memo: null,
      scheme: scheme,
      domain: deriveDomain(url, scheme),
      order: bookmarks.length
    });
    saveData(bookmarks);
    render();
    return true;
  }
```

- [ ] **Step 4: `extractUrlFromDrop()` とドロップイベントリスナーを追加する**

`bookmarks.html:2263-2271` は現在:

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

この直後（`/* ---------- 追加フォーム ---------- */` コメントの手前）に次のブロックを追加する:

```js

  /* ---------- 「+追加」ボタンへのURLドラッグ&ドロップ ---------- */

  // ドロップされたDataTransferから実際のURL文字列を取り出す。
  // ブラウザがリンクをドラッグする際に設定するtext/uri-list(複数行になりうる。
  // #で始まる行はコメントなので除外し、最初の有効な行を採用)を優先し、無ければtext/plainにフォールバックする。
  function extractUrlFromDrop(dataTransfer) {
    var uriList = dataTransfer.getData('text/uri-list');
    if (uriList) {
      var lines = uriList.split(/\r?\n/).map(function (l) { return l.trim(); })
        .filter(function (l) { return l && l.charAt(0) !== '#'; });
      if (lines.length) return lines[0];
    }
    return dataTransfer.getData('text/plain').trim();
  }

  document.getElementById('toggle-add-btn').addEventListener('dragover', function (e) {
    if (draggedCardId) return; // カード並べ替え中の誤反応を防ぐ
    var types = e.dataTransfer.types;
    if (types.indexOf('text/uri-list') === -1 && types.indexOf('text/plain') === -1) return;
    e.preventDefault();
    this.classList.add('add-btn-drop-target');
  });

  document.getElementById('toggle-add-btn').addEventListener('dragleave', function () {
    this.classList.remove('add-btn-drop-target');
  });

  document.getElementById('toggle-add-btn').addEventListener('drop', function (e) {
    e.preventDefault();
    this.classList.remove('add-btn-drop-target');
    var url = extractUrlFromDrop(e.dataTransfer);
    quickAddBookmark(url);
  });
```

- [ ] **Step 5: `bookmarks.html` を手動確認する**

自動テストは存在しないため、ブラウザでの手動確認を行う（ヘッドレス環境でブラウザが使えない場合は、後述の代替確認で置き換える）。

1. `bookmarks.html` を開く。別のブラウザタブでどこか適当なページを開き、そのページ内のリンク（または、そのタブのアドレスバーのアイコン）を `bookmarks.html` の「+追加」ボタンへドラッグする。ボタンの上に重なった時点で枠線（視覚フィードバック）が表示されることを確認する。
2. そのままドロップし、即座に新しいカード（未分類・タグ無し・タイトル自動生成）が一覧に追加されることを確認する。
3. テキストエディタ等で適当な文章を選択してドラッグし、「+追加」ボタンにドロップする。何も起きない（新しいカードが追加されない）ことを確認する。
4. 既存のブックマークカードをドラッグして並べ替えている最中に、ドラッグ中のカードが「+追加」ボタンの上を通過しても、ボタンに枠線が表示されない（誤反応しない）ことを確認する。
5. 既存のカード並べ替え・グループ間へのドラッグ移動が、これまで通り正常に動作することを確認する（回帰確認）。
6. 既存の「+追加」ボタンをクリックし、通常のフル入力フォーム（タグ・グループ・ログイン情報・メモを含む）での追加が、これまで通り動作することを確認する（回帰確認）。

ヘッドレス環境等でブラウザ（`DISPLAY`）が使えない場合は、代わりに次の静的確認を行い、その旨をレポートに明記する:
- `grep -n "add-btn-drop-target\|quickAddBookmark\|extractUrlFromDrop" bookmarks.html` で、Step 1〜4で追加したすべてのクラス名・関数名がCSS定義・HTML参照・JS参照で過不足なく対応していることを確認する。
- `quickAddBookmark`, `extractUrlFromDrop` の各識別子が過不足なく1回だけ定義されていることを確認する。
- ブラウザでのインタラクティブ確認（上記1〜6、特に実際のドラッグ&ドロップ操作とそのdataTransferの中身）は未実施であり、マージ前に人間による実ブラウザでの確認が必須である旨をレポートに明記する。

- [ ] **Step 6: Commit**

```bash
git add bookmarks.html
git commit -m "feat: add drag-and-drop URL quick-add to the toggle-add-btn in bookmarks.html"
```

---

### Task 2: `bookmarks-filesync.html` に同等の機能を実装する（未接続時は無効化する）

Task 1と同じ機能を実装するが、`bookmarks-filesync.html` の「+追加」ボタンはファイル未接続の間 `disabled` になる（`HEADER_BUTTON_IDS` の仕組み）。`disabled` 属性がドラッグ&ドロップイベントの発火を確実に抑止するとは限らないため、`dragover` と `drop` の両方に `fileHandle`（ファイル未接続時は `null`）のガードを明示的に追加する点がTask 1との差分。

**Files:**
- Modify: `bookmarks-filesync.html:147`（CSS追加、Task 1と同一内容）
- Modify: `bookmarks-filesync.html:1080`（`title` 属性追加。既存の `disabled` 属性はそのまま残す）
- Modify: `bookmarks-filesync.html:1633-1640`（`quickAddBookmark()` 追加、Task 1と同一内容）
- Modify: `bookmarks-filesync.html:2573`（`extractUrlFromDrop()`（Task 1と同一内容）とドロップイベントリスナー（`fileHandle` ガード付き）追加）

**Interfaces:**
- Consumes: Task 1と同じ関数群（`classifyUrl`, `deriveDomain`, `deriveTentativeTitle`, `makeId`, `saveData`, `getGroupOrderForName`, `render`）の `bookmarks-filesync.html` 側の定義。加えて既存の `fileHandle`（`bookmarks-filesync.html:1268`で宣言される、ファイル未接続時は `null`）、`draggedCardId`（`bookmarks-filesync.html:2327`で宣言される既存のモジュール変数）。
- Produces: なし（本プラン最終タスク）

- [ ] **Step 1: CSSを追加する**

`bookmarks-filesync.html:143-147` は現在:

```css
  .btn-amber {
    background: var(--amber);
    color: var(--amber-ink);
  }
  .btn-amber:hover { background: #eeb857; }
```

この直後に、Task 1 Step 1 と全く同じ行を追加する:

```css
  #toggle-add-btn.add-btn-drop-target { box-shadow: 0 0 0 3px var(--amber-ink); }
```

- [ ] **Step 2: 「+追加」ボタンに `title` 属性を追加する**

`bookmarks-filesync.html:1080` は現在:

```html
    <button class="btn btn-amber" id="toggle-add-btn" type="button" disabled>+ 追加</button>
```

次のように変更する（既存の `disabled` 属性はそのまま残す）:

```html
    <button class="btn btn-amber" id="toggle-add-btn" type="button" disabled title="URLをドラッグ&amp;ドロップして追加">+ 追加</button>
```

- [ ] **Step 3: `quickAddBookmark()` を追加する**

`bookmarks-filesync.html:1633-1640` は現在:

```js
  function getGroupOrderForName(name) {
    if (!name) return 0;
    var existing = bookmarks.find(function (b) { return b.group === name; });
    if (existing) return existing.groupOrder;
    var max = -1;
    bookmarks.forEach(function (b) { if (b.group && b.groupOrder > max) max = b.groupOrder; });
    return max + 1;
  }
```

この直後に、Task 1 Step 3 と全く同じ関数を追加する:

```js

  // 「+追加」ボタンへのURLドラッグ&ドロップで使う最小限のブックマーク作成処理。
  // タグ・グループ・ログイン情報・メモは付けず、タイトルのみ自動生成する(必要なら追加後に編集フォームで補足する)。
  function quickAddBookmark(url) {
    var scheme = classifyUrl(url);
    if (!scheme) return false;
    bookmarks.push({
      id: makeId(),
      url: url,
      title: deriveTentativeTitle(url, scheme),
      tags: [],
      group: null,
      groupOrder: getGroupOrderForName(null),
      loginId: null,
      loginPassword: null,
      memo: null,
      scheme: scheme,
      domain: deriveDomain(url, scheme),
      order: bookmarks.length
    });
    saveData(bookmarks);
    render();
    return true;
  }
```

- [ ] **Step 4: `extractUrlFromDrop()`（Task 1と同一）とドロップイベントリスナー（`fileHandle` ガード付き）を追加する**

`bookmarks-filesync.html:2565-2573` は現在:

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

この直後（`/* ---------- 追加フォーム ---------- */` コメントの手前）に次のブロックを追加する。`extractUrlFromDrop()` と `dragleave` リスナーはTask 1と全く同じ内容だが、**`dragover` と `drop` の両方に `fileHandle` の未接続ガードが追加で入る点に注意する**（`disabled` 属性だけに頼らず、未接続の間はドラッグ&ドロップでも追加できないようにするため）:

```js

  /* ---------- 「+追加」ボタンへのURLドラッグ&ドロップ ---------- */

  // ドロップされたDataTransferから実際のURL文字列を取り出す。
  // ブラウザがリンクをドラッグする際に設定するtext/uri-list(複数行になりうる。
  // #で始まる行はコメントなので除外し、最初の有効な行を採用)を優先し、無ければtext/plainにフォールバックする。
  function extractUrlFromDrop(dataTransfer) {
    var uriList = dataTransfer.getData('text/uri-list');
    if (uriList) {
      var lines = uriList.split(/\r?\n/).map(function (l) { return l.trim(); })
        .filter(function (l) { return l && l.charAt(0) !== '#'; });
      if (lines.length) return lines[0];
    }
    return dataTransfer.getData('text/plain').trim();
  }

  document.getElementById('toggle-add-btn').addEventListener('dragover', function (e) {
    if (!fileHandle) return; // ファイル未接続の間は無効化する
    if (draggedCardId) return; // カード並べ替え中の誤反応を防ぐ
    var types = e.dataTransfer.types;
    if (types.indexOf('text/uri-list') === -1 && types.indexOf('text/plain') === -1) return;
    e.preventDefault();
    this.classList.add('add-btn-drop-target');
  });

  document.getElementById('toggle-add-btn').addEventListener('dragleave', function () {
    this.classList.remove('add-btn-drop-target');
  });

  document.getElementById('toggle-add-btn').addEventListener('drop', function (e) {
    e.preventDefault();
    this.classList.remove('add-btn-drop-target');
    if (!fileHandle) return; // ファイル未接続の間は無効化する
    var url = extractUrlFromDrop(e.dataTransfer);
    quickAddBookmark(url);
  });
```

- [ ] **Step 5: `bookmarks-filesync.html` を手動確認する**

`bookmarks-filesync.html` は Chrome または Edge で開くこと（File System Access API 非対応ブラウザでは「未対応」画面が出るため、確認自体ができない）。

1. Chrome/Edge で `bookmarks-filesync.html` を開き、ファイルを接続する**前**の状態で、別タブのリンクを「+追加」ボタンへドラッグする。ボタンに枠線が表示されないこと、ドロップしても何も起きないことを確認する。
2. ファイル選択ダイアログでJSONファイルを読み込み、接続する（既存のブックマークJSONでも、空の `[]` を保存した新規ファイルでもよい）。
3. 接続後、Task 1 Step 5 の 1〜6 と同じ確認項目をすべて実施する。
4. 既存の「+追加」ボタンからの通常のフル入力フォームでの追加が、接続後は引き続き動作することを確認する（回帰確認）。

ヘッドレス環境等でブラウザが使えない場合は、Task 1 Step 5 と同様の静的確認（`grep -n "add-btn-drop-target\|quickAddBookmark\|extractUrlFromDrop" bookmarks-filesync.html` で全識別子の対応を確認）に加え、次の filesync 固有の確認を行う:
- `grep -n "if (!fileHandle) return;" bookmarks-filesync.html` で、`dragover` と `drop` の両方のリスナー内にガードが存在することを確認する（`confirmSearchAdd` 等、過去の機能で既に存在するガードと合わせて件数が増えていることも合わせて確認する）。

未実施の項目（実ブラウザでのインタラクティブ確認、特にファイル未接続時の無効化挙動）をレポートに明記する。

- [ ] **Step 6: 意図的な差分・共通部分の対称性を確認する**

次のコマンドで、両ファイルの `quickAddBookmark`・`extractUrlFromDrop`（完全に同一であるべき部分）が一致することを確認する:

```bash
for fn in "quickAddBookmark" "extractUrlFromDrop" "add-btn-drop-target"; do
  a=$(grep -c -- "$fn" bookmarks.html)
  b=$(grep -c -- "$fn" bookmarks-filesync.html)
  if [ "$a" != "$b" ]; then echo "MISMATCH: $fn (bookmarks.html=$a, bookmarks-filesync.html=$b)"; fi
done
echo "checked common identifiers"
```

`checked common identifiers` の前に `MISMATCH` が出ないことを確認する。

続けて、意図的にファイル間で異なるべき部分（`fileHandle` ガード）が想定通りの側にのみ存在することを確認する:

```bash
grep -c "fileHandle" bookmarks.html                # 期待値: 0 (bookmarks.htmlにはfileHandleという概念自体が無い)
```

`bookmarks.html` の上記コマンドが `0` 以外の場合は、意図と異なる実装になっている可能性があるため見直すこと。

- [ ] **Step 7: Commit**

```bash
git add bookmarks-filesync.html
git commit -m "feat: add drag-and-drop URL quick-add to the toggle-add-btn in bookmarks-filesync.html"
```
