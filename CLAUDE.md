# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**bookmark-shelf** is a personal bookmark manager, delivered as standalone single-file HTML apps (no build step, no
package manager, no server, no dependencies). Each file is a complete app: inline `<style>`, inline
`<script>`, vanilla ES5-style JS in an IIFE. Open the file directly in a browser to run it.

There are two variants:

- `bookmarks.html` — persists to the browser's `localStorage` (key `bookmarks_data`). Works in any
  modern browser.
- `bookmarks-filesync.html` — persists directly to a JSON file on disk via the File System Access
  API (`showOpenFilePicker` / `showSaveFilePicker`). Requires Chrome or Edge; shows an
  "unsupported browser" screen otherwise.

**These two files duplicate almost all of their CSS and rendering/UI logic.** When changing shared
behavior (card layout, search/sort/tag filtering, drag-and-drop, add/edit forms, import sanitization,
credential display, etc.), apply the change to both files — they are not generated from a common
source. Only the data layer (top of each `<script>` block) is meaningfully different between them.

## Running / testing

No build, lint, or test commands exist. To verify a change, open the file in a browser and exercise
the feature manually:

```
xdg-open bookmarks.html              # localStorage variant, any browser
xdg-open bookmarks-filesync.html     # File System Access variant, must be Chrome/Edge
```

There is no automated test suite — verify UI changes by hand in the browser per the standard
"test the golden path and edge cases" practice.

## Architecture (per file)

Each file's script is a single IIFE with these layers, in order:

1. **Data layer** — this is the part that differs between the two files:
   - `bookmarks.html`: `loadData()`/`saveData()` read/write `localStorage` synchronously. On load,
     bookmarks missing a manual-sort `order` field are migrated in place (assigned by array index)
     — this is a one-time compatibility shim for data saved before drag-to-reorder existed. The same
     migration pattern backfills `pinned`/`lastAccessed`/`accessCount` (added `false`/`null`/`0`) for
     data saved before favorites/frequency-sort existed, gated on `pinned` being non-boolean.

     A further migration backfills `favoriteOrder` via `assignFavoriteOrder()` for pinned bookmarks saved before favorites reordering existed — the same function both files' JSON-import path (and bookmarks-filesync.html's initial file read) also calls.
     It's gated on `pinned === true` specifically, not just `typeof favoriteOrder !== 'number'`.
     The latter alone would also match every already-unpinned bookmark (whose `favoriteOrder` is permanently `null` by design) and retrigger the migration on every single load.

     `bookmarks.html`'s only `localStorage` key is the bookmark data itself (`STORAGE_KEY`). Two former additional keys have since been removed along with the features they backed: `SORT_MODE_KEY` persisted a manual-vs-frequency display toggle (via a `loadSortMode()` that coerced any unrecognized stored value back to `'manual'`), and `DENSITY_MODE_KEY` backed a multi-density display toggle (see the Render layer below) — any leftover value for either key in a user's existing `localStorage` is simply never looked at again.
   - `bookmarks-filesync.html`: adds an IndexedDB-backed store (`idbGetHandle`/`idbSetHandle`/
     `idbClearHandle`) that remembers the last-used `FileSystemFileHandle` so the app can offer to
     reconnect on next load without re-prompting the file picker. `boot()` drives the
     connect/reconnect/unsupported-browser flow on startup. Because permission re-grants
     (`requestPermission`) require a live user gesture, the pending handle is kept in
     `pendingReconnectHandle` rather than re-fetched from IndexedDB inside the reconnect button's
     click handler. Writes go through a `saveChain` promise chain so overlapping saves (e.g. rapid
     drag-and-drop) serialize onto the same file instead of racing `createWritable()` calls.
