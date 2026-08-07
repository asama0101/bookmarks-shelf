# URLドラッグ&ドロップによる追加

- 日付: 2026-08-07（2026-08-08 改訂: 最終レビュー指摘を受け、ドロップ時の挙動を「即時追加」から「URL事前入力済みの追加パネルを開く」に変更）
- 対象ファイル: `bookmarks.html`, `bookmarks-filesync.html`

## 背景

現状、ブックマークを追加するには「+追加」ボタンをクリックしてパネルを開き、フォームを操作する必要がある。ブラウザの別タブやアドレスバーからリンクをドラッグしている状態からでも、クリックや入力を挟まずに追加できれば手数が減る。

過去に「検索バーへのURL入力」「クリップボード読み取りボタン」という2つの新規UI経路を追加する設計を検討・実装したが、方向性が合わず破棄した経緯がある。今回はそれらとは異なるアプローチとして、既存の「+追加」ボタンをドロップターゲットとして流用する。

### 改訂の経緯

当初の設計は、ドロップと同時にタグ・グループなしの最小構成でブックマークを即座に保存する「即時追加」だった。実装完了後の最終レビューで、タグフィルターやグループフィルターを適用中のままドロップすると、追加自体は成功しているのに一覧には一切表示されず、ドロップが無視されたのか区別がつかない、という指摘を受けた。この指摘を受けてユーザーに確認したところ、「常にドロップでは即時追加せず、URL欄事前入力済みの追加パネルを開く」という挙動に変更する方針が決まった。フィルターの状態に関わらず常に同じ挙動になるため分かりやすく、ユーザーはパネル上でタイトル・タグ・グループを確認・追記してから保存できる。

## 要件

- ブラウザの別タブのリンクやアドレスバーのアイコンなどをドラッグし、既存の「+追加」ボタンにドロップすると、既存の追加パネルが開き、URL欄にそのURLが自動入力され、タイトル欄も既存の自動補完ロジックにより自動入力された状態になる。ブックマークはこの時点ではまだ保存されない（ユーザーが「保存」を押すか、他のフィールドを補足してから保存する）。
- URLとして認識できないものがドロップされた場合は何もしない（パネルを開かない。エラー表示も行わない）。
- 新規UI要素は追加しない（既存の「+追加」ボタンのみを使う）。
- 既存のカード並べ替え・グループ移動用のドラッグ&ドロップとは衝突しない。
- `bookmarks-filesync.html` では、ファイル未接続の間はこの機能も無効化する。

## 設計

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

`#toggle-add-btn` に `dragover` / `dragleave` / `drop` のリスナーを追加する。既存のクリックハンドラ（フォームのトグル開閉）はそのまま変更しない。追加フォーム関連の既存変数（`addPanel`, `newUrlInput`）を使うため、この3つのリスナーは追加フォームのJS（`addPanel`/`newUrlInput`等の宣言、および既存の「+追加」ボタンのクリックハンドラ）の直後に置く。

- `dragover`: ドラッグされているデータに `text/uri-list` または `text/plain` が含まれる場合のみ `preventDefault()` してドロップ対象として有効化し、視覚フィードバック用のクラスを付与する。カード並べ替え中（`draggedCardId` が設定されている間）は反応しない。
- `dragleave`: 視覚フィードバック用のクラスを除去する。
- `drop`: 常に `preventDefault()` してからURLを抽出する。`classifyUrl()` で有効なURLと判定できた場合のみ、追加パネルを開いてURL欄に値をセットし、`input` イベントを手動発火させて既存のタイトル自動補完ロジック（`newUrlInput` の `input` リスナー）を働かせ、URL欄にフォーカスする。無効なURLの場合は何もしない。

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
  if (!classifyUrl(url)) return; // URLとして認識できない場合は何もしない
  addPanel.hidden = false;
  newUrlInput.value = url;
  newUrlInput.dispatchEvent(new Event('input')); // 既存のタイトル自動補完(newUrlInputのinputリスナー)を発火させる
  newUrlInput.focus();
});
```

`quickAddBookmark()` のような「即座に保存する」専用関数は不要になった（既存の追加フォーム・submitハンドラがそのまま使われるため）。

### 既存のドラッグ&ドロップとの非衝突

- `#toggle-add-btn` はヘッダー内にあり、`.group-sections`（カード・グループ区画）とはDOM上重ならないため、既存の `bindDragReorder` / `bindGroupSectionDragEvents` のドロップハンドラとは物理的に衝突しない。
- カードの並べ替え中は `dragover` で `draggedCardId`（カードのドラッグ中に設定される既存のモジュール変数）をチェックして早期リターンすることで、内部の並べ替えドラッグがこのボタン上を通過しても誤ってボタンがハイライトされないようにする。
- カードの `id` 文字列（`makeId()` が生成する UUID や `id-<timestamp>-<random>` 形式）は `classifyUrl()` の許可パターンに一致しないため、万一カードがこのボタンにドロップされても `drop` ハンドラは無効なURLとして何もしない（実害はない）。

