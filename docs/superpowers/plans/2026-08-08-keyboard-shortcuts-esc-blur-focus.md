# Escで閉じるものが無い場合にフォーカスを外す Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `closeTopmostLayer()` が実行された際、閉じる層（ヘルプパネル・グループ管理モーダル・タグ管理モーダル・編集/追加フォーム・選択モード）が何も無い場合に、現在フォーカスしている要素からフォーカスを外し、初期状態に戻す。

**Architecture:** `closeTopmostLayer()` の末尾、選択モードを閉じる分岐の直後に新しい分岐を1つ追加する。`bookmarks.html` に実装した後、`bookmarks-filesync.html` に同一内容を移植する。

**Tech Stack:** Vanilla JS (ES5-style, IIFE), 単一HTMLファイル、ビルドなし。

## Global Constraints

- 対象ファイルは `bookmarks.html` と `bookmarks-filesync.html` の両方。挙動は完全に同一にする。
- 選択モードを閉じる分岐（`state.selectMode` が true の場合）は、これまで通りその1段階だけを実行して終了する（同じ Esc の1回で続けてフォーカスも外す、という二重処理はしない）。
- `document.activeElement` が `document.body`（＝そもそも何もフォーカスされていない）場合は何もしない。
- 検索欄のEscクリア処理（`clearSearch()` とその優先判定条件）は一切変更しない。検索欄にフォーカスがあり検索文字列が入力されている間は、`closeTopmostLayer()` 自体が呼ばれないため今回の変更の対象にならない。
- `closeTopmostLayer()` 内の他の既存分岐（ヘルプパネル・グループ管理モーダル・タグ管理モーダル・編集/追加フォームを閉じる処理）は一切変更しない。

---

### Task 1: `bookmarks.html` — フォーカス解除分岐の実装

**Files:**
- Modify: `bookmarks.html:2547-2552`（`closeTopmostLayer()` の末尾）

**Interfaces:**
- Consumes: 既存の `closeTopmostLayer()` 関数本体、`state.selectMode`, `state.selectedIds`, `render()`（すべて同ファイル内で本タスクの変更箇所より前に既に宣言・定義済み）
- Produces: `closeTopmostLayer()` の末尾に追加される新しい分岐（新規関数・新規識別子は無い）。Task 2 でファイル間の対応関係を確認する対象になる。

- [ ] **Step 1: `closeTopmostLayer()` の末尾を書き換える**

`bookmarks.html:2547-2552` の現在の内容:

```js
    if (state.selectMode) {
      state.selectMode = false;
      state.selectedIds.clear();
      render();
    }
  }
```

これを以下に置き換える:

```js
    if (state.selectMode) {
      state.selectMode = false;
      state.selectedIds.clear();
      render();
      return;
    }

    // 閉じる層が何も無い場合、フォーカスを完全に外して初期状態に戻す
    if (document.activeElement && document.activeElement !== document.body && typeof document.activeElement.blur === 'function') {
      document.activeElement.blur();
    }
  }
```

- [ ] **Step 2: ブラウザで動作確認する**

`xdg-open bookmarks.html` で開き、以下を確認する:

1. カード一覧のいずれかのカードにフォーカスした状態（枠線が表示されている状態）で Esc を押すと、フォーカスの枠線が消え、何もフォーカスされていない状態になる。
2. `t`/`g` でタグ/グループナビにフォーカスした状態で Esc を押すと、同様にフォーカスが外れる。
3. 検索欄にフォーカスしているが何も入力していない状態で Esc を押すと、検索欄からフォーカスが外れる。
4. 検索欄にフォーカスし、何か入力した状態で Esc を押すと、これまで通り検索がクリアされ検索欄にフォーカスが残る（今回の変更の対象外であることの回帰確認）。
5. ヘルプパネル・グループ管理モーダル・タグ管理モーダル・編集/追加フォーム・選択モードが開いている状態でそれぞれ Esc を押すと、これまで通りその層だけが1段階閉じ、フォーカスは外れない（例えば選択モードを閉じた直後にフォーカスが消えたりしない）ことを確認する。
6. 何もフォーカスされていない状態（ページ読み込み直後など）で Esc を押しても、エラーが発生しないことを確認する（開発者ツールのコンソールにエラーが出ないことを確認する）。

- [ ] **Step 3: コミットする**

```bash
git add bookmarks.html
git commit -m "feat: blur focus on Esc when no layer is open in bookmarks.html"
```

---

### Task 2: `bookmarks-filesync.html` — 同内容の移植と対称性確認

**Files:**
- Modify: `bookmarks-filesync.html:2851-2856`（`closeTopmostLayer()` の末尾、Task 1 の `bookmarks.html:2547-2552` に相当）

**Interfaces:**
- Consumes: Task 1 で確定した `closeTopmostLayer()` の変更内容（コードは同一、行番号のみ異なる）。`bookmarks-filesync.html` 側の既存 `state.selectMode`, `state.selectedIds`, `render()` を使う。
- Produces: `bookmarks-filesync.html` 内の `closeTopmostLayer()` 末尾（`bookmarks.html` と同一内容）。

- [ ] **Step 1: 現在の行番号を確認する**

`bookmarks.html` の変更中に行番号がずれている可能性があるため、着手前に以下を実行して実際の行番号を確認する:

```bash
grep -n "state.selectMode) {" bookmarks-filesync.html
```

- [ ] **Step 2: `closeTopmostLayer()` の末尾を書き換える**

現在の内容（`bookmarks.html` の変更前と同一パターン）:

```js
    if (state.selectMode) {
      state.selectMode = false;
      state.selectedIds.clear();
      render();
    }
  }
```

これを以下に置き換える:

```js
    if (state.selectMode) {
      state.selectMode = false;
      state.selectedIds.clear();
      render();
      return;
    }

    // 閉じる層が何も無い場合、フォーカスを完全に外して初期状態に戻す
    if (document.activeElement && document.activeElement !== document.body && typeof document.activeElement.blur === 'function') {
      document.activeElement.blur();
    }
  }
```

- [ ] **Step 3: 対称性を grep で検証する**

`bookmarks.html` と `bookmarks-filesync.html` の該当コードが一致することを確認する:

```bash
grep -c "document.activeElement.blur()" bookmarks.html
grep -c "document.activeElement.blur()" bookmarks-filesync.html
```

Expected: 両ファイルとも `1` が出力される。

- [ ] **Step 4: ブラウザで動作確認する**

`xdg-open bookmarks-filesync.html` で開き（Chrome/Edge必須）、ファイルを接続した上で Task 1 Step 2 と同じ6項目を確認する。

- [ ] **Step 5: コミットする**

```bash
git add bookmarks-filesync.html
git commit -m "feat: blur focus on Esc when no layer is open in bookmarks-filesync.html"
```
