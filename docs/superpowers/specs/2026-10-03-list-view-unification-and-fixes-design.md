# リスト表示一本化・お気に入り機能拡張・UX改善 Design

**ステータス:** ユーザー承認待ち(brainstorming grill完了、design-audit未実施)

**対象ブランチ:** `feature/ui-layout-density`(継続)。直前のタスクで3段階密度モード(ゆったり/コンパクト/リスト)+お気に入り並び替え機能を実装し、ユーザー受け入れ確認を待っていた状態だったが、本Specの内容(特にR1)がその密度モードの大半を削除する決定のため、mainへの未マージを活かして同ブランチ上で継続する。

**対象ファイル:** `bookmarks.html` / `bookmarks-filesync.html` の両方。本Specの全要件はR1を除きデータモデルに影響しない表示・操作ロジックの変更であり、`favoriteOrder`等の既存フィールドに変更は無い。

## 背景

ユーザーが実際に密度モード機能を試した結果、3モードは不要で「リストだけでいい」という結論に至った。加えて、リスト運用を前提とした上で以下6点の独立した改善要求が出た: リスト行でのドラッグ操作性、お気に入り行の機能不足、検索対象の不足、フィルタ選択時のスクロール位置、リスト行のタグ非表示、キーボードショートカットの過多。

grillの結果、以下が確定済み:
- 密度トグルは削除し、リスト表示に一本化する(ゆったり/コンパクトのコード・UIを削除)。
- お気に入りもリスト表示に一本化する。
- ショートカットは「検索フォーカス(`/`)」「矢印キー/Home/Endのナビ移動」「カード操作(Enter開く/`e`編集/Delete削除)」「Escapeチェーン」を残し、「`n`新規追加」「`t`/`g`タブ切替」「`?`ヘルプ」「`x`フィルタ全解除」を削除する(対応するボタンのクリックで操作可能なため)。

## 現状コードの実測(grill中にExploreで確認済み、file:lineはbookmarks.html基準、filesyncは概ね+250〜300行オフセット)

- 密度トグル関連の全参照箇所: CSS(152)、マークアップ(1167)、`DENSITY_MODE_KEY`(1335)、`loadDensityMode()`(1341)、`state.densityMode`初期化(1362)、`render()`内のアクティブボタン同期(1709-1711)、`renderFavoriteCardComfy`参照(1974-1975)、`group-grid--compact`/`--list`分岐(1992, 2051)、`renderCardComfy`(2559)、`renderCardCompact`(2621)、`renderCard()`ディスパッチ(2713-2715)、トグルのclickリスナー(2866-2870)。filesyncは同構造で CSS(161)、マークアップ(1247)、`DENSITY_MODE_KEY`(1435)、`loadDensityMode()`(1439)、初期化(1454)、`render()`同期(1988-1989)、`renderFavoriteCardComfy`参照(2261-2262)、grid分岐(2279, 2327)、`renderCardComfy`(2824)、`renderCardCompact`(2886)、ディスパッチ(2978-2980)、clickリスナー(2160-2164)。
- `renderCardList()`(2670-2710、filesync 2935-2975): `<article class="card card--list" data-card-id="...">` を出力。`.card-stub`等の掴み手要素は無い。
- ドラッグ機構: `bindDragEvents(card, b)`(2482-2491、filesync 2747-2756)が`card`要素自体に`draggable="true"`を設定し`bindDragReorder`(2289-2329)を紐付ける。ゲート条件は`!state.selectMode && state.sortMode !== 'frequency'`(2140、filesyncは`!state.selectMode`のみ、2405)で、`densityMode`には依存していない。`bindGroupSectionDragEvents(container)`(2151-2171、filesync 2416-2436)も同様に密度無依存。結論: リスト行の並び替え・グループ間移動は**既に機能する設計**であり、欠けているのは掴みやすい専用領域(視覚的アフォーダンス)のみ。
- `getSearchTagFiltered()`(1650-1661、filesync 1933-1944)の検索対象文字列(1656/1939): `(b.title + ' ' + b.url + ' ' + (b.tags || []).join(' ')).toLowerCase()` — `memo`/`loginId`/`domain`/`group`名は対象外。
- タグナビclick(1797-1803)・グループナビclick(1823-1829)・`switchSidebarTab()`(1833-1839、filesync概ね2079-2085/2105/2112)いずれにもスクロール処理は存在しない(`scrollIntoView`/`scrollTo`/`scrollTop`は両ファイルに1件も無い)。
- `renderFavoriteCardComfy()`(1936-1956、filesync 2223-2243)・`renderFavoriteCardList()`(1958-1971、filesync 2245-2258)はいずれも`data-action="unpin"`のみを出力する。`copy-login-id`/`copy-login-password`/`edit`は無い。
- ショートカット定義: `handleGlobalShortcut()`(3110-3168、filesync 3361-3419)、`handleCardShortcut()`(3171-3209、filesync 3422-3460)、トップレベル`keydown`リスナー(3211+、filesync 3462+)。Escapeチェーンの実体は`closeTopmostLayer()`(3046-3099、filesync 3297-3350)。

