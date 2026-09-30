# お気に入り・よく使う順・ブックマークレット登録・UNCパス対応 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** bookmark-shelf（`bookmarks.html` / `bookmarks-filesync.html`）に、お気に入りピン留め、よく使う順自動並び替え、ブックマークレットによる登録支援、NFS/UNCフォルダパス対応の4機能を追加する。

**Architecture:** 既存の単一HTMLファイル・IIFE構成（ビルド・依存ゼロ）を維持する。お気に入りは既存の`#group-sections`とは別の独立コンテナ（`#favorites-section`）・専用レンダー関数として実装し、既存の`data-card-id`ベースの単一要素イベント割り当てループ（グループ内カードの編集・削除・タグ・ドラッグ等）に一切手を加えない設計にする（design-audit CP-Aで確定した方針）。よく使う順は`order`フィールドを書き換えない動的ソートとして`sortItems()`に集約する。ブックマークレットは`location.search`のクエリパラメータ解析＋既存の接続確立フック（filesync版は`setConnected()`）への合流で実現する。

**Tech Stack:** Vanilla JS (ES5相当)、localStorage / File System Access API、CSS。新規ライブラリ・依存の追加なし。

**Spec:** `docs/superpowers/specs/2026-09-30-favorites-frequency-sort-bookmarklet-unc-design.md`（design-audit CP-A PASS済み、コミット9849f48）

## Global Constraints

- 両ファイル（`bookmarks.html`・`bookmarks-filesync.html`）に同一パターンを適用するのが原則。ただしR3（並び替えモード・アクセス記録）は`bookmarks.html`限定（Spec「対象ファイルと変更範囲」節）。
- 新規の外部依存・ビルドツールを追加しない（CLAUDE.md記載のプロジェクト方針）。
- コード識別子・関数名は英語、UI文言・コメントは既存パターンに合わせ日本語。
- テストは`browser-manual-e2e`プロファイル（自動テストランナーなし、claude-in-chromeで実ブラウザ操作しRED/GREENをJS評価・スクリーンショットで確認、`progress.md`に証拠として記録）。CIステージ整備（CP-E）はスキップ（プロジェクトにCI/ビルドパイプラインが存在しないため）。
- `order`/`groupOrder`フィールドは既存の手動並び替えの正典であり、よく使う順モードはこれらを書き換えない（表示直前の動的ソートのみ）。

## Review Focus

- UNCパス（`\\server\share`）が`classifyUrl()`内で`http`/`https`/`file`と誤判定されないこと（判定順序と正規表現の排他性）
- `accessCount===0`または`lastAccessed===null`のブックマークが「よく使う順」で常に最下位（スコア0）になること
- `bookmarks-filesync.html`に並び替えモードのUI・アクセス記録処理が一切出現しないこと（R3はbookmarks.html限定）
- お気に入りセクションの空欄に他グループのカードをドロップしても、対象カードの`group`が変化しないこと（構造的に`.group-section`のドロップターゲット機構の対象外であることの確認）
- `bookmarks-filesync.html`が未接続の状態でブックマークレット経由（`?url=...`付き）で起動されても、接続確立前に保存不能なデータ入力が発生しないこと（接続確立後に自動オープンされること）

---

### Task 1: データモデル拡張（pinned/lastAccessed/accessCount）とマイグレーション

**Files:**
- Modify: `bookmarks.html:1186-1194`（`loadData()`内の既存マイグレーション部）
- Modify: `bookmarks.html:1311-1325`（`sanitizeBookmark()`の戻り値オブジェクト）
- Modify: `bookmarks.html:2351-2364`（追加フォーム送信時の`bookmarks.push({...})`）
- Modify: `bookmarks-filesync.html:1613-1627`（`sanitizeBookmark()`の戻り値オブジェクト。`readBookmarksFromHandle()`は全エントリを`sanitizeBookmark()`経由で読むため、filesync側に個別のマイグレーションループは不要）
- Modify: `bookmarks-filesync.html:2653-2666`（追加フォーム送信時の`bookmarks.push({...})`。bookmarks.htmlと同一パターン、offset+302）

**Interfaces:**
- Produces: `bookmark.pinned: boolean`、`bookmark.lastAccessed: number | null`、`bookmark.accessCount: number`（全タスクの前提となるデータフィールド）

- [ ] **Step 1: シナリオを定義する（RED確認用）**

  シナリオ: 「pinned/lastAccessed/accessCountを持たない旧形式データを読み込むと、欠損フィールドが補完される」
  - RED: 実装前は`localStorage`に`{id:'x', url:'https://example.com', title:'t', tags:[], group:null, groupOrder:0, order:0}`（新フィールド無し）を`bookmarks_data`キーで保存してページをリロードした際、`JSON.parse(localStorage.getItem('bookmarks_data'))[0].pinned`が`undefined`のままになる。
  - GREEN: リロード後、同じ評価で`pinned === false`、`lastAccessed === null`、`accessCount === 0`になる。

- [ ] **Step 2: claude-in-chromeでbookmarks.htmlを開き、実装前の状態でシナリオを実行しREDを確認する**

  `file:///home/asama/bookmarks-shelf/bookmarks.html`を開き、上記の旧形式データを`localStorage.setItem`で注入してリロード、JS評価で`pinned`が`undefined`であることを確認する。

- [ ] **Step 3: `loadData()`のマイグレーション部を拡張する（`bookmarks.html:1186-1194`）**

  既存の`needsOrderMigration`/`needsGroupOrderMigration`と同じパターンで、新規フィールドの欠損検知・補完を追加する。

  ```js
  // 手動並べ替え機能の導入前に保存されたデータには order が無いため、読み込み時の並び順で補完する
  var needsOrderMigration = parsed.some(function (b) { return typeof b.order !== 'number'; });
  if (needsOrderMigration) parsed.forEach(function (b, i) { b.order = i; });

  // グループ並べ替え機能の導入前に保存されたデータには groupOrder が無いため、現在のアルファベット順で補完する
  var needsGroupOrderMigration = parsed.some(function (b) { return typeof b.groupOrder !== 'number'; });
  if (needsGroupOrderMigration) assignGroupOrder(parsed);

  // お気に入り・よく使う順機能の導入前に保存されたデータには pinned/lastAccessed/accessCount が無いため、初期値で補完する
  var needsFavFieldsMigration = parsed.some(function (b) { return typeof b.pinned !== 'boolean'; });
  if (needsFavFieldsMigration) {
    parsed.forEach(function (b) {
      if (typeof b.pinned !== 'boolean') b.pinned = false;
      if (typeof b.lastAccessed !== 'number') b.lastAccessed = null;
      if (typeof b.accessCount !== 'number') b.accessCount = 0;
    });
  }

  if (needsOrderMigration || needsGroupOrderMigration || needsFavFieldsMigration) localStorage.setItem(STORAGE_KEY, JSON.stringify(parsed));
  return parsed;
  ```

