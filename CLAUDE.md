# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠️ STALE FORK — READ BEFORE MAKING CHANGES

This is a **preserved fork** of [futurepress/epubjs-reader](https://github.com/futurepress/epubjs-reader), kept here in case the upstream library is abandoned or removed. **Do not treat this as active development code.**

Before modifying anything in this directory:
- Confirm the change cannot be made in the consuming project instead (e.g., Web or Mobile)
- Check whether the upstream library has already fixed the issue
- If a patch is truly necessary, document it clearly so it can be reapplied if the fork is ever refreshed
- Prefer minimal, targeted changes — avoid refactoring or modernizing

This fork is **not published** and is not consumed by any other project in this monorepo. It exists solely as a code preservation backup.

---

## Commands

```bash
npm install          # Install dependencies
npm start            # Start dev server (auto-finds open port starting at 8080)
grunt                # Build: concat + minify src/ → reader/js/
grunt watch          # Rebuild on src/ changes (concat + uglify only, no copy)
```

After any change to `src/`, run `grunt` to update `reader/js/reader.js` and `reader/js/reader.min.js`. Do not edit files in `reader/js/` directly — they are generated outputs.

## Architecture

This is a vanilla JavaScript epub reader built on [epub.js](https://github.com/futurepress/epub.js). It uses the `EPUBJS` global namespace with jQuery for DOM and RSVP for promises.

### Source → Build Flow

```
src/core.js              ─┐
src/reader.js             ├─ grunt concat → reader/js/reader.js
src/controllers/*.js      ┘                reader/js/reader.min.js

node_modules/epubjs/      ─── grunt copy → reader/js/epub.js
src/plugins/              ─── grunt copy → reader/js/plugins/
```

The `reader/` directory is the deployable static app. `reader/index.html` loads scripts in this order: jQuery → zip → screenfull → epub.js → reader.js → optional plugins.

### Reader Initialization

`EPUBJS.Reader` (in `src/reader.js`) is the main class. On construction it:
1. Merges URL query parameters into `this.settings` (so `?bookPath=URL&restore=true` work as overrides)
2. Creates a `new ePub(bookPath)` instance and calls `book.renderTo("viewer")`
3. After `book.ready` resolves, instantiates all controllers via `EPUBJS.reader.XController.call(reader, book)`

### Controllers Pattern

Controllers in `src/controllers/` are factory functions called with `call(reader, book)`. They close over the reader instance, set up DOM event listeners, and return an object of their public methods. They are not classes.

| Controller | Responsibility |
|---|---|
| `ReaderController` | Next/prev navigation, sidebar slide, loader, arrow keys |
| `SettingsController` | Settings modal, sidebar reflow toggle |
| `ControlsController` | Toolbar buttons (fullscreen, bookmark toggle) |
| `SidebarController` | Panel switching (TOC / Bookmarks / Notes views) |
| `TocController` | Renders table of contents |
| `BookmarksController` | Bookmark list UI |
| `NotesController` | Annotations/notes UI |
| `MetaController` | Displays book title and chapter title in titlebar |

### Plugins

`src/plugins/search.js` and `src/plugins/hypothesis.js` attach to `EPUBJS.reader.plugins` and are instantiated automatically for any key in that object. Currently commented out in `reader/index.html`.

### Settings Persistence

Reader state (bookmarks, annotations, font size, last position CFI) is stored in `localStorage` under the key:
```
"epubjsreader:" + EPUBJS.VERSION + ":" + window.location.host + ":" + bookPath
```
Settings are only saved/restored when `?restore=true` is passed in the URL.

### Changing the Book

Edit `reader/index.html` line 24 to change the default book URL, or pass `?bookPath=URL` as a query parameter when opening the reader.
