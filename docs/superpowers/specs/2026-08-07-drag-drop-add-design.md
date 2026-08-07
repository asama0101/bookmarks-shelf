# URLドラッグ&ドロップによる追加

- 日付: 2026-08-07
- 対象ファイル: `bookmarks.html`, `bookmarks-filesync.html`

## 背景

現状、ブックマークを追加するには「+追加」ボタンをクリックしてパネルを開き、フォームを操作する必要がある。ブラウザの別タブやアドレスバーからリンクをドラッグしている状態からでも、クリックや入力を挟まずに追加できれば手数が減る。

過去に「検索バーへのURL入力」「クリップボード読み取りボタン」という2つの新規UI経路を追加する設計を検討・実装したが、方向性が合わず破棄した経緯がある。今回はそれらとは異なるアプローチとして、既存の「+追加」ボタンをドロップターゲットとして流用する。

## 要件

- ブラウザの別タブのリンクやアドレスバーのアイコンなどをドラッグし、既存の「+追加」ボタンにドロップすると、そのURLが最小構成（タグ・グループ・ログイン情報・メモなし、タイトル自動生成）の新規ブックマークとして即座に追加される。
- URLとして認識できないものがドロップされた場合は何もしない（エラー表示・フォールバックパネル表示は行わない）。
- 新規UI要素は追加しない（既存の「+追加」ボタンのみを使う）。
- 既存のカード並べ替え・グループ移動用のドラッグ&ドロップとは衝突しない。
- `bookmarks-filesync.html` では、ファイル未接続の間はこの機能も無効化する。

## 設計

### 共通処理: `quickAddBookmark(url)`

新規の独立した関数として追加する（既存の追加フォームsubmitハンドラは変更しない）。

```js
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

### データ抽出: `extractUrlFromDrop(dataTransfer)`

ドロップされた `DataTransfer` からURL文字列を取り出す。ブラウザがリンクをドラッグする際に設定する `text/uri-list`（複数行になりうる。`#` で始まる行はコメントなので除外し、最初の有効な行を採用）を優先し、無ければ `text/plain` にフォールバックする。

```js
function extractUrlFromDrop(dataTransfer) {
  var uriList = dataTransfer.getData('text/uri-list');
  if (uriList) {
    var lines = uriList.split(/\r?\n/).map(function (l) { return l.trim(); })
      .filter(function (l) { return l && l.charAt(0) !== '#'; });
    if (lines.length) return lines[0];
  }
  return dataTransfer.getData('text/plain').trim();
}
```

### 「+追加」ボタンへのドロップ処理

`#toggle-add-btn` に `dragover` / `dragleave` / `drop` のリスナーを追加する。既存のクリックハンドラ（フォームを開く）はそのまま変更しない。

- `dragover`: ドラッグされているデータに `text/uri-list` または `text/plain` が含まれる場合のみ `preventDefault()` してドロップ対象として有効化し、視覚フィードバック用のクラスを付与する。カード並べ替え中（`draggedCardId` が設定されている間）は反応しない。
- `dragleave`: 視覚フィードバック用のクラスを除去する。
- `drop`: 常に `preventDefault()` してからURLを抽出し、`quickAddBookmark()` に渡す。無効なURLの場合は `quickAddBookmark()` が何もせず `false` を返すのでそのままで良い（追加のエラー処理は不要）。

```js
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

### 既存のドラッグ&ドロップとの非衝突

- `#toggle-add-btn` はヘッダー内にあり、`.group-sections`（カード・グループ区画）とはDOM上重ならないため、既存の `bindDragReorder` / `bindGroupSectionDragEvents` のドロップハンドラとは物理的に衝突しない。
- カードの並べ替え中は `dragover` で `draggedCardId`（カードのドラッグ中に設定される既存のモジュール変数）をチェックして早期リターンすることで、内部の並べ替えドラッグがこのボタン上を通過しても誤ってボタンがハイライトされないようにする。
- カードの `id` 文字列（`makeId()` が生成する UUID や `id-<timestamp>-<random>` 形式）は `classifyUrl()` の許可パターンに一致しないため、万一カードがこのボタンにドロップされても `quickAddBookmark()` は無効なURLとして何もしない（実害はない）。

### 視覚フィードバック（CSS）

```css
#toggle-add-btn.add-btn-drop-target {
  box-shadow: 0 0 0 3px var(--amber-ink);
}
```

### HTML: ヒントテキスト

`#toggle-add-btn` に `title` 属性を追加する:

```html
<button class="btn btn-amber" id="toggle-add-btn" type="button" title="URLをドラッグ&amp;ドロップして追加">+ 追加</button>
```

（`bookmarks-filesync.html` では既存の `disabled` 属性はそのまま残す。)

### `bookmarks-filesync.html` 固有の対応

`#toggle-add-btn` は接続前 `disabled` になっているが、`disabled` 属性がドラッグ&ドロップイベントを確実に抑止するとは限らないため、念のため `dragover` と `drop` の両方に `fileHandle` の未接続ガードを明示的に入れる（他の新機能で採用してきた多重防御パターンと同じ）。

```js
document.getElementById('toggle-add-btn').addEventListener('dragover', function (e) {
  if (!fileHandle) return;
  if (draggedCardId) return;
  var types = e.dataTransfer.types;
  if (types.indexOf('text/uri-list') === -1 && types.indexOf('text/plain') === -1) return;
  e.preventDefault();
  this.classList.add('add-btn-drop-target');
});

document.getElementById('toggle-add-btn').addEventListener('drop', function (e) {
  e.preventDefault();
  this.classList.remove('add-btn-drop-target');
  if (!fileHandle) return;
  var url = extractUrlFromDrop(e.dataTransfer);
  quickAddBookmark(url);
});
```

（`dragleave` は `fileHandle` の有無に関わらずクラスを外すだけなので、ガード不要でTask 1と同一内容。）

## 変更しないもの

- 既存の「+追加」フォーム自体のフィールド構成・submitハンドラ・クリックハンドラ（トグル開閉）のロジック
- カード並べ替え・グループ移動用の既存ドラッグ&ドロップのロジック
- キーボードショートカット

## 既知の制約（今回の対象外）

`.group-section` 以外の領域（ヘッダーの空白部分など）に外部リンクをドロップした場合、ブラウザの既定動作でそのURLへページ遷移してしまう可能性がある。これは今回の変更が原因ではなく既存の挙動であり、ページ全体を対象にした対策は今回のスコープ外とする。

## 検証方法

自動テストは存在しないプロジェクトのため、ブラウザでの手動確認のみ（`bookmarks.html` と `bookmarks-filesync.html` の両方で実施）:

- 別タブのリンク、またはブラウザのアドレスバーのURLを「+追加」ボタンへドラッグ&ドロップし、ドラッグオーバー中にボタンへ視覚フィードバック（枠線）が出ることを確認。ドロップすると即座に新しいカード（未分類・タグ無し・タイトル自動生成）が追加されることを確認。
- URLでないもの（選択したテキスト、画像など）をボタンにドロップし、何も起きないことを確認。
- 既存のカードをドラッグして並べ替えている最中、「+追加」ボタンの上を通過してもボタンがハイライトされないことを確認。
- 既存のカード並べ替え・グループ間移動・グループ管理モーダルの並べ替えが、これまで通り動作することを確認（回帰確認）。
- 既存の「+追加」ボタンのクリックによるフル入力フォームでの追加が、これまで通り動作することを確認（回帰確認）。
- `bookmarks-filesync.html` では、ファイル未接続の間に同様のドラッグ&ドロップを試み、何も起きないことを確認してから、接続後に上記が動作することを確認。