- [ ] **Step 4: `sanitizeBookmark()`の戻り値に新規フィールドを追加する（`bookmarks.html:1311-1325`・`bookmarks-filesync.html:1613-1627`、同一パターン）**

  ```js
  return {
    id: makeId(),
    url: url,
    title: title,
    tags: tags,
    group: group,
    groupOrder: typeof raw.groupOrder === 'number' ? raw.groupOrder : undefined,
    loginId: loginId,
    loginPassword: loginPassword,
    memo: memo,
    scheme: scheme,
    domain: deriveDomain(url, scheme),
    order: 0, // インポート確定後にファイル内の並び順で採番し直す
    pinned: typeof raw.pinned === 'boolean' ? raw.pinned : false,
    lastAccessed: typeof raw.lastAccessed === 'number' ? raw.lastAccessed : null,
    accessCount: typeof raw.accessCount === 'number' && raw.accessCount >= 0 ? Math.floor(raw.accessCount) : 0
  };
  ```

- [ ] **Step 5: 追加フォーム送信時の新規ブックマーク生成にも新規フィールドを追加する（`bookmarks.html:2351-2364`・`bookmarks-filesync.html:2653-2666`、同一パターン）**

  ```js
  bookmarks.push({
    id: makeId(),
    url: url,
    title: title,
    tags: tags,
    group: group,
    groupOrder: getGroupOrderForName(group),
    loginId: loginId,
    loginPassword: loginPassword,
    memo: memo,
    scheme: scheme,
    domain: deriveDomain(url, scheme),
    order: bookmarks.length,
    pinned: false,
    lastAccessed: null,
    accessCount: 0
  });
  ```

- [ ] **Step 6: claude-in-chromeで同一シナリオを再実行しGREENを確認する**

  Step2と同じ手順でリロードし、`pinned === false && lastAccessed === null && accessCount === 0`を確認する。

- [ ] **Step 7: JSONエクスポート/インポートでの往復を確認する（追加シナリオ）**

  `pinned:true, lastAccessed:1700000000000, accessCount:5`を持つブックマークをエクスポート→インポートし、`sanitizeBookmark()`経由で値が保持されること（`accessCount:5`かつ`pinned:true`）をJS評価で確認する。

- [ ] **Step 8: コミット**

  ```bash
  git add bookmarks.html bookmarks-filesync.html
  git commit -m "feat: pinned/lastAccessed/accessCountフィールドを追加しマイグレーションを実装"
  ```

---

### Task 2: `classifyUrl()`のUNC対応と`deriveTentativeTitle()`のUNC対応

**Files:**
- Modify: `bookmarks.html:1216-1221`（`classifyUrl()`）
- Modify: `bookmarks.html:1223-1230`（`deriveDomain()`）
- Modify: `bookmarks.html:1232-1246`（`deriveTentativeTitle()`）
- Modify: `bookmarks-filesync.html`（対応箇所、bookmarks.htmlと同一パターン、offset+302前後。実装前にgrepで正確な行番号を確認する）

**Interfaces:**
- Produces: `classifyUrl(raw)`が`'\\server\share\folder'`形式の文字列に対し`'unc'`を返す。`deriveDomain(url, 'unc')`は`null`を返す。`deriveTentativeTitle(url, 'unc')`はパス末尾のフォルダ/ファイル名を返す。

- [ ] **Step 1: シナリオを定義する（RED確認用）**

  シナリオ: 「UNCパスを追加フォームのURL欄に貼り付けると、フォーム送信時にバリデーションエラーにならず登録できる」
  - RED: 実装前は`classifyUrl('\\\\server\\share\\folder')`が`null`を返すため、追加フォーム送信時に「URLは http:// / https:// / file:/// のいずれかで始まる必要があります。」のアラートが出て登録できない。
  - GREEN: 実装後は`classifyUrl('\\\\server\\share\\folder') === 'unc'`となり、登録が成功する。

- [ ] **Step 2: claude-in-chromeでbookmarks.htmlを開き、実装前の状態でシナリオを実行しREDを確認する**

  追加フォームのURL欄に`\\server\share\folder`を入力して送信し、アラートが表示されることを確認する（またはコンソールで`classifyUrl('\\\\server\\share\\folder')`を直接評価し`null`であることを確認する）。

- [ ] **Step 3: `classifyUrl()`にUNC判定を追加する**

  ```js
  function classifyUrl(raw) {
    if (/^https:\/\//i.test(raw)) return 'https';
    if (/^http:\/\//i.test(raw)) return 'http';
    if (/^\\\\/.test(raw)) return 'unc';
    if (/^file:\/\/\//i.test(raw)) return 'file';
    return null;
  }
  ```

- [ ] **Step 4: `deriveDomain()`にUNC分岐を追加する**

  ```js
  function deriveDomain(url, scheme) {
    if (scheme === 'file' || scheme === 'unc') return null;
    try {
      return new URL(url).hostname;
    } catch (e) {
      return null;
    }
  }
  ```

- [ ] **Step 5: `deriveTentativeTitle()`にUNC分岐を追加する**

  ```js
  function deriveTentativeTitle(url, scheme) {
    try {
      if (scheme === 'file') {
        var rawPath = url.replace(/^file:\/\//i, '');
        var path;
        try { path = decodeURIComponent(rawPath); } catch (e2) { path = rawPath; }
        var segments = path.split('/').filter(Boolean);
        return segments.length ? segments[segments.length - 1] : path;
      }
      if (scheme === 'unc') {
        var uncSegments = url.split('\\').filter(Boolean);
        return uncSegments.length ? uncSegments[uncSegments.length - 1] : url;
      }
      return new URL(url).hostname;
    } catch (e) {
      return '';
    }
  }
  ```

- [ ] **Step 6: `bookmarks-filesync.html`の対応箇所に同一の変更を適用する**

  `grep -n "function classifyUrl" bookmarks-filesync.html`で正確な行番号を確認してから、Step3〜5と同一の変更を適用する。

- [ ] **Step 7: claude-in-chromeで同一シナリオを再実行しGREENを確認する**

  `\\server\share\folder`を追加フォームに入力して送信し、エラーなく登録されカード一覧に表示されることを確認する。あわせて`classifyUrl('https://example.com') === 'https'`など既存スキームが誤判定されていないことも確認する（回帰確認）。