## 機能要件

### R1: 密度モード廃止・リスト表示への一本化

- `renderCardComfy()`・`renderCardCompact()`・`renderFavoriteCardComfy()`を削除する。
- `renderCard()`・`renderFavoriteCard()`はディスパッチャをやめる。現在の`renderCardList()`・`renderFavoriteCardList()`の関数本体を、それぞれ`renderCard()`・`renderFavoriteCard()`という元の(密度モード導入前の)名前へ統合し直す。`renderCardList`/`renderFavoriteCardList`という名前自体も削除する(唯一のモードに「List」という限定語を残す意味が無いため)。呼び出し元(`renderCard(b)`/`renderFavoriteCard(b)`を呼んでいる箇所)は名前を変える必要が無い。
- `#density-toggle`のマークアップ・CSS、`state.densityMode`、`DENSITY_MODE_KEY`とその読み書き(`loadDensityMode()`含む)、トグルのclickリスナーを削除する。
- `group-grid--compact`/`group-grid--list`という「モード別の修飾クラス」という構造自体をやめ、リスト用のレイアウトを`.group-grid`・`.card`の基本セルクタに直接統合する(今や分岐の無い唯一のモードに対して「modifier」クラスを維持する意味が無いため)。
- 既存ユーザーのlocalStorageに残る`DENSITY_MODE_KEY`の値は読み書きしないことで実質無害化する(明示的な削除処理は不要、YAGNI)。
- 密度モード専用だった`.ind-btn`等のCSS/マークアップのうち、コンパクトモード専用だったメモの丸アイコン表現(`renderCardCompact()`内でのみ使われていたもの)は削除する。ログインID/パスワードの`.ind-btn.filled`/`.ind-btn.empty`(コピー可否を色・斜線で示す表現)はリスト表示でも使われているため維持する。
- 以下のCSSブロックも削除対象(`.card--list`/`.group-grid--list`は基本セレクタへ統合する側、それ以外は死蔵化するため削除する)。filesync側は同構造の対応箇所を同様に処理する:
  - `bookmarks.html:1008-1014`(filesync `1085-1091`): `.card--compact ...` — 削除。
  - `bookmarks.html:1016`(filesync `1093`): `.group-grid--compact` — 削除。
  - `bookmarks.html:1059-1078`(filesync 同等箇所): `.card--list ...` — ルール内容を`.card`の基本セレクタへ統合し、`--list`修飾クラスは削除する。
  - `bookmarks.html:1096`(filesync `1173`): `.group-grid--list` — ルール内容を`.group-grid`の基本セレクタへ統合し、`--list`修飾クラスは削除する。
  - `bookmarks.html:1098-1118`: `.card-favorite .card-stub` と `.card-favorite--list ...` — R3(お気に入りのリスト一本化)に伴い、同じ方針(`--list`の内容を基本セレクタへ統合、`.card-favorite .card-stub`は後述の通り維持)で処理する。