### 視覚フィードバック（CSS）

```css
#toggle-add-btn.add-btn-drop-target {
  box-shadow: 0 0 0 3px var(--amber-ink);
}
```

### HTML: ヒントテキスト

`#toggle-add-btn` に `title` 属性を追加する（ドロップすると即保存ではなくパネルが開く挙動に合わせた文言にする）:

```html
<button class="btn btn-amber" id="toggle-add-btn" type="button" title="URLをドラッグ&amp;ドロップして追加パネルを開く">+ 追加</button>
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
  if (!classifyUrl(url)) return;
  addPanel.hidden = false;
  newUrlInput.value = url;
  newUrlInput.dispatchEvent(new Event('input'));
  newUrlInput.focus();
});
```

（`dragleave` は `fileHandle` の有無に関わらずクラスを外すだけなので、ガード不要で `bookmarks.html` と同一内容。）

## 変更しないもの

- 既存の「+追加」フォーム自体のフィールド構成・submitハンドラ・クリックハンドラ（トグル開閉）のロジック
- カード並べ替え・グループ移動用の既存ドラッグ&ドロップのロジック
- キーボードショートカット
- タグ・グループフィルターの自動解除は行わない（ドロップ後は必ずパネルが開くため、フィルター中でも「追加が成功したのに一覧に見えない」という当初の問題は発生しない）

## 既知の制約（今回の対象外）

`.group-section` 以外の領域（ヘッダーの空白部分など）に外部リンクをドロップした場合、ブラウザの既定動作でそのURLへページ遷移してしまう可能性がある。これは今回の変更が原因ではなく既存の挙動であり、ページ全体を対象にした対策は今回のスコープ外とする。`bookmarks-filesync.html` ではこの遷移が発生すると `fileHandle` も失われるため、localStorage版より影響が大きい点に留意する（再接続フローで復旧は可能）。

## 検証方法

自動テストは存在しないプロジェクトのため、ブラウザでの手動確認のみ（`bookmarks.html` と `bookmarks-filesync.html` の両方で実施）:

- 別タブのリンク、またはブラウザのアドレスバーのURLを「+追加」ボタンへドラッグ&ドロップし、ドラッグオーバー中にボタンへ視覚フィードバック（枠線）が出ることを確認。ドロップすると追加パネルが開き、URL欄にそのURLが、タイトル欄に自動生成されたタイトルが入っていることを確認する。この時点ではまだブックマークは保存されておらず、「保存」を押すと一覧に追加されることを確認する。
- タグフィルターまたはグループフィルターを適用した状態で上記を行い、同様にパネルが開くことを確認する（フィルター中でも挙動が変わらないことの確認）。
- URLでないもの（選択したテキスト、画像など）をボタンにドロップし、何も起きない（パネルが開かない）ことを確認する。
- 既存のカードをドラッグして並べ替えている最中、「+追加」ボタンの上を通過してもボタンがハイライトされないことを確認する。
- 既存のカード並べ替え・グループ間移動・グループ管理モーダルの並べ替えが、これまで通り動作することを確認する（回帰確認）。
- 既存の「+追加」ボタンのクリックによる、通常のフル入力フォームでの追加が、これまで通り動作することを確認する（回帰確認）。
- `bookmarks-filesync.html` では、ファイル未接続の間に同様のドラッグ&ドロップを試み、何も起きないことを確認してから、接続後に上記が動作することを確認する。