- [ ] **Step 8: コミット**

  ```bash
  git add bookmarks.html bookmarks-filesync.html
  git commit -m "feat: classifyUrl()にUNCパス(\\\\server\\share)判定を追加"
  ```

---

### Task 3: UNC/file://パスの「開く」リンク修正と「パスをコピー」ボタン追加

**Files:**
- Modify: `bookmarks.html:1248`付近（`resolveOpenHref()`新規関数、`deriveTentativeTitle()`の直後、`/* ---------- ユーティリティ ---------- */`セクション）
- Modify: `bookmarks.html:2122-2182`（`renderCard()`: タイトルリンクのhref・「パスをコピー」ボタン追加）
- Modify: `bookmarks.html:1692-1742`（カード単位イベント割り当てループ: `copy-path`アクション追加）
- Modify: `bookmarks-filesync.html`（対応箇所、同一パターン）

**Interfaces:**
- Consumes: Task1の`bookmark`型、Task2の`classifyUrl()`/`scheme`判定
- Produces: `resolveOpenHref(b): string`（他タスク・Task5の`renderFavoriteCard()`から再利用する）

- [ ] **Step 1: シナリオを定義する（RED確認用）**

  シナリオ: 「UNCパスのカードに『パスをコピー』ボタンが表示され、クリックでクリップボードに元のパス文字列がコピーされる」
  - RED: 実装前はUNCパスのカードに`data-action="copy-path"`のボタンが存在しない。
  - GREEN: 実装後はボタンが存在し、クリックすると`navigator.clipboard`（またはfallback）に`\\server\share\folder`がコピーされ、ボタンテキストが「コピーしました」に変わる。

- [ ] **Step 2: claude-in-chromeでbookmarks.htmlを開き、実装前の状態でシナリオを実行しREDを確認する**

  Task2で登録したUNCパスのカードに`[data-action="copy-path"]`が存在しないことをJS評価（`document.querySelector('[data-action="copy-path"]') === null`）で確認する。

- [ ] **Step 3: `resolveOpenHref()`ヘルパー関数を追加する（`deriveTentativeTitle()`の直後）**

  ```js
  // UNCパスはブラウザのfile://ハンドリングに直接渡せないため、表示直前にfile://表記へ変換する。
  // 保存値(b.url)自体は変換しない(design-audit CP-Aで確定: 環境依存の表記ゆれをsanitizeBookmark段階に持ち込まないため)
  function resolveOpenHref(b) {
    if (b.scheme === 'unc') return 'file:' + b.url.replace(/\\/g, '/');
    return b.url;
  }
  ```

- [ ] **Step 4: `renderCard()`のタイトルリンクhrefと「パスをコピー」ボタンを追加する（`bookmarks.html:2159-2182`）**

  ```js
  var pathCopyHtml = (scheme === 'file' || scheme === 'unc')
    ? '<button type="button" tabindex="-1" class="btn-quiet" data-action="copy-path">パスをコピー</button>'
    : '';

  return (
    '<article class="card" data-card-id="' + escapeHtml(b.id) + '" tabindex="-1">' +
      '<div class="card-stub">' + grip + '</div>' +
      '<div class="card-body">' +
        '<div class="card-top">' +
          checkboxHtml +
          '<div class="seal">' + iconHtml + '</div>' +
          '<div class="card-title-wrap">' +
            '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer" tabindex="-1">' + escapeHtml(b.title) + '</a></h3>' +
          '</div>' +
        '</div>' +
        (tagsHtml ? '<div class="card-tags">' + tagsHtml + '</div>' : '') +
        credentialsHtml +
        memoHtml +
        '<div class="card-footer">' +
          '<div class="card-actions">' +
            pathCopyHtml +
            '<button type="button" tabindex="-1" class="btn-quiet" data-action="edit">編集</button>' +
            '<button type="button" tabindex="-1" class="btn-danger" data-action="delete">削除</button>' +
          '</div>' +
        '</div>' +
      '</div>' +
    '</article>'
  );
  ```

- [ ] **Step 5: カード単位イベント割り当てループに`copy-path`ハンドラを追加する（`bookmarks.html:1738-1740`付近、`copyPwBtn`ブロックの直後）**

  ```js
  var copyPathBtn = card.querySelector('[data-action="copy-path"]');
  if (copyPathBtn) copyPathBtn.addEventListener('click', function () { copyToClipboard(b.url, copyPathBtn); });
  ```

- [ ] **Step 6: `bookmarks-filesync.html`の対応箇所（`renderCard()`・カード単位イベントループ）に同一の変更を適用する**

- [ ] **Step 7: claude-in-chromeで同一シナリオを再実行しGREENを確認する**

  UNCパスのカードで「パスをコピー」ボタンをクリックし、ボタンテキストが「コピーしました」に変わることを確認する。あわせて、UNCカードのタイトルリンクの`href`が`file://server/share/folder`形式になっていること（`document.querySelector('.card-title a').getAttribute('href')`で確認）、既存のhttps/fileスキームのカードのリンク・favicon表示が従来通りであること（回帰確認）も確認する。

- [ ] **Step 8: コミット**

  ```bash
  git add bookmarks.html bookmarks-filesync.html
  git commit -m "feat: UNC/file://パスの開くリンク変換とパスコピーボタンを追加"
  ```

---

### Task 4: ピン留めトグルボタンの追加

**Files:**
- Modify: `bookmarks.html:2122-2182`（`renderCard()`: ピン留めボタン追加）
- Modify: `bookmarks.html:1692-1742`（カード単位イベント割り当てループ: `toggle-pin`アクション追加）
- Modify: `bookmarks-filesync.html`（対応箇所、同一パターン）

**Interfaces:**
- Consumes: Task1の`bookmark.pinned`
- Produces: カード上の`[data-action="toggle-pin"]`ボタン（クリックで`b.pinned`をトグルし保存・再描画）

- [ ] **Step 1: シナリオを定義する（RED確認用）**

  シナリオ: 「カードのピン留めボタンをクリックすると`pinned`がtrueになり、ボタンの見た目が変わる」
  - RED: 実装前はカードに`[data-action="toggle-pin"]`ボタンが存在しない。
  - GREEN: 実装後はボタンが存在し、クリックすると対象ブックマークの`pinned`が`true`になり、`localStorage`に保存され、ボタンに`active`クラスが付く。

- [ ] **Step 2: claude-in-chromeでbookmarks.htmlを開き、実装前の状態でシナリオを実行しREDを確認する**

  任意のカードで`[data-action="toggle-pin"]`が存在しないことを確認する。