- `.card-stub`の基底CSS(`bookmarks.html:782-811`)は密度モードとは無関係に`renderEditCard()`(`bookmarks.html:2721`)が常時使用している既存クラスであり、削除対象外。R2のリスト行グリップはこのクラスを再利用する。
- 実装後の残存確認用grepは`densityMode|DENSITY_MODE_KEY|card--compact|group-grid--compact|card--list|group-grid--list|card-favorite--list|renderCardComfy|renderCardCompact|renderFavoriteCardComfy`に拡張する(「リスク・懸念点」節のgrepパターンもこれに合わせて更新する)。

### R2: リスト行へのドラッグ用グリップ追加

- 既存の`.card-stub`クラス(かつてゆったり/コンパクトモードの掴み手だった要素)を、リスト行の掴み手として再利用・再スタイリングする。R1でゆったり/コンパクトが削除されるため、`.card-stub`は今後リスト行専用の掴み手として存在する。
- ドラッグ機構自体(`bindDragEvents`/`bindDragReorder`/`bindGroupSectionDragEvents`)はR1実測の通り密度に依存せず既に機能しているため、JS側の変更は不要。見た目(掴み手の表示)のみの変更。

### R3: お気に入りのリスト表示一本化+コピー/編集機能追加

- `renderFavoriteCardComfy()`を削除し、`renderFavoriteCardList()`のみを残す(R1と同時に実施)。`renderFavoriteCard()`のディスパッチャ分岐も解消する。
- `renderFavoriteCardList()`に以下の`data-action`ボタンを追加する:
  - `copy-login-id` / `copy-login-password`: 通常カードのリスト行と同じ`.ind-btn`(filled/empty)表現を使い、クリックで既存の`copyToClipboard()`を呼ぶ。
  - `edit`: 通常カードのeditハンドラ(`bookmarks.html:2093`、`state.editingId = b.id; render();`というインライン処理で、専用関数化されていない)と同じ2行を、お気に入りセクションのクリックハンドラ内に複製する。
- お気に入りセクションは`data-card-id`ではなく`data-fav-id`を使う独立したDOM・イベントバインド(CLAUDE.md記載の既存制約)であるため、これら新規アクションは**お気に入りセクション自身のクリックハンドラ内に追加実装**し、通常カードの`data-card-id`ベースの汎用クリックループを再利用しない。コピー(`copyToClipboard()`呼び出し)は共通関数を呼ぶだけで複製しないが、edit(上記`state.editingId`設定)は通常カード側がインライン処理のため複製が必要。
- `renderFavoritesSection()`は`state.editingId`を参照しないため、お気に入り行で`edit`を押しても**お気に入りセクション自身の表示は変化しない**。編集フォームは、そのブックマークが属する通常のグループセクション側に開く(既存の`render()`機構どおり)。ユーザーは編集フォームを見るためにそのグループセクションまでスクロールする必要がある。この挙動は仕様として明記し、お気に入りセクション側で何らかの視覚的フィードバックを追加実装することはしない(YAGNI、要求外)。
- 既存のドラッグ並び替え機構(`favoriteOrder`・`bindFavoriteDragEvents`・`moveFavorite`・`.fav-grip`)は本要件の対象外、無変更。

### R4: 検索対象にメモを追加

- `getSearchTagFiltered()`の検索対象文字列(1656/1939)に`b.memo`を追加する: `(b.title + ' ' + b.url + ' ' + (b.tags || []).join(' ') + ' ' + (b.memo || '')).toLowerCase()`。
- `loginId`・`domain`・`group`名は本要件の対象外(ユーザーから要求が無いためYAGNI)。

### R5: グループ/タグ選択時にスクロールを先頭へ戻す

- タグナビclickハンドラ(1797-1803)・グループナビclickハンドラ(1823-1829)で、`state.tag`/`state.group`を変更して`render()`を呼んだ直後に`window.scrollTo({ top: 0, behavior: 'auto' })`を呼ぶ。
- 「すべて解除」ボタン(`#clear-all-filters-btn`)のクリックハンドラにも同様に適用する(フィルタ解除も表示内容が変わるため)。
- スクロール対象は`window`で確定する(実装時の確認は不要): `.main`(`bookmarks.html:449-453`)に`overflow`指定は無く、`overflow-y: auto`は`.sidebar`(`:369`、`@media (min-width:761px)`内)と`.modal`(`:593`)専用でどちらも本要件のスクロール対象ではない。`html, body`(`:27-34`)にも`overflow`指定が無いため、ページ全体のスクロールは常に`window`レベルで発生する。

