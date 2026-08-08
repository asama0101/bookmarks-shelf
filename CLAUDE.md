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
     — this is a one-time compatibility shim for data saved before drag-to-reorder existed.
   - `bookmarks-filesync.html`: adds an IndexedDB-backed store (`idbGetHandle`/`idbSetHandle`/
     `idbClearHandle`) that remembers the last-used `FileSystemFileHandle` so the app can offer to
     reconnect on next load without re-prompting the file picker. `boot()` drives the
     connect/reconnect/unsupported-browser flow on startup. Because permission re-grants
     (`requestPermission`) require a live user gesture, the pending handle is kept in
     `pendingReconnectHandle` rather than re-fetched from IndexedDB inside the reconnect button's
     click handler. Writes go through a `saveChain` promise chain so overlapping saves (e.g. rapid
     drag-and-drop) serialize onto the same file instead of racing `createWritable()` calls.
2. **URL/domain helpers** — `classifyUrl()` allowlists only `http://`, `https://`, `file:///`
   schemes; `deriveDomain()`/`deriveTentativeTitle()` derive a favicon domain or a fallback title
   (file paths are parsed by string splitting rather than `URL()` since `file://` paths can contain
   `#`/`?` characters that would otherwise be misparsed as fragment/query).
3. **`sanitizeBookmark()`** — the single validation gate for any bookmark data not created through
   the in-app add/edit forms (JSON import in both files, plus the initial file read in the filesync
   variant). Re-derives an `id`, coerces every field to an expected type/shape, and rejects entries
   whose URL doesn't match `classifyUrl()`. Both import and file-load flows report how many entries
   were skipped rather than silently dropping them.
4. **Render layer** — `render()` is the single re-render entrypoint, called after every state
   mutation; it rebuilds tag nav, group nav, selection bar, and group sections from scratch via
   string concatenation (no virtual DOM/diffing). With no group filter active, bookmarks are grouped
   into every existing group plus an "未分類" (ungrouped) section shown side-by-side. When
   `state.group` is set, `renderSections()` renders only that one section (the selected group, or
   未分類) instead of the full side-by-side set — `state.tag` still narrows which bookmarks appear
   *within* the rendered section(s) the same way it always has. `state.group` uses `null` for "no
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
   or the corresponding × on the filter indicator chip, same as before tabs existed. The `t`/`g`
   keyboard shortcuts switch to the corresponding tab before focusing its list, even when that tab
   isn't currently shown. The default tab on load is グループ (group), not タグ (tag) — this default
   is expressed in two separate places that must be kept in sync by hand: `state.sidebarTab`'s
   initial value (`'group'`) and the tab buttons'/panels' hardcoded `aria-selected`/`hidden`
   attributes in the markup. `switchSidebarTab()` is never called on startup, so nothing reconciles
   the two automatically — changing only one (e.g. flipping the initial `state.sidebarTab` without
   also updating the markup, or vice versa) produces a silent bug where the internal state and the
   on-screen tab disagree.
5. **Drag-and-drop** — cards are draggable for both manual reordering within/across group sections
   (`moveBookmark`) and for dropping into a group section's empty area to append at that group's end
   (`moveToGroupEnd`). Both paths renumber every bookmark's `order` field afterward and persist, and
   also propagate the target group's `groupOrder` onto the moved bookmark. Group section headers
   themselves show only a title and count — reordering and renaming groups is not done on the section
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
   forms → select mode), and a keydown-listener guard suppresses other global single-letter shortcuts
   while it's open. A higher-priority branch pre-empts this whole chain: while `#search-input` is
   focused and `state.search` is non-empty, Escape instead clears the search and refocuses the input
   (`clearSearch()`, mirroring the `#search-clear-btn` click) — but only when the group modal, tag
   modal, and help panel are all closed; if any of those is open, Escape falls through to
   `closeTopmostLayer()` as before regardless of search-input focus/content, since none of those
   layers trap keyboard focus and a backgrounded search input could otherwise steal Escape via Tab.
   The 未分類 pseudo-section is never listed in the modal and is never a valid
   group-reorder target — it always renders last.

   The header's **「+追加」button** (`#toggle-add-btn`) doubles as a drop target for URLs dragged in
   from elsewhere (another tab, the address bar) — there is no dedicated function like a
   `quickAddBookmark()`; the drop handler reuses the existing add panel/add form as-is.
   `extractUrlFromDrop()` pulls a URL string out of the dropped `DataTransfer` (preferring
   `text/uri-list`, falling back to `text/plain`); if it's a URL `classifyUrl()` accepts, the drop
   handler opens the add panel, sets the URL field, and manually dispatches an `input` event so the
   existing title auto-fill (the `newUrlInput` `input` listener) runs — it does not save immediately,
   saving still happens only when the user presses the existing "保存" button. While a card is being
   reordered (`draggedCardId` is set), the button's `dragover` handler bails out by reading that same
   card-reorder module variable, even though the button itself sits outside the card grid — this
   cross-purpose reuse of `draggedCardId` is intentional but non-obvious. In
   `bookmarks-filesync.html`, this is also disabled while no file is connected (`fileHandle` is
   `null`) — both `dragover` and `drop` check `fileHandle` explicitly rather than relying on the
   button's `disabled` attribute alone.

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
`{ id, url, title, tags[], group, groupOrder, loginId, loginPassword, memo, scheme, domain, order }`.

- `loginPassword` is stored and displayed **in plaintext** by design (there's a visible warning in
  the UI, `※ パスワードは平文で保存されます`) — this is a deliberate trade-off for a local personal
  tool, not an oversight.
- `group` is a plain string or `null` (ungrouped); groups are not a separate entity, just a value
  bookmarks share.
- `order` is a manual sort index; cards within a group are always displayed in `order` sequence
  (there is no sort-mode selector — manual drag-to-reorder is the only ordering).
- `groupOrder` is a manual sort index for the group *section itself* (which group's section appears
  before which), denormalized the same way `group` is: every bookmark sharing a `group` value is
  expected to carry the same `groupOrder`. Every code path that sets `b.group` must also set
  `b.groupOrder` (via `getGroupOrderForName()`, which resolves an existing group's current value or
  assigns a new trailing value) — unlike `order`, `groupOrder` cannot be safely recomputed from
  array position, so it can't be blindly renumbered the way `order` is. Missing `groupOrder` values
  (e.g. data saved before this field existed) are backfilled once via `assignGroupOrder()`, seeded
  from the then-current alphabetical order — the same one-time-migration-shim pattern `order` uses.