- [ ] **Step 3: `renderCard()`にピン留めボタンを追加する（`bookmarks.html:2173-2178`の`.card-actions`内、編集ボタンの前）**

  ```js
  '<div class="card-actions">' +
    pathCopyHtml +
    '<button type="button" tabindex="-1" class="btn-quiet' + (b.pinned ? ' active' : '') + '" data-action="toggle-pin" aria-label="' + (b.pinned ? 'ピン留めを解除' : 'ピン留め') + '">' + (b.pinned ? '★' : '☆') + '</button>' +
    '<button type="button" tabindex="-1" class="btn-quiet" data-action="edit">編集</button>' +
    '<button type="button" tabindex="-1" class="btn-danger" data-action="delete">削除</button>' +
  '</div>' +
  ```

- [ ] **Step 4: カード単位イベント割り当てループに`toggle-pin`ハンドラを追加する（`bookmarks.html:1710-1714`付近、`editBtn`/`delBtn`と同じ並び）**

  ```js
  var pinBtn = card.querySelector('[data-action="toggle-pin"]');
  if (pinBtn) pinBtn.addEventListener('click', function () {
    b.pinned = !b.pinned;
    saveData(bookmarks);
    render();
  });
  ```

- [ ] **Step 5: `bookmarks-filesync.html`の対応箇所に同一の変更を適用する**

- [ ] **Step 6: claude-in-chromeで同一シナリオを再実行しGREENを確認する**

  ピン留めボタンをクリックし、ボタンが★表示・`active`クラス付きに変わること、`localStorage.getItem('bookmarks_data')`（filesync版はファイル内容）に`pinned:true`が反映されることを確認する。もう一度クリックして元に戻ることも確認する。

- [ ] **Step 7: コミット**

  ```bash
  git add bookmarks.html bookmarks-filesync.html
  git commit -m "feat: カードにピン留めトグルボタンを追加"
  ```

---

### Task 5: お気に入りセクションのレンダリング

**Files:**
- Modify: `bookmarks.html:1121`（HTML: `#favorites-section`コンテナを`#group-sections`の直前に追加）
- Modify: `bookmarks.html:1388-1428`（`render()`: `renderFavoritesSection()`呼び出しを追加）
- Modify: `bookmarks.html:1650`付近（`renderFavoritesSection()`・`renderFavoriteCard()`を新規関数として`renderSections()`の直前に追加）
- Modify: `bookmarks.html`のCSS（`.group-section`定義群の直後に`.favorites-section`・`.card-favorite`のスタイルを追加）
- Modify: `bookmarks-filesync.html`（対応箇所、同一パターン。ただしfilesync側の「開く」リンクにはアクセス記録を付けない＝Task4までの内容のみ）

**Interfaces:**
- Consumes: Task1の`bookmark.pinned`、Task3の`resolveOpenHref()`、既存の`getSearchTagFiltered()`・`sortItems()`・`escapeHtml()`
- Produces: `renderFavoritesSection()`（`render()`から呼ばれる）、`renderFavoriteCard(b)`

**設計メモ（design-audit CP-A確定事項の実装方針）**: お気に入りセクションのカードは`data-card-id`属性を持たず`data-fav-id`を使う独自のDOM要素とし、`#group-sections`の外側（`#favorites-section`）に描画する。これにより、既存のカード単位イベント割り当てループ（`bookmarks.html:1692-1742`）・`bindGroupSectionDragEvents()`（`.group-section`クラスのみを対象）・`getNavigableCards()`（`#group-sections .card[tabindex]`のみを対象）のいずれからも自動的に除外される。**この3箇所には一切手を加えない**（コード変更不要、構造的な分離のみで要件を満たす）。

- [ ] **Step 1: シナリオを定義する（RED確認用）**

  シナリオ: 「ブックマークをピン留めすると、画面最上部の『お気に入り』セクションに表示され、かつ元のグループ内表示にも残る」
  - RED: 実装前はピン留めしても専用セクションが存在せず、`document.getElementById('favorites-section')`が存在しない（またはHTML未追加のためnull）。
  - GREEN: 実装後はピン留め後に`#favorites-section`内に該当カード（`[data-fav-id="<id>"]`）が現れ、かつ`#group-sections`内の元のカード（`[data-card-id="<id>"]`）も引き続き存在する。

- [ ] **Step 2: claude-in-chromeでbookmarks.htmlを開き、実装前の状態でシナリオを実行しREDを確認する**

  任意のカードをピン留め後、`document.getElementById('favorites-section')`が`null`であることを確認する。

- [ ] **Step 3: HTMLに`#favorites-section`コンテナを追加する（`bookmarks.html:1119-1122`）**

  ```html
  <main class="main">
    <p class="result-count" id="result-count"></p>
    <div id="favorites-section"></div>
    <div id="group-sections"></div>
  </main>
  ```

- [ ] **Step 4: `renderFavoritesSection()`・`renderFavoriteCard()`を追加する（`renderSections()`の直前、`bookmarks.html:1650`付近）**

  ```js
  function renderFavoriteCard(b) {
    var scheme = b.scheme;
    var iconHtml;
    if (scheme === 'file' || scheme === 'unc') {
      iconHtml = document.getElementById('icon-file').innerHTML;
    } else {
      var faviconUrl = 'https://www.google.com/s2/favicons?domain=' + encodeURIComponent(b.domain || '') + '&sz=64';
      iconHtml = '<img src="' + escapeHtml(faviconUrl) + '" alt="" onerror="this.replaceWith(document.getElementById(\'icon-globe\').content.cloneNode(true).firstElementChild)">';
    }
    return (
      '<article class="card card-favorite" data-fav-id="' + escapeHtml(b.id) + '">' +
        '<div class="card-body">' +
          '<div class="card-top">' +
            '<div class="seal">' + iconHtml + '</div>' +
            '<div class="card-title-wrap">' +
              '<h3 class="card-title"><a href="' + escapeHtml(resolveOpenHref(b)) + '" target="_blank" rel="noopener noreferrer">' + escapeHtml(b.title) + '</a></h3>' +
            '</div>' +
          '</div>' +
          '<div class="card-footer">' +
            '<div class="card-actions">' +
              '<button type="button" class="btn-quiet" data-action="unpin">ピン解除</button>' +
            '</div>' +
          '</div>' +
        '</div>' +
      '</article>'
    );
  }

  // お気に入りは検索/タグ/グループの絞り込みルールを通常セクションと共有する(getSearchTagFilteredを再利用)。
  // #group-sections とは独立したDOMツリーのため、data-card-idベースのイベント割り当てループ・
  // bindGroupSectionDragEvents()・getNavigableCards()のいずれからも対象外になる(意図的な設計)。
  function renderFavoritesSection() {
    var container = document.getElementById('favorites-section');
    var pinned = getSearchTagFiltered().filter(function (b) { return b.pinned; });
    if (!pinned.length) {
      container.innerHTML = '';
      return;
    }
    var items = sortItems(pinned);
    container.innerHTML =
      '<section class="favorites-section">' +
        '<div class="group-section-header"><h2 class="group-section-title">お気に入り</h2><span class="group-section-count">' + items.length + ' 件</span></div>' +
        '<div class="group-grid">' + items.map(renderFavoriteCard).join('') +
        '</div>' +
      '</section>';

    container.querySelectorAll('[data-fav-id]').forEach(function (el) {
      var id = el.getAttribute('data-fav-id');
      var b = bookmarks.filter(function (x) { return x.id === id; })[0];
      if (!b) return;
      var unpinBtn = el.querySelector('[data-action="unpin"]');
      if (unpinBtn) unpinBtn.addEventListener('click', function () {
        b.pinned = false;
        saveData(bookmarks);
        render();
      });
    });
  }
  ```