2. **URL/domain helpers** — `classifyUrl()` allowlists `http://`, `https://`, `file:///`, and UNC
   paths (a leading `\\`, tested before the `file:///` check) as `'unc'`; `deriveDomain()`/
   `deriveTentativeTitle()` derive a favicon domain or a fallback title (file paths are parsed by
   string splitting rather than `URL()` since `file://` paths can contain `#`/`?` characters that
   would otherwise be misparsed as fragment/query; `unc` paths are split on `\` the same way). UNC
   bookmarks are stored with their original backslash form (`b.url` is never rewritten, so import/
   export and `sanitizeBookmark()` don't need to know about the conversion); `resolveOpenHref()`
   converts a `unc`-scheme bookmark to a `file:` href (swapping `\` for `/`) only at the point a link
   is rendered. `resolveIconHtml()` centralizes the file-vs-favicon icon decision (`file` and `unc`
   both get the fixed file icon, everything else gets a favicon with a globe fallback) and is shared
   by `renderCard()` and `renderFavoriteCard()` — the two card renderers previously computed this
   inline and independently, which let them silently disagree on whether `unc` counted as file-like;
   extracting the shared helper closes that gap structurally rather than requiring both call sites to
   be kept in sync by hand.
3. **`sanitizeBookmark()`** — the single validation gate for any bookmark data not created through
   the in-app add/edit forms (JSON import in both files, plus the initial file read in the filesync
   variant). Re-derives an `id`, coerces every field to an expected type/shape, and rejects entries
   whose URL doesn't match `classifyUrl()`. Both import and file-load flows report how many entries
   were skipped rather than silently dropping them.
4. **Render layer** — `render()` is the single re-render entrypoint, called after every state
   mutation; it rebuilds tag nav, group nav, selection bar, and group sections from scratch via
   string concatenation (no virtual DOM/diffing). With no group filter active, bookmarks are grouped
   into every existing group plus an "未分類" (ungrouped) section shown side-by-side.

   `getSearchTagFiltered()`'s match string is `title + url + tags + memo` (lowercased); `loginId`, `domain`, and `group` name are not searched.

   Search and tag filters both narrow which *sections* render, not just which bookmarks show inside
   them: when `state.search.trim()` is non-empty *or* `state.tag !== null`, `renderSections()`
   narrows the side-by-side set to only the groups with at least one match under the active
   search/tag filter. A `.search-no-results` message (`検索条件やタグ絞り込みに一致するブックマーク
   がありません。条件を変更するか、クリアしてください。`, worded to cover a tag-only filter as well
   as a search-only or combined one) renders instead if none match at all. When `state.group` is
   set, `renderSections()` renders only that one section (the selected group, or 未分類) instead of
   the full side-by-side set; the same search-or-tag narrowing still applies on top of that
   single-section view, so an active search and/or tag filter with zero matches inside the selected
   group hides that section too and falls back to the same message — the two narrowing rules are
   not mutually exclusive. `state.tag` therefore does double duty: it narrows which bookmarks appear
   *within* the rendered section(s) the same way it always has, and (like `state.search`) also
   decides which sections get rendered at all.

   `state.group` uses `null` for "no
   group filter" and `''` (empty string) for "filter to 未分類", which is a different sentinel from
   `b.group`'s own `null`-for-ungrouped convention — do not conflate the two. `render()` clears
   `state.tag`/`state.group`/`state.editingGroup` back to their default when the thing they
   reference (a tag, a named group, a renamed-away group) disappears, so a stale filter or open
   rename form can't get stuck showing nothing. The sidebar itself is a two-tab switcher
   (`state.sidebarTab`, `'tag'` or `'group'`, toggled by `switchSidebarTab()`) showing only the
   タグ or グループ nav list at a time — switching tabs never touches `state.tag`/`state.group`,
   so a filter set on the hidden tab stays active (and still shows in the filter indicator).
   A "すべて解除" button next to the tabs clears both `state.tag` and `state.group` at once and is
   visible whenever either filter is active (`updateSidebarClearButtons()`, run on every
   `render()`) — clearing a single axis is still done via its nav button (re-click to toggle off)
   or the corresponding × on the filter indicator chip, same as before tabs existed.
   Tab switching is click-only, via the `#sidebar-tab-tag`/`#sidebar-tab-group` buttons calling `switchSidebarTab()` — there is no keyboard shortcut for it (a `t`/`g` shortcut pair existed earlier but was removed, along with `n` for opening the add form, `?` for the help panel, and `x` for clearing all filters; every one of those actions remains reachable by clicking its header/nav button, only the keyboard shortcut was dropped).
   Each of those shortcuts was removed because a corresponding button already existed and performed the same action via a click, making the keyboard shortcut redundant.
   Clicking a tag-nav button, a group-nav button, a list-row tag chip (see the Render layer paragraph on card markup below), or this "すべて解除" button also scrolls the page back to the top (`window.scrollTo({ top: 0, behavior: 'auto' })`), since any of these changes which content is visible further down the page.
   The default tab on load is グループ (group), not タグ (tag) — this default is expressed in two separate places that must be kept in sync by hand: `state.sidebarTab`'s initial value (`'group'`) and the tab buttons'/panels' hardcoded `aria-selected`/`hidden` attributes in the markup.
   `switchSidebarTab()` is never called on startup, so nothing reconciles the two automatically — changing only one (e.g. flipping the initial `state.sidebarTab` without also updating the markup, or vice versa) produces a silent bug where the internal state and the on-screen tab disagree.

   A pinned bookmark (`b.pinned`) additionally renders in a standalone **favorites section** (`#favorites-section`, populated by `renderFavoritesSection()`, called from `render()` right before `renderSections()`) — a DOM tree kept entirely separate from `#group-sections`.
   It's built from `renderFavoriteCard()` markup (`data-fav-id`, not `data-card-id`), which supports opening the link, unpinning, copying the login ID/password (via the same shared `.ind-btn`/`renderCredentialIndicators()` helper `renderCard()` uses), and editing.
   All four are wired by `renderFavoritesSection()`'s own click-binding loop (`container.querySelectorAll('[data-fav-id]')`), not the group section's per-card loop — so it's intentionally invisible to every mechanism keyed on `data-card-id`: the per-card event-binding loop in `renderSections()`, `bindGroupSectionDragEvents()`, and the roving-tabindex keyboard navigation (`getNavigableCards()`).
   Clicking a favorite row's edit button sets `state.editingId = b.id` and re-renders the same as any other edit trigger, but `renderFavoritesSection()` never checks `state.editingId`, so the favorites section's own markup doesn't change — the edit form opens in the bookmark's normal group section instead (`renderSections()` swaps in `renderEditCard()` there), and the user has to scroll to that section to see it.
   A pinned bookmark still renders in its normal group section too — favorites is a duplicate view, not a move.

   The favorites section reuses `getSearchTagFiltered()` so an active search/tag filter narrows it
   the same way it narrows group sections, but it is unconditionally hidden whenever
   `state.group !== null` (a specific group, or `''` for 未分類, is selected) regardless of whether
   the selected group itself contains pinned bookmarks — matching how every other group section stops
   rendering under a group filter except the one selected. This unconditional-hide rule was a
   post-review fix: filtering the favorites section down by "does `getSearchTagFiltered()`'s
   group-scoped result contain a pinned item" reads similar but is a different, spec-violating
   condition — it would keep the section visible under a group filter whenever that group happens to
   have a pin.

   A bookmark's usage is tracked via `frequencyScore(b)`, `accessCount / (daysSinceLastAccess + 1)`, computed from `b.accessCount`/`b.lastAccessed`. A bookmark with `accessCount === 0` scores `0` unconditionally (skipping the days-since-access term entirely) so a never-opened bookmark ranks last without needing a placeholder "very large elapsed days" value for its `null` `lastAccessed`. Opening a bookmark via its title link increments `accessCount` and stamps `lastAccessed = Date.now()` (in the normal grid, the favorites section, and the frequent section described below); `bookmarks-filesync.html` carries the same three fields through `sanitizeBookmark()`/new-bookmark defaults for cross-file JSON portability, but has no UI to write to them and never increments them.

   A bookmark with `accessCount > 0` can additionally render in a standalone **frequent section** (`#frequent-section`, populated by `renderFrequentSection()`, called from `render()` right before `renderFavoritesSection()` — so it appears immediately above the favorites section) — `bookmarks.html` only, since `bookmarks-filesync.html` never increments `accessCount`. It filters `getSearchTagFiltered()` down to bookmarks with `accessCount > 0`, sorts them by `frequencyScore()` descending, and keeps only the top 5; it is hidden entirely when that set is empty or when `state.group !== null` — the same hide rule the favorites section uses. It reuses the favorites section's `.card-favorite--grid` tile markup via its own `renderFrequentCard()`, but keys elements by `data-freq-id` rather than `data-fav-id` and binds clicks through its own `container.querySelectorAll('[data-freq-id]')` loop, giving it a DOM tree and event wiring fully independent of the favorites section's `data-fav-id` loop. Its cards have no drag grip: unlike favorites (manually ordered via `favoriteOrder`), the frequent section's order is always recomputed from `frequencyScore()` on every render, so there is nothing for a manual drag to reorder. Each card supports the same toggle-pin (☆/★), login ID/password copy, and edit actions as a favorites-section card, but has no unpin-only or remove action of its own — membership in the list is derived entirely from `accessCount`/`frequencyScore()`, not a field the user sets on the card itself.

   Both files render every card in a single, fixed list-row layout.
   A three-way display density toggle (`'comfy'`/`'compact'`/`'list'`, `state.densityMode`, its own `localStorage` key `DENSITY_MODE_KEY`, the `#density-toggle` header control, and the `renderCardComfy`/`renderCardCompact`/`renderFavoriteCardComfy` render functions) existed earlier but was removed entirely once user testing showed the list layout alone was sufficient — `renderCard()` and `renderFavoriteCard()` are now plain functions, not dispatchers; there is exactly one rendering path per card type.

   The markup still carries the `.card--list`/`.group-grid--list`/`.card-favorite--list` modifier classes as unconditional, always-written literal class names (not a state-driven branch) rather than folding their rules into the base `.card`/`.group-grid`/`.seal`/`.card-title`/`.card-footer` selectors — those base selectors are shared with `renderEditCard()` (the edit-form card), which needs to keep its own look regardless of the list layout, so rewriting them directly would leak list styling into the edit form.

   Each list row has a drag grip (`.row-grip`, a plain inline `<span>` with no background box — same structure as the favorites row's `.fav-grip`, but colored `--sage` instead of `--amber` so it reads as a quieter, row-level control rather than the favorites section's own accent) and, only when the bookmark has tags, a `.card-tags` row of `.tag-stamp` chips (visually matching the sidebar's tag-nav chips) — clicking one toggles `state.tag` the same way clicking a sidebar tag-nav entry does.
   `renderEditCard()` still renders its own empty `.card-stub` div (the boxed, dark-background drag-handle style used before this grip was simplified) purely as a decorative accent strip next to the edit form — it is unrelated to dragging and this change left it untouched.
   Login-ID and login-password render as shared `.ind-btn` indicators built by `renderCredentialIndicators(b, tabindexAttr)` (also used by `renderFavoriteCard()`): a filled field is a colored, clickable `.ind-btn.filled` `<button data-action="copy-login-id">` / `data-action="copy-login-password">` that copies its value via `copyToClipboard()`; an empty one is a dimmed, slashed `.ind-btn.empty` `<span>` that isn't clickable.
   Memo has no indicator icon — a non-empty memo instead renders its text inline next to the title as a truncated `.row-memo-preview` `<span>` (plain text with a small `#ic-memo` glyph prefix, not clickable); an empty memo renders nothing.
   Pin/edit/delete remain icon-only buttons.

   A bookmarklet-based quick-add feature (a `javascript:` link that reopened the app with the current
   page's URL/title as query params) was removed: Chromium-based browsers refuse to navigate a
   non-`file://` page to a `file://` URL that carries a query string, so `location.search` arrived
   empty and the add form never got prefilled — the feature never actually worked end to end once
   tested against a real external site.
5. **Drag-and-drop** — cards are draggable for both manual reordering within/across group sections
   (`moveBookmark`) and for dropping into a group section's empty area to append at that group's end
   (`moveToGroupEnd`). Both paths renumber every bookmark's `order` field afterward and persist, and
   also propagate the target group's `groupOrder` onto the moved bookmark.

   The favorites section supports its own drag-to-reorder, independent of the group-section drag-and-drop above.
   `bindFavoriteDragEvents()` wires `[data-fav-id]` elements (not `[data-card-id]`) through the same `bindDragReorder()` helper `bindDragEvents()` uses.
   It tracks its own drag state in a module-scoped `draggedFavId` rather than sharing `draggedCardId`, since a favorites-section drag and a group-grid drag are otherwise unrelated gestures that shouldn't share mutable state.
   Dropping within the favorites section calls `moveFavorite()`, which only renumbers the pinned bookmarks' `favoriteOrder`.
   `moveFavorite()` uses the same `compareFavoriteOrder()` comparator as the display sort, so sort order and reorder math can't disagree.
   It never writes to `order`/`group`, so reordering favorites can't move a bookmark out of its group or disturb its position within that group's own section.

   Because dragging a favorite card never sets `draggedCardId`, `bindGroupSectionDragEvents()`'s `dragover`/`drop` handlers both start with `if (!draggedCardId) return;`.
   Without this guard, dragging a favorite card over a group section would fall through to `moveToGroupEnd()` and silently reassign that bookmark's real `group`/`order`, corrupting data instead of being a no-op.
   This guard was added in a post-implementation whole-branch review once the favorites-drag feature made that cross-drag spillover possible.

   Group section headers themselves show only a title and count — reordering and renaming groups is not done on the section
   header, it's done in the **グループ管理 (group management) modal** (`#group-manage-overlay`,
   opened via the "管理" button, which lives inside the サイドバー's グループ tab panel and is
   only reachable while that tab is active), which lists every group name
   as a row supporting both drag-and-drop and up/down buttons (`moveGroupStep()`), both funneling into
   `moveGroupSection()`, which renumbers `groupOrder` across every bookmark in every group. Renaming
   (previously inline on the section header) also lives in this modal, toggled per-row via
   `state.editingGroup`. The modal's row drag-and-drop uses its own module-scoped `manageDraggedGroup`
   variable — separate from card drag-and-drop — since the modal and the card grid never overlap in
   the DOM, so no shared disambiguation variable (like the old `dragKind`) is needed. The modal hooks
   into the same `closeTopmostLayer()` Escape-key layering chain as the help panel, the tag management
   modal, and add/edit forms (help panel → group management modal → tag management modal → edit/add
   forms → select mode); when none of those layers is open, Escape instead blurs whatever element
   currently holds keyboard focus rather than doing nothing. A keydown-listener guard suppresses other global single-letter shortcuts
   while it's open. A higher-priority branch pre-empts this whole chain: while `#search-input` is
   focused and `state.search` is non-empty, Escape instead clears the search and refocuses the input
   (`clearSearch()`, mirroring the `#search-clear-btn` click) — but only when the group modal, tag
   modal, and help panel are all closed; if any of those is open, Escape falls through to
   `closeTopmostLayer()` as before regardless of search-input focus/content, since none of those
   layers trap keyboard focus and a backgrounded search input could otherwise steal Escape via Tab.
   The 未分類 pseudo-section is never listed in the modal and is never a valid
   group-reorder target — it always renders last.

   A parallel **タグ管理 (tag management) modal** (`#tag-manage-overlay`) exists for tags, opened via
   the "管理" button inside the サイドバー's タグ tab panel (the same structure as the group modal's
   "管理" button in its グループ tab panel — each tab has its own management entry point). Unlike the
   group modal, it has no reordering — rows are listed via `getTags()` in its existing alphabetical
   order, since tags have no analog to `groupOrder`. Each row supports renaming (toggled per-row via
   `state.editingTag`, rendered by `renderTagManageRow()`) and deletion (`deleteTag()`, which prompts a
   confirm dialog reporting how many bookmarks carry the tag before stripping it — it removes the tag
   from every bookmark's `tags` array but never deletes the bookmarks themselves). Renaming merges into
   an existing tag name if the new name collides with one already present elsewhere, and also
   deduplicates within a single bookmark's own `tags` array if the rename produces a collision there.
   The modal is wired into the same `closeTopmostLayer()` Escape-key chain described above, immediately
   after the group management modal and before edit/add forms.

## Data model

Each bookmark:
`{ id, url, title, tags[], group, groupOrder, loginId, loginPassword, memo, scheme, domain, order,
pinned, lastAccessed, accessCount, favoriteOrder }`.

- `loginPassword` is stored and displayed **in plaintext** by design (there's a visible warning in
  the UI, `※ パスワードは平文で保存されます`) — this is a deliberate trade-off for a local personal
  tool, not an oversight.
- `scheme` is one of `'https'`/`'http'`/`'file'`/`'unc'`, assigned by `classifyUrl()`. `'unc'` covers
  Windows UNC paths (`\\server\share\...`) pasted straight into the URL field; like `'file'`, its
  `domain` is `null` and it renders with the fixed file icon, but it additionally needs
  `resolveOpenHref()` at render time to become a clickable `file:` link (see Architecture above).
- `pinned` (boolean, default `false`) controls whether a bookmark also appears in the standalone
  favorites section, independent of its `group`/`order` — pinning doesn't move or remove it from its
  group.
- `favoriteOrder` (`number` or `null`) is a manual sort index for the favorites section itself. It's set only while `pinned === true`, and reset back to `null` the moment a bookmark is unpinned. Like `groupOrder`, it can't be safely recomputed from array position, so it's backfilled once via `assignFavoriteOrder()` (the one-time-migration-shim pattern `order`/`groupOrder` also use) for pinned bookmarks saved before favorites reordering existed. `compareFavoriteOrder()` is the single comparator for `favoriteOrder`, used both to sort the favorites section for display and to compute `moveFavorite()`'s reorder. It treats a `null` `favoriteOrder` as sorting last rather than erroring, for data caught mid-migration.
- `lastAccessed` (`number` timestamp or `null`) / `accessCount` (`number`, default `0`) track when and how often a bookmark's link has been opened; both start `null`/`0` and are only ever written by `bookmarks.html`'s usage tracking (`frequencyScore()`, used to rank the frequent section — see Architecture above) — `bookmarks-filesync.html` carries the fields but never updates them itself.
- `group` is a plain string or `null` (ungrouped); groups are not a separate entity, just a value
  bookmarks share.
- `order` is a manual sort index; cards within a group are displayed in `order` sequence.
- `groupOrder` is a manual sort index for the group *section itself* (which group's section appears
  before which), denormalized the same way `group` is: every bookmark sharing a `group` value is
  expected to carry the same `groupOrder`. Every code path that sets `b.group` must also set
  `b.groupOrder` (via `getGroupOrderForName()`, which resolves an existing group's current value or
  assigns a new trailing value) — unlike `order`, `groupOrder` cannot be safely recomputed from
  array position, so it can't be blindly renumbered the way `order` is. Missing `groupOrder` values
  (e.g. data saved before this field existed) are backfilled once via `assignGroupOrder()`, seeded
  from the then-current alphabetical order — the same one-time-migration-shim pattern `order` uses.

---
Last updated: 2026-10-03