### R6: リスト行にタグ表記を追加

- リスト行(`renderCardList`→R1で`renderCard`に統合)のタイトル行付近に、タグが存在する場合のみ小さなチップ群を表示する。
- 既存のサイドバーのタグナビ等で使われているタグチップのスタイルと視覚的に一貫させる(新規に別系統のスタイルを作らない)。空の場合は何も表示しない(既存のメモプレビューチップの「無ければ何も出さない」方針と同じ)。

### R7: ショートカットキーの最小化

- `handleGlobalShortcut()`(3110-3168、filesync 3361-3419)から以下を削除する: `n`(新規追加)、`t`(タグタブ切替)、`g`(グループタブ切替)、`?`(ヘルプ)、`x`(フィルタ全解除)。
- 残す: `/`(検索フォーカス)、矢印キー/Home/Endによるナビ移動。
- `handleCardShortcut()`(3171-3209、filesync 3422-3460)は変更しない(Enter/`e`/Delete/Backspace/矢印キーによるカード間移動は全て維持)。
- Escapeチェーン(`closeTopmostLayer()`、3046-3099/3297-3350)は変更しない。
- ヘルプパネルの説明文(どのキーが何をするかの一覧)を、削除後の一覧に合わせて更新する。削除した5キーに対応するボタン(新規追加ボタン、タグ/グループタブ、ヘルプボタン、フィルタ全解除ボタン)自体はクリックで引き続き操作可能であり、キーボードショートカットのみを削除する。

## 既存アーキテクチャとの整合性

- R1によって、CLAUDE.mdの「3段階密度モード」節全体が無効化される。本Spec実装後、CLAUDE.md/README.mdのドキュメント同期(doc-sync)で、密度モード関連の記載を全て削除し、リスト表示を唯一の表示形式として記載し直す必要がある。
- `pinned`/`favoriteOrder`/お気に入りセクションの独立性(`data-fav-id`ベース)等、密度モードと無関係な既存アーキテクチャはすべて無変更。
- R3のお気に入り編集機能追加は、CLAUDE.mdの「お気に入りカードは開く・ピン解除のみサポート」という記載された制約を明示的に変更する(設計上の意図的な変更であり、CLAUDE.md更新対象)。

## エッジケース

- メモが無いブックマークはR4の検索対象に空文字列が加わるのみで、既存の検索結果に影響しない。
- タグが1個も無いブックマークはR6でチップ領域自体を表示しない(既存のメモプレビューと同じ「空なら非表示」方針)。
- お気に入りセクションでR3のコピー/編集ボタンを使った結果(編集でピン解除された場合等)、お気に入りセクションからその行が消えることは既存の`render()`再描画で自然に処理される(新規の考慮は不要)。
- ショートカット削除後も、削除したキーに対応するボタン(新規追加・タブ切替・ヘルプ・フィルタ全解除)自体のクリック操作・既存のaria属性・フォーカス制御は無変更。

## 非対象(スコープ外)

- ダークモード・モバイル最適化は対象外(既存スコープ外のまま)。
- `loginId`/`domain`/`group`名を検索対象に含めることは対象外(R4はメモのみ)。
- `bookmarks.html`の頻度順ソート機能(`state.sortMode`)への変更は対象外。

## リスク・懸念点

- R1はR3・R6の前提(リスト行のみが存在する状態)を作る変更のため、実装順序はR1 → (R2, R3, R6) → R4, R5, R7 という依存関係がある。R2/R3/R6はR1完了後でなければ対象のレンダリング関数が確定しない。
- R1の削除範囲が広い(密度トグルUI・CSS・state・localStorageキー・3つのレンダー関数)ため、削除漏れが残存するリスクがある。実装後に`grep -n "densityMode\|DENSITY_MODE_KEY\|card--compact\|group-grid--compact\|card--list\|group-grid--list\|card-favorite--list\|renderCardComfy\|renderCardCompact\|renderFavoriteCardComfy"`で両ファイルに残存が無いことを確認する。
