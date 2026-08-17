# 検索ヒットグループのみ表示 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 検索ボックスに入力した際、検索ヒットが1件も無いグループのセクション自体を非表示にする(現状は0件でも「該当するブックマークがありません」という空プレースホルダ付きのセクション見出しが常に表示される)。全体で0件のときは、セクションが1つも無い空白画面ではなく、全体向けの「一致しません」メッセージを表示する。

**Architecture:** `renderSections()` の `sectionKeys` 決定直後に、`state.search.trim()` が非空のときだけ「`filtered`(既存の`getSearchTagFiltered()`の結果)に1件もヒットを持たないグループキーを除外する」フィルタを追加する。`state.group`(サイドバーのグループ絞り込み)が特定グループに設定されていて`sectionKeys`が元々1要素の場合も、同じフィルタが素通しで適用され、そのグループの検索ヒットが0件なら同様に非表示になる(分岐追加不要)。`sectionKeys`が空配列になった場合は、`container.innerHTML`に個別セクションの代わりに全体向けメッセージ用の1要素を出力する。タグのみの絞り込み(検索語が空)には一切影響しない。

**Tech Stack:** Vanilla JS (ES5-style IIFE)、ビルドなし。自動テストは存在しないため、ブラウザでの手動確認(claude-in-chromeによるDOM検証)が検証手段。

## Global Constraints

- 対象は `bookmarks.html` と `bookmarks-filesync.html` の両方。該当ロジック(`renderSections()`とその周辺CSS)は完全同一のため、同じ変更を両方に適用する。
- 変更をトリガーするのは `state.search.trim()` が非空のときのみ。`state.tag`(タグ絞り込み)のみの場合、および検索語が空の場合は、これまで通り全グループ+未分類のセクション見出しを常に表示する(0件でも個別の空プレースホルダを出す従来動作を維持)。
- `state.group`(サイドバーのグループ絞り込み。null/''/グループ名の3値)が特定グループに設定されている状態でさらに検索し、そのグループ内のヒットが0件になった場合も、同じ新仕様(セクションを隠して全体メッセージを表示)を適用する。個別分岐は追加しない。
- 未分類(`UNGROUPED = null`)セクションも名前付きグループと全く同じ扱いで、ヒット0件なら同様に非表示にする。
- 全体で表示すべきセクションが0件になった場合、`container.innerHTML`が空文字列にならないようにし、「検索条件に一致するブックマークがありません。」という全体向けメッセージ要素を表示する。既存の`.group-section-empty`と同系統の見た目(点線ボーダー・モノスペースフォント)だが、個別グループの空プレースホルダとは別のCSSクラス(`.search-no-results`)を新設し、パディングを大きめにして全体メッセージであることを視覚的に区別する。
- 自動テストが存在しないプロジェクトのため、各タスクの検証はブラウザでの手動確認で行う。Task 3では、変更前(RED)と変更後(GREEN)の両方をclaude-in-chromeでのDOM評価によって確認し、実装者の報告を鵜呑みにせず独立して再検証する。

---

### Task 1: `bookmarks.html` に検索ヒットグループのみ表示するロジックを実装する

**Files:**
- Modify: `bookmarks.html:1650-1671`(`renderSections()`の`sectionKeys`算出〜`container.innerHTML`代入)
- Modify: `bookmarks.html:716-723`(`.group-section-empty`直後にCSS追加)

**Interfaces:**
- Consumes: 既存の`getSearchTagFiltered()`(変更しない)、`state.search`
- Produces: なし(内部ロジックのみ。新規CSSクラス`.search-no-results`はTask 2で同一定義を踏襲すること)

- [ ] **Step 1: CSSを追加する**

`bookmarks.html:716-723`の以下のブロックの直後に:

```css
  .group-section-empty {
    font-family: var(--font-mono);
    font-size: 0.8rem;
    color: var(--ink-soft);
    padding: 18px;
    text-align: center;
    border: 1px dashed var(--sage);
    border-radius: 6px;
  }
```

次のCSSを追加する:

```css
  .search-no-results {
    font-family: var(--font-mono);
    font-size: 0.85rem;
    color: var(--ink-soft);
    padding: 40px 18px;
    text-align: center;
    border: 1px dashed var(--sage);
    border-radius: 6px;
  }
```

- [ ] **Step 2: `renderSections()`を書き換える**

`bookmarks.html:1650-1671`の以下のコードを:

```js
    var sectionKeys = state.group !== null
      ? [state.group === '' ? UNGROUPED : state.group]
      : groupNames.concat([UNGROUPED]);

    document.getElementById('result-count').textContent =
      filtered.length + ' 件表示中(全 ' + bookmarks.length + ' 件) — カードを他のグループ区画にドラッグすると移動できます';

    container.innerHTML = sectionKeys.map(function (key) {
      var items = sortItems(filtered.filter(function (b) { return (b.group || null) === key; }));
      var body = items.length
        ? '<div class="group-grid" data-group-grid="' + escapeHtml(key || '') + '">' +
            items.map(function (b) { return b.id === state.editingId ? renderEditCard(b) : renderCard(b); }).join('') +
          '</div>'
        : '<div class="group-section-empty" data-group-grid="' + escapeHtml(key || '') + '">該当するブックマークがありません。ここにドラッグして移動できます。</div>';

      return (
        '<section class="group-section" data-group="' + escapeHtml(key || '') + '">' +
          renderGroupSectionHeader(key, items.length) +
          body +
        '</section>'
      );
    }).join('');
```

次のコードに置き換える:

```js
    var sectionKeys = state.group !== null
      ? [state.group === '' ? UNGROUPED : state.group]
      : groupNames.concat([UNGROUPED]);
    if (state.search.trim()) {
      sectionKeys = sectionKeys.filter(function (key) {
        return filtered.some(function (b) { return (b.group || null) === key; });
      });
    }

    document.getElementById('result-count').textContent =
      filtered.length + ' 件表示中(全 ' + bookmarks.length + ' 件) — カードを他のグループ区画にドラッグすると移動できます';

    container.innerHTML = sectionKeys.length
      ? sectionKeys.map(function (key) {
          var items = sortItems(filtered.filter(function (b) { return (b.group || null) === key; }));
          var body = items.length
            ? '<div class="group-grid" data-group-grid="' + escapeHtml(key || '') + '">' +
                items.map(function (b) { return b.id === state.editingId ? renderEditCard(b) : renderCard(b); }).join('') +
              '</div>'
            : '<div class="group-section-empty" data-group-grid="' + escapeHtml(key || '') + '">該当するブックマークがありません。ここにドラッグして移動できます。</div>';

          return (
            '<section class="group-section" data-group="' + escapeHtml(key || '') + '">' +
              renderGroupSectionHeader(key, items.length) +
              body +
            '</section>'
          );
        }).join('')
      : '<div class="search-no-results">検索条件に一致するブックマークがありません。</div>';
```

- [ ] **Step 3: 変更が意図通りか確認する**

Run: `grep -n "search-no-results\|state.search.trim()" bookmarks.html`
Expected: CSSクラス定義(1件)、`renderSections()`内の新規filter条件(1件)、既存の他箇所での`state.search.trim()`利用(`getSearchTagFiltered()`等)がヒットし、構文エラーが無いこと。

- [ ] **Step 4: コミット**

```bash
git add bookmarks.html
git commit -m "$(cat <<'EOF'
feat: bookmarks.htmlの検索でヒット0件のグループを非表示にする

renderSections()のsectionKeys決定後に、検索語が非空のときだけ
filteredに1件もヒットしないグループキーを除外するフィルタを追加。
全体で0件のときはセクションの代わりに全体向けメッセージ
(.search-no-results)を表示する。
EOF
)"
```

---

### Task 2: `bookmarks-filesync.html` に同じ変更を適用する

**Files:**
- Modify: `bookmarks-filesync.html:1952-1973`(`renderSections()`の`sectionKeys`算出〜`container.innerHTML`代入。Task 1と同一内容)
- Modify: `bookmarks-filesync.html`の`.group-section-empty`定義直後(Task 1と同一CSSを追加)

**Interfaces:**
- Consumes: 既存の`getSearchTagFiltered()`(filesync側、変更しない)、`state.search`
- Produces: なし。Task 1と同一の`.search-no-results`定義・同一のフィルタロジックを、filesync側のコードとして独立に定義する(2ファイルは共通モジュールを持たないため重複するが、内容はTask 1と完全一致させること)。

- [ ] **Step 1: CSSを追加する**

`bookmarks-filesync.html`内で`.group-section-empty`のCSSブロック(Task 1のbookmarks.html:716-723と同一内容)を検索し、その直後にTask 1のStep 1と同一の`.search-no-results`ブロックを追加する。

- [ ] **Step 2: `renderSections()`を書き換える**

`bookmarks-filesync.html:1952-1973`の該当コード(Task 1のStep 2で示した「変更前」のbookmarks.html版と同一内容)を、Task 1のStep 2で示した「変更後」のコードと同一内容に置き換える。