- [ ] **Step 5: `render()`から`renderFavoritesSection()`を呼ぶ（`bookmarks.html:1418`、`renderSections()`の直前）**

  ```js
  renderFavoritesSection();
  renderSections();
  ```

- [ ] **Step 6: CSSに`.favorites-section`・`.card-favorite`を追加する（`.group-section`関連クラス定義の直後）**

  ```css
  .favorites-section {
    border: 1px solid var(--amber);
    border-radius: 12px;
    margin-bottom: 24px;
    padding: 12px;
  }
  .favorites-section .group-section-title { color: var(--amber-ink); }
  .card-favorite { padding: 10px 12px; }
  .card-favorite .card-actions { justify-content: flex-end; }
  ```

- [ ] **Step 7: `bookmarks-filesync.html`の対応箇所（HTML・`render()`・新規関数・CSS）に同一の変更を適用する**

- [ ] **Step 8: claude-in-chromeで同一シナリオを再実行しGREENを確認する**

  カードをピン留めし、`#favorites-section`内に`[data-fav-id]`カードが現れること、`#group-sections`内の元のカードも引き続き存在すること、ピン留めが1件も無い状態では`#favorites-section`が空（非表示相当）であること、検索・タグ絞り込みでお気に入りセクションも連動して絞り込まれることを確認する。「ピン解除」ボタンでお気に入りセクションから消え、グループ内表示は残ることも確認する。

- [ ] **Step 9: 選択モード・ドラッグ&ドロップとの非干渉を確認する（Review Focus該当）**

  選択モードをオンにしても`#favorites-section`内にチェックボックスが出ないこと、グループ内カードを`#favorites-section`の空欄にドラッグしても何も起きず`group`が変化しないこと（`.favorites-section`が`.group-section`クラスを持たないため`bindGroupSectionDragEvents()`の対象外であることをJS評価`document.querySelector('.favorites-section').classList.contains('group-section') === false`で確認）、矢印キーでのカード間フォーカス移動がお気に入りセクションのカードに移らないこと（`getNavigableCards()`が`#group-sections .card[tabindex]`のみを対象とするため）を確認する。

- [ ] **Step 10: コミット**

  ```bash
  git add bookmarks.html bookmarks-filesync.html
  git commit -m "feat: お気に入り専用セクションを追加(重複表示・簡易カード)"
  ```

---

### Task 6: 並び替えモードのUIとアクセス記録機構（`bookmarks.html`のみ）

**Files:**
- Modify: `bookmarks.html:1011-1012`（HTML: `#toggle-sort-mode-btn`を`.header-actions`に追加）
- Modify: `bookmarks.html:1163-1176`（`state`オブジェクトに`sortMode`を追加）
- Modify: `bookmarks.html:1367-1371`（`sortItems()`: 並び替えモード分岐）
- Modify: `bookmarks.html:1423-1428`（`render()`: トグルボタンの表示更新）
- Modify: `bookmarks.html:1692-1742`（カード単位イベント割り当てループ: タイトルリンクのアクセス記録）
- Modify: `bookmarks.html`（トグルボタンのクリックハンドラを追加フォーム関連コードの近くに新規追加）

**Interfaces:**
- Consumes: Task1の`bookmark.lastAccessed`/`accessCount`
- Produces: `state.sortMode: 'manual' | 'frequency'`、`frequencyScore(b): number`（Task7で再利用）

**注意**: このTaskは`bookmarks.html`のみが対象。`bookmarks-filesync.html`には一切変更を加えない（Spec R3の明示的なスコープ制限）。

- [ ] **Step 1: シナリオを定義する（RED確認用）**

  シナリオ1: 「カードのタイトルリンクをクリックすると`accessCount`がインクリメントされ`lastAccessed`が更新される」
  - RED: 実装前はクリックしても`accessCount`/`lastAccessed`が変化しない。
  - GREEN: 実装後はクリック後に対象ブックマークの`accessCount`が1増え、`lastAccessed`が現在時刻に更新される。

  シナリオ2: 「『よく使う順』トグルボタンが存在し、クリックで`state.sortMode`が切り替わる」
  - RED: 実装前は`#toggle-sort-mode-btn`が存在しない。
  - GREEN: 実装後はボタンが存在し、クリックのたびに`state.sortMode`が`'manual'`⇔`'frequency'`を往復する。

- [ ] **Step 2: claude-in-chromeでbookmarks.htmlを開き、実装前の状態でシナリオを実行しREDを確認する**

  任意のカードのタイトルリンクをクリックしても`accessCount`が変化しないこと、`#toggle-sort-mode-btn`が存在しないことを確認する。

- [ ] **Step 3: HTMLにトグルボタンを追加する（`bookmarks.html:1011-1012`、`#toggle-select-btn`の直後）**

  ```html
  <button class="btn btn-amber" id="toggle-add-btn" type="button">+ 追加</button>
  <button class="btn btn-ghost" id="toggle-select-btn" type="button">選択</button>
  <button class="btn btn-ghost" id="toggle-sort-mode-btn" type="button">よく使う順で表示</button>
  <button class="btn btn-ghost" id="export-btn" type="button">エクスポート</button>
  ```

