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
   mutation; it rebuilds tag nav, selection bar, and group sections from scratch via string
   concatenation (no virtual DOM/diffing). Bookmarks are always grouped into every existing group
   plus an "未分類" (ungrouped) section shown side-by-side — there is no "filter down to one group"
   view; group membership only affects which section a card renders in.
5. **Drag-and-drop** — cards are draggable for both manual reordering within/across group sections
   (`moveBookmark`) and for dropping into a group section's empty area to append at that group's end
   (`moveToGroupEnd`). Both paths renumber every bookmark's `order` field afterward and persist.

## Data model

Each bookmark: `{ id, url, title, tags[], group, loginId, loginPassword, memo, scheme, domain, order }`.

- `loginPassword` is stored and displayed **in plaintext** by design (there's a visible warning in
  the UI, `※ パスワードは平文で保存されます`) — this is a deliberate trade-off for a local personal
  tool, not an oversight.
- `group` is a plain string or `null` (ungrouped); groups are not a separate entity, just a value
  bookmarks share.
- `order` is a manual sort index, only meaningful when the sort mode is `manual` (the other modes,
  `title-asc`/`title-desc`, ignore it).