- [ ] **Step 3: 変更が意図通りか確認する**

Run: `grep -n "search-no-results\|state.search.trim()" bookmarks-filesync.html`
Expected: Task 1のStep 3と同様、CSSクラス定義(1件)と新規filter条件(1件)がヒットする。

- [ ] **Step 4: コミット**

```bash
git add bookmarks-filesync.html
git commit -m "$(cat <<'EOF'
feat: bookmarks-filesync.htmlの検索でヒット0件のグループを非表示にする

bookmarks.htmlと同じ変更をfilesync版にも適用。
EOF
)"
```

---

### Task 3: ブラウザでの手動検証(RED/GREEN証拠の独立採取)

**Files:**
- なし(検証のみ。修正が必要になった場合はTask 1〜2の該当箇所に戻って直す)

**Interfaces:**
- Consumes: Task 1〜2で完成した両ファイル、およびTask 1のBASEコミット(変更前の`bookmarks.html`)
- Produces: RED/GREENのDOM評価ログ(スクリーンショット不要、`document.querySelectorAll`等の評価結果で可)

- [ ] **Step 1: RED証拠を採取する(変更前の挙動を再現)**

`git show <Task1のBASEコミット>:bookmarks.html` で変更前のファイル内容を一時ファイルに書き出し、ブラウザで開く。複数グループ(例: 「仕事」「趣味」)+未分類にそれぞれ1件以上のブックマークがある状態を作り、いずれか1グループにしかヒットしない検索語を入力する。

Expected(バグ再現): ヒットしないグループにも「該当するブックマークがありません。ここにドラッグして移動できます。」というプレースホルダ付きのセクション見出しが表示され続けることをDOM評価(`document.querySelectorAll('.group-section').length`が全グループ数のままであること等)で確認し、ログとして記録する。

- [ ] **Step 2: GREEN証拠を採取する(修正後の挙動)**

`bookmarks.html`(Task 1適用後の作業ツリー版)をブラウザで開き、Step 1と同じ検索語を入力する。

Expected: ヒットしたグループのセクションのみDOMに存在し(`document.querySelectorAll('.group-section').length`がヒットしたグループ数と一致)、ヒット0件のグループのセクションが存在しないことを確認する。

- [ ] **Step 3: グループ絞り込み中の検索で0件になるケースを確認する**

サイドバーのグループタブから特定グループに絞り込んだ状態(`state.group`が単一グループ)で、そのグループ内に存在しない検索語を入力する。

Expected: そのグループのセクションも非表示になり、`.search-no-results`要素が1つ表示され、「検索条件に一致するブックマークがありません。」というテキストが表示される。

- [ ] **Step 4: 全体で0件になるケースを確認する**

グループ絞り込みを解除した状態で、どのブックマークにもヒットしない検索語を入力する。

Expected: `.group-section`が0件、`.search-no-results`が1件表示される。

- [ ] **Step 5: 未分類のみヒットするケースを確認する**

未分類のブックマークのみがヒットする検索語を入力する。

Expected: 未分類セクションのみ表示され、他の名前付きグループのセクションは非表示になる。

- [ ] **Step 6: タグ併用時の挙動を確認する**

タグを1つ選択した状態で、そのタグを持つブックマークの一部にのみヒットする検索語を入力する。

Expected: タグとsearchの両方の条件を満たすブックマークを含むグループのみ表示される(タグ条件だけで絞り込んだ場合の挙動には影響しないことも合わせて確認: 検索語を消してタグのみにすると、ヒット0件のグループも見出し+プレースホルダ付きで表示に戻ることを確認する)。

- [ ] **Step 7: 検索クリアで元の表示に復帰することを確認する**

Step 2〜6のいずれかの状態から検索語をクリア(✕ボタンまたはEsc)する。

Expected: 全グループ+未分類のセクション見出しが(ヒット0件のものも含めて)従来通りすべて表示される回帰確認。

- [ ] **Step 8: `bookmarks-filesync.html`でStep 1〜7を繰り返す**

Chrome/Edgeで`bookmarks-filesync.html`を開き(ファイル読み込みまたは新規作成)、Step 1〜7と同じ確認を行う。

Expected: `bookmarks.html`と同じ挙動になる。

- [ ] **Step 9: RED/GREENログをreportファイルにまとめて報告する**

各Stepで確認したDOM評価結果(該当セクション数・`.search-no-results`の有無・テキスト内容)を簡潔にまとめ、Spec Compliance形式で報告する。

---