- [ ] **Step 4: `state`に`sortMode`を追加する（`bookmarks.html:1163-1176`）**

  ```js
  var STORAGE_KEY = 'bookmarks_data';
  var SORT_MODE_KEY = 'bookmarks_sort_mode';
  var UNGROUPED = null;

  function loadSortMode() {
    try { return localStorage.getItem(SORT_MODE_KEY) === 'frequency' ? 'frequency' : 'manual'; }
    catch (e) { return 'manual'; }
  }

  var state = {
    search: '',
    tag: null,
    group: null,
    editingId: null,
    editingGroup: null,
    editingTag: null,
    selectMode: false,
    selectedIds: new Set(),
    sidebarTab: 'group',
    sortMode: loadSortMode()
  };
  ```

- [ ] **Step 5: `sortItems()`を並び替えモード対応にする（`bookmarks.html:1367-1371`）**

  ```js
  // 未アクセス(accessCount===0)は常に最小スコア(0)になるため、lastAccessedがnullでも
  // 「十分大きい経過日数」のような仮値を用意せず自然に最下位へ落ちる
  function frequencyScore(b) {
    if (!b.accessCount) return 0;
    var daysSinceLastAccess = Math.max(0, (Date.now() - b.lastAccessed) / 86400000);
    return b.accessCount / (daysSinceLastAccess + 1);
  }

  function sortItems(items) {
    if (state.sortMode === 'frequency') {
      return items.slice().sort(function (a, b) { return frequencyScore(b) - frequencyScore(a); });
    }
    return items.slice().sort(function (a, b) {
      return (a.order || 0) - (b.order || 0);
    });
  }
  ```

- [ ] **Step 6: トグルボタンのクリックハンドラを追加する（追加フォーム関連コードの近く、`bookmarks.html:2321-2324`の`toggle-add-btn`ハンドラの直後）**

  ```js
  document.getElementById('toggle-sort-mode-btn').addEventListener('click', function () {
    state.sortMode = state.sortMode === 'frequency' ? 'manual' : 'frequency';
    try { localStorage.setItem(SORT_MODE_KEY, state.sortMode); } catch (e) {}
    render();
  });
  ```

- [ ] **Step 7: `render()`にトグルボタンの表示更新を追加する（`bookmarks.html:1423-1425`、`selectBtn`の更新の直後）**

  ```js
  var sortModeBtn = document.getElementById('toggle-sort-mode-btn');
  sortModeBtn.textContent = state.sortMode === 'frequency' ? '手動順で表示' : 'よく使う順で表示';
  sortModeBtn.classList.toggle('active', state.sortMode === 'frequency');
  ```

- [ ] **Step 8: カード単位イベント割り当てループにアクセス記録ハンドラを追加する（`bookmarks.html:1741`の直前、`bindDragEvents`呼び出しの手前）**

  ```js
  var titleLink = card.querySelector('.card-title a');
  if (titleLink) {
    titleLink.addEventListener('click', function () {
      b.accessCount = (b.accessCount || 0) + 1;
      b.lastAccessed = Date.now();
      saveData(bookmarks);
      render();
    });
  }

  if (!state.selectMode) bindDragEvents(card, b);
  ```

- [ ] **Step 9: claude-in-chromeで同一シナリオを再実行しGREENを確認する**

  カードのタイトルリンクをクリックし（`target="_blank"`のため新規タブが開くが元のタブは維持される）、`accessCount`が1増え`lastAccessed`が更新されることを確認する。`#toggle-sort-mode-btn`をクリックし、ボタンテキストが「手動順で表示」に変わり`state.sortMode === 'frequency'`になることを確認する。ページをリロードしても`sortMode`が`localStorage`から復元されることも確認する。

- [ ] **Step 10: `bookmarks-filesync.html`に変更が一切無いことを確認する（Review Focus該当）**

  `git diff --stat`で`bookmarks-filesync.html`が変更されていないことを確認する。

- [ ] **Step 11: コミット**

  ```bash
  git add bookmarks.html
  git commit -m "feat: 並び替えモード(手動順/よく使う順)とアクセス記録を追加(bookmarks.htmlのみ)"
  ```

---

### Task 7: 「よく使う順」ソートのお気に入りセクションへの適用とアクセス記録の拡張

**Files:**
- Modify: `bookmarks.html`（`renderFavoriteCard()`のタイトルリンクにアクセス記録ハンドラを追加。Task5で追加した関数）

**Interfaces:**
- Consumes: Task5の`renderFavoriteCard()`・`renderFavoritesSection()`、Task6の`frequencyScore()`・`sortItems()`

**設計メモ**: `renderFavoritesSection()`は既にTask5で`sortItems(pinned)`を呼んでいるため、Task6で`sortItems()`自体を並び替えモード対応にした時点で、お気に入りセクションの並び順は自動的に「よく使う順」モードに追従する（追加のソートロジックは不要）。本Taskで必要なのは、お気に入りセクションの「開く」リンクからもアクセス記録が発火するようにする変更のみ。

- [ ] **Step 1: シナリオを定義する（RED確認用）**

  シナリオ: 「お気に入りセクションの『よく使う順』での並び順が`accessCount`/`lastAccessed`に応じて変わり、お気に入りの『開く』リンクをクリックしてもアクセス記録が発火する」
  - RED: 実装前はお気に入りセクションの「開く」リンクをクリックしても`accessCount`が変化しない（Task5時点では記録ハンドラが無い）。
  - GREEN: 実装後はクリックすると対象ブックマークの`accessCount`が増え、「よく使う順」モードでお気に入りセクション内の並びがスコア降順になる。

- [ ] **Step 2: claude-in-chromeでbookmarks.htmlを開き、実装前の状態でシナリオを実行しREDを確認する**

  お気に入りセクション内のカードの「開く」リンクをクリックしても`accessCount`が変化しないことを確認する。

- [ ] **Step 3: `renderFavoriteCard()`のタイトルリンクにアクセス記録ハンドラを追加する**

  `renderFavoritesSection()`内、既存の`unpinBtn`のイベント割り当てと同じループ（`container.querySelectorAll('[data-fav-id]')`）に以下を追加する。

  ```js
  container.querySelectorAll('[data-fav-id]').forEach(function (el) {
    var id = el.getAttribute('data-fav-id');
    var b = bookmarks.filter(function (x) { return x.id === id; })[0];
    if (!b) return;

    var titleLink = el.querySelector('.card-title a');
    if (titleLink) {
      titleLink.addEventListener('click', function () {
        b.accessCount = (b.accessCount || 0) + 1;
        b.lastAccessed = Date.now();
        saveData(bookmarks);
        render();
      });
    }

    var unpinBtn = el.querySelector('[data-action="unpin"]');
    if (unpinBtn) unpinBtn.addEventListener('click', function () {
      b.pinned = false;
      saveData(bookmarks);
      render();
    });
  });
  ```

