# 矢印キーのグループ間移動対応

- 日付: 2026-08-09
- 対象ファイル: `bookmarks.html`, `bookmarks-filesync.html`

## 背景

現状、カードにフォーカスしている状態での矢印キー(↑↓←→)は、すべて `moveCardFocus(card, direction)` という単一のロジックで処理されている。`getNavigableCards()` が `#group-sections .card[tabindex]` を**グループ区画をまたいだ1本のDOM順リスト**として扱うため、↑↓←→のどれを押しても「一覧全体の中で次/前のカードへ移動する」という同じ挙動になり、方向とグループの関係が一切ない。

グループ区画をまたいで移動したいとき、現状は同じ方向キーを区画の境目を意識せず連打し続けるしかなく、直感的でない。↑↓を「グループ間の移動」に、←→を「グループ内でのカード移動」に役割分担させたい。

## 要件

- 対象はカードグリッド(`#group-sections`)にフォーカスがある場面のみ。サイドバーのタグ/グループnav、グループ管理・タグ管理モーダル、フォーム内の矢印キー挙動は変更しない。
- **↑/↓**: グループ間移動にする。
  - 移動先は現在フォーカス中のカードが属する区画の、表示順で1つ次/前の`.group-section`の**先頭カード**(区画内でDOM順最初のカード)。
  - 区画の順序は現在の表示順(`groupOrder`順、未分類は常に最後。`state.group`でのフィルタ中はその1区画のみ)にそのまま従う。追加のソートロジックは持たない。
  - 表示中カードが0件の区画(タグ絞り込み等で空になった区画)はスキップし、その先/前にカードのある区画へ移動する。
  - 最初の区画でさらに↑、最後の区画でさらに↓を押すと、表示順で反対側の端の区画へ循環する。
  - カードを持つ区画が実質1つしかない場合(フィルタで1グループのみ表示中など)は何も起きない。
- **←/→**: 現在の区画内でのカード前後移動にする。他区画のカードには移動しない。区画の端(最初/最後のカード)でさらに同方向を押しても何も起きない(循環しない)。
- ヘルプパネル(`#help-panel`)の矢印キー説明を新しい役割分担に合わせて更新する。

## 設計

### 新しいヘルパー関数(既存の `moveCardFocus` を置き換え)

`bookmarks.html:2680-2686` (filesync: `2982-2988`) の `moveCardFocus` を削除し、代わりに2つの関数を追加する。

```js
// 同じグループ区画内のカードのみを対象に、DOM順で前後移動する(区画をまたがない。端では何も起きない)。
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

`getNavigableCards()`・`applyRovingTabindex()`・`deleteCardWithFocusHandoff()`・`handleGlobalShortcut()` 内の未フォーカス時の「一覧へ入る」処理は、いずれも一覧全体を対象とする既存の挙動のままで変更不要(このspecが対象とするのは、既にカードにフォーカスしている状態での方向キーの意味だけ)。

### `handleCardShortcut()` の変更

`bookmarks.html:2588-2598` (filesync: `2890-2900`) の `switch` 文を次のように変更する。

変更前:
```js
      case 'ArrowDown':
      case 'ArrowRight':
        e.preventDefault();
        moveCardFocus(card, 1);
        return;
      case 'ArrowUp':
      case 'ArrowLeft':
        e.preventDefault();
        moveCardFocus(card, -1);
        return;
```

変更後:
```js
      case 'ArrowRight':
        e.preventDefault();
        moveCardFocusWithinGroup(card, 1);
        return;
      case 'ArrowLeft':
        e.preventDefault();
        moveCardFocusWithinGroup(card, -1);
        return;
      case 'ArrowDown':
        e.preventDefault();
        moveCardFocusToGroup(card, 1);
        return;
      case 'ArrowUp':
        e.preventDefault();
        moveCardFocusToGroup(card, -1);
        return;
```

### ヘルプパネルの変更

`bookmarks.html:1074` (filesync: `1174` 付近)の矢印キーの行を分割する。

変更前:
```html
    <dt><kbd>&#8593;</kbd> <kbd>&#8595;</kbd> <kbd>&#8592;</kbd> <kbd>&#8594;</kbd></dt><dd>カード間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
```

変更後:
```html
    <dt><kbd>&#8593;</kbd> <kbd>&#8595;</kbd></dt><dd>グループ間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
    <dt><kbd>&#8592;</kbd> <kbd>&#8594;</kbd></dt><dd>同じグループ内でカード間のフォーカス移動(未フォーカス時は一覧へ入る)</dd>
```

## 変更しないもの

- 未フォーカス時に矢印キーで一覧へ入る挙動(`handleGlobalShortcut()` のArrowキー分岐、`bookmarks.html:2556-2571`)
- サイドバーのタグ/グループnav内での矢印キー移動(`moveNavFocus()`)
- グループ管理・タグ管理モーダル表示中の矢印キー無効化
- フォーム内(`isTextInputElement`)での矢印キー処理
- `Home`/`End`キーの挙動
- `getNavigableCards()`・`applyRovingTabindex()`・`deleteCardWithFocusHandoff()` の実装

## 検証方法

自動テストは存在しないプロジェクトのため、ブラウザでの手動確認のみ(`bookmarks.html` と `bookmarks-filesync.html` の両方で実施)。

- 複数グループ+未分類を含むデータで、あるグループのカードにフォーカスした状態から↓を押すと、次の区画(表示順)の先頭カードにフォーカスが移ることを確認する。↑では前の区画の先頭カードに移ることを確認する。
- 最後の区画で↓を押すと最初の区画へ、最初の区画で↑を押すと最後の区画へ、それぞれ循環することを確認する。
- タグ絞り込みである区画の表示カードが0件になっている状態で↑↓を押すと、その区画が飛ばされることを確認する。
- グループフィルタ(`state.group`)で1区画のみ表示中に↑↓を押しても何も起きないことを確認する。
- 同じ区画内で←→を押すと、区画内のカードを前後に移動できることを確認する。区画内の最初/最後のカードでさらに同方向を押しても何も起きない(他区画へ移動しない)ことを確認する。
- `?`キーでヘルプパネルを開き、矢印キーの説明が新しい役割分担どおりに2行に分かれて表示されることを確認する。
- 未フォーカス状態から↑↓←→いずれを押しても、これまで通り一覧の起点カードへフォーカスが入ることを確認する(回帰確認)。
- `Home`/`End`・`c`/`p`/`Delete`/`e`/`Enter`など、他のショートカットがこれまで通り動作することを確認する(回帰確認)。