- [ ] **Step 4: claude-in-chromeで同一シナリオを再実行しGREENを確認する**

  お気に入りセクションの「開く」リンクをクリックし`accessCount`が増えることを確認する。2件以上ピン留めした状態で「よく使う順」モードに切り替え、`accessCount`の多い方が上に表示されることを確認する。

- [ ] **Step 5: コミット**

  ```bash
  git add bookmarks.html
  git commit -m "feat: お気に入りセクションの開くリンクにもアクセス記録を適用"
  ```

---

### Task 8: `bookmarks.html`のブックマークレットURLパラメータ自動プリフィル

**Files:**
- Modify: `bookmarks.html:2752-2754`（スクリプト末尾、`render();`の直後にクエリパラメータ処理を追加）

**Interfaces:**
- Consumes: Task1〜7で確定した`newUrlInput`/`newTitleInput`/`addPanel`変数（`bookmarks.html:2293-2296`で既に宣言済み）
- Produces: `applyBookmarkletParams()`（起動時に一度だけ呼ばれる）

- [ ] **Step 1: シナリオを定義する（RED確認用）**

  シナリオ: 「`?url=https://example.com&title=Example`付きで起動すると、追加パネルが自動的に開きURL/タイトルがプリフィルされる」
  - RED: 実装前は`?url=...`付きで開いても追加パネルは閉じたまま。
  - GREEN: 実装後は`#add-panel`の`hidden`が`false`になり、`#new-url`の値が`https://example.com`、`#new-title`の値が`Example`になる。

- [ ] **Step 2: claude-in-chromeで`bookmarks.html?url=https%3A%2F%2Fexample.com&title=Example`を開き、実装前の状態でREDを確認する**

  `document.getElementById('add-panel').hidden === true`であることを確認する。

- [ ] **Step 3: クエリパラメータ処理を追加する（`bookmarks.html:2754`の`render();`の直後）**

  ```js
  render();

  (function applyBookmarkletParams() {
    var params = new URLSearchParams(location.search);
    var paramUrl = params.get('url');
    if (!paramUrl) return;
    newUrlInput.value = paramUrl;
    newTitleInput.value = params.get('title') || '';
    addPanel.hidden = false;
    newUrlInput.focus();
    if (!newTitleInput.value) newUrlInput.dispatchEvent(new Event('input')); // タイトル未指定ならURLからの仮タイトル補完(既存ロジック)を発火させる
  })();
  ```

- [ ] **Step 4: claude-in-chromeで同一シナリオを再実行しGREENを確認する**

  `#add-panel`が開き、`#new-url`/`#new-title`にプリフィルされていることを確認する。許可外スキーム（例: `?url=ftp://example.com`）で開いた場合は通常通りプリフィルされ（バリデーションはフォーム送信時のみ行われる既存仕様のため）、送信時にアラートが表示されることも確認する。`?url=`パラメータが無い通常起動では追加パネルが開かないこと（回帰確認）も確認する。

- [ ] **Step 5: コミット**

  ```bash
  git add bookmarks.html
  git commit -m "feat: URLクエリパラメータによる追加フォーム自動プリフィルを追加"
  ```

---

### Task 9: `bookmarks-filesync.html`のブックマークレットURLパラメータ自動プリフィル（接続待ち対応）

**Files:**
- Modify: `bookmarks-filesync.html:1421`付近（`pendingReconnectHandle`宣言の近くに`pendingBookmarkletData`を追加）
- Modify: `bookmarks-filesync.html:1387-1399`（`setConnected()`: 接続確立後にプリフィルを適用するフックを追加）

**Interfaces:**
- Consumes: `setConnected(handle, data)`（`boot()`の成功パス・`open-file-btn`・`create-file-btn`の3経路すべてから呼ばれる既存の共通関数）
- Produces: `applyPendingBookmarkletData()`

**設計メモ（design-audit CP-A再評価で確定した方針）**: `setConnected()`は`boot()`（起動時に既に権限がある場合）・`open-file-btn`・`create-file-btn`の3経路すべてから呼ばれる唯一の「接続確立完了」フックである。ここに1箇所フックを追加するだけで、reconnect/open/createのいずれの経路でも保留中のブックマークレットデータが適用される。

- [ ] **Step 1: シナリオを定義する（RED確認用）**

  シナリオ: 「未接続状態で`?url=...`付きで起動し、ファイルを開いて接続すると、接続直後に追加パネルが自動プリフィルされる」
  - RED: 実装前は接続後も追加パネルは自動で開かない。
  - GREEN: 実装後は`open-file-btn`（または`create-file-btn`・再接続）で接続完了した直後に、`#add-panel`が開きURL/タイトルがプリフィルされる。

- [ ] **Step 2: claude-in-chromeで`bookmarks-filesync.html?url=https%3A%2F%2Fexample.com&title=Example`を開き、実装前の状態でREDを確認する**

  「新規作成」ボタンでファイルを作成・接続した後も`#add-panel`が閉じたままであることを確認する。

- [ ] **Step 3: `pendingBookmarkletData`を宣言し、起動時にクエリパラメータを読み取る（`pendingReconnectHandle`宣言の直後、`bookmarks-filesync.html:1421`付近）**

  ```js
  var pendingReconnectHandle = null;

  // ブックマークレット経由の url/title パラメータは、接続確立前に読み取っておき、
  // setConnected() 内で(接続経路によらず)まとめて適用する
  var pendingBookmarkletData = (function () {
    var params = new URLSearchParams(location.search);
    var paramUrl = params.get('url');
    return paramUrl ? { url: paramUrl, title: params.get('title') || '' } : null;
  })();
  ```

- [ ] **Step 4: `setConnected()`に接続確立後のフックを追加する（`bookmarks-filesync.html:1387-1399`）**

  ```js
  function setConnected(handle, data) {
    fileHandle = handle;
    bookmarks = data;
    state.editingId = null;
    state.selectMode = false;
    state.selectedIds.clear();
    document.getElementById('layout').hidden = false;
    document.getElementById('connect-screen').hidden = true;
    document.getElementById('file-status').textContent = '接続中: ' + handle.name;
    document.getElementById('disconnect-btn').hidden = false;
    setHeaderButtonsEnabled(true);
    render();
    applyPendingBookmarkletData();
  }

  function applyPendingBookmarkletData() {
    if (!pendingBookmarkletData) return;
    var data = pendingBookmarkletData;
    pendingBookmarkletData = null;
    newUrlInput.value = data.url;
    newTitleInput.value = data.title;
    addPanel.hidden = false;
    newUrlInput.focus();
    if (!data.title) newUrlInput.dispatchEvent(new Event('input'));
  }
  ```

- [ ] **Step 5: claude-in-chromeで3経路すべてを確認する**

  1. 未接続状態から「新規作成」ボタンで接続 → 接続直後に追加パネルが自動プリフィルされることを確認する（GREEN）。
  2. 未接続状態から「既存ファイルを開く」ボタンで接続 → 同様に確認する。
  3. 前回接続済みファイルへの自動再接続（権限が残っている場合の`boot()`経路）→ 同様に確認する（この経路は権限プロンプトが不要な環境でのみ再現可能。困難な場合は`setConnected()`の呼び出し元3箇所すべてが同一関数を経由していることをコードリーディングで確認し、その旨を証拠として記録する）。
  4. `?url=`パラメータが無い通常起動では、接続後も追加パネルが自動で開かないこと（回帰確認）を確認する。

- [ ] **Step 6: コミット**

  ```bash
  git add bookmarks-filesync.html
  git commit -m "feat: filesync版でブックマークレット経由の自動プリフィルを接続確立後に適用"
  ```

---

### Task 10: ブックマークレット生成UI導線

**Files:**
- Modify: `bookmarks.html:1093-1094`（ヘルプパネル: `</dl>`の直後に生成UIを追加）
- Modify: `bookmarks.html`（生成ロジックのJSを追加、`toggleHelp()`近くまたはヘルプパネル初期化コードの近く）
- Modify: `bookmarks-filesync.html:1193-1194`（対応箇所、同一パターン）

**Interfaces:**
- Consumes: なし（`location.href`から動的に生成するため、設置パスに依存しない）

- [ ] **Step 1: シナリオを定義する（RED確認用）**

  シナリオ: 「ヘルプパネルに『ブックマークレットを生成』導線があり、生成されたリンクの`href`が`javascript:`スキームで現在のURLを組み込んでいる」
  - RED: 実装前はヘルプパネルにブックマークレット関連の要素が存在しない。
  - GREEN: 実装後は`#bookmarklet-link`のようなリンク要素が存在し、`href`が`javascript:location.href='<現在のベースURL>?url='+...`の形式になっている。

- [ ] **Step 2: claude-in-chromeでbookmarks.htmlを開きヘルプパネル（`?`キーまたは該当ボタン）を開き、実装前の状態でREDを確認する**

  ヘルプパネル内にブックマークレット関連の要素が存在しないことを確認する。

- [ ] **Step 3: ヘルプパネルに生成UIのHTML雛形を追加する（`bookmarks.html:1093-1094`、`</dl>`の直後）**

  ```html
    </dl>
    <div class="bookmarklet-generator">
      <h3>ブックマークレット</h3>
      <p>下のリンクをブラウザのブックマークバーへドラッグして登録すると、閲覧中のページをワンクリックでこのアプリに登録できます。</p>
      <a href="#" id="bookmarklet-link" class="btn btn-ghost">ここに登録 &rarr; bookmark-shelf</a>
    </div>
  </section>
  ```

- [ ] **Step 4: リンクの`href`を動的に生成するJSを追加する（ヘルプパネル関連コードの近く、`toggleHelp()`の定義の直後）**

  ```js
  (function setupBookmarkletLink() {
    var link = document.getElementById('bookmarklet-link');
    if (!link) return;
    var baseUrl = location.origin + location.pathname; // クエリ文字列を含まない現在のページURL
    var code = "javascript:location.href='" + baseUrl +
      "?url='+encodeURIComponent(location.href)+'&title='+encodeURIComponent(document.title)";
    link.setAttribute('href', code);
  })();
  ```

- [ ] **Step 5: `bookmarks-filesync.html`の対応箇所に同一の変更を適用する**

- [ ] **Step 6: claude-in-chromeで同一シナリオを再実行しGREENを確認する**

  `document.getElementById('bookmarklet-link').getAttribute('href')`が`javascript:location.href='...'+...`形式で、現在のページのベースURLを含んでいることを確認する。手動でこのコードを別ページのアドレスバーから実行し、`bookmarks.html?url=...&title=...`へ遷移してTask8の自動プリフィルが発火することを統合的に確認する。

- [ ] **Step 7: コミット**

  ```bash
  git add bookmarks.html bookmarks-filesync.html
  git commit -m "feat: ヘルプパネルにブックマークレット生成導線を追加"
  ```

---

## Self-Review

1. **Spec coverage**: R1(データモデル・マイグレーション)→Task1、R2(お気に入り)→Task4・5、R3(並び替えモード)→Task6・7、R4(ブックマークレット)→Task8・9・10、R5(UNC対応)→Task2・3。全要件に対応するタスクあり。
2. **Placeholder scan**: 「TBD」「後で」等の記述なし。全ステップに実コードを記載済み。
3. **Type consistency**: `resolveOpenHref(b)`（Task3で定義）は`renderCard()`（Task3）・`renderFavoriteCard()`（Task5）の両方で同名・同シグネチャで使用。`frequencyScore(b)`・`sortItems(items)`（Task6で定義）はTask7で再利用時も同名。`pendingBookmarkletData`（Task9）・`applyPendingBookmarkletData()`（Task9）の命名は一貫。
4. **Review Focus**: 上記5項目それぞれに対応する確認ステップを各タスクに追加済み（Task2 Step7、Task6 Step9〜10、Task5 Step9、Task9 Step5）。

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-09-30-favorites-frequency-sort-bookmarklet-unc.md`. Please review the plan. Which execution approach would you prefer?

- **Subagent-driven** - A fresh subagent implements each task and a fresh reviewer checks it before the next one starts, then a whole-branch review at the end. Most thorough; costs a fresh context per task and per review.
- **Native** - I implement every task myself in this session, then one fresh reviewer on the most capable model checks the whole branch. Cheapest and fastest; no independent review until the end.

For this plan I recommend **Subagent-driven**, because Task5（お気に入りセクション）・Task6（並び替えモード）・Task9（filesync接続待ち）はdesign-audit CP-Aで一度既存アーキテクチャとの衝突が見つかった経緯があり、タスクごとの独立レビューで同種の見落とし（既存のイベント割り当て・ドラッグ機構・接続フックとの相互作用）を早期に検出できるメリットが大きいためです。Does the plan capture what you want, and which approach should we use?
