# Reading Tracker

A personal reading log that runs entirely in the browser. You add books, track them through a status (Want to read → Reading → Finished, or Did not finish), log page progress, rate them 1–5 stars, and write notes. Covers are fetched automatically from Open Library.

To run it, open `index.html` in a browser (`open index.html`). There is no build step, no server, no dependencies, and no tests.

## What's in it

Everything lives in one file, `index.html`: the styles in `<style>`, the markup, and all the JavaScript in a single `<script>`. The script is split into sections by `// ---------- Name ----------` comments:

| Section | What it does |
|---|---|
| State | `load()`, `normalize()`, `save()`, the storage-failure checks |
| Warning bar | `showNotice()`, used when saving fails or saved data can't be read |
| Covers (Open Library) | `findCovers()`, plus `backfillCovers()`, which looks up covers in the background for books that don't have one |
| Rendering | `render()`, which redraws the stats, filter tabs and book list; `visibleBooks()` / `sortBooks()` |
| Book actions | The Start / Finish / Edit buttons on each card, and the inline current-page input |
| Dialog | The add/edit form, the rating stars, the read-month picker, the cover picker |
| Toolbar | Add button, filter tabs, search, sort |
| Import / export | Export JSON (full backup), Export CSV (for spreadsheets), Import JSON (replaces everything) |
| Other tabs | Keeps several open tabs in sync through the `storage` event |

### Book record

```js
{
  id, title, author, isbn,          // isbn holds only digits/X, from cleanIsbn()
  coverId,                          // Open Library cover id, or null
  coverChecked,                     // true once a cover lookup ran or the user chose one
  status,                           // 'want' | 'reading' | 'finished' | 'dnf'
  pages, currentPage,               // 0 means unknown
  read,                             // 'YYYY-MM', only for finished books, otherwise ''
  notes,
  rating,                           // 1–5, or 0 for no rating
  createdAt, updatedAt,             // ms timestamps
}
```

Everything is saved as `{ books: [...] }` under the localStorage key `reading-tracker:v1`.

## Key decisions

- **Vanilla HTML/CSS/JS, no frameworks, no build.** The app is small enough that a framework would add more weight than it saves. It can be opened straight from disk, and it will keep working years from now without dependency upkeep. Keep it that way: don't add npm, a bundler, or libraries unless there's a strong reason.
- **One file.** This keeps it trivially portable (copy one file and you have the app). Use the section comments to stay oriented rather than splitting into modules.
- **localStorage for persistence.** There's no backend or account, and the data stays on the user's device. The trade-off is that data lives in one browser on one device and can be lost if site data is cleared, which is why JSON export/import exists as the backup and transfer path.
- **Rendering is "rebuild the HTML string."** `render()` rebuilds the list with template strings and `innerHTML`. Event handlers are delegated on containers (`els.books`, `els.tabs`, `els.coverPicker`) through `data-*` attributes, so they survive re-renders. The one exception is `refreshCover()`, which swaps a single cover in place so background lookups don't redraw the whole list.
- **The read date is a month, not a day.** People rarely remember the exact day they finished a book. `read` is `YYYY-MM` and is shown as "Read October 2026."
- **Open Library for covers.** It's free and needs no API key. ISBN matches are tried first, then a title/author search. Up to 6 choices are shown in the dialog.
- **CSV export is for spreadsheets, JSON is for backup.** CSV has Title, Author, Status, Date finished, Rating, Notes. It includes all books regardless of the current filter or search, in the current sort order. The CSV can't be imported back; only JSON round-trips.
- **Fail loudly, never silently lose data.** If saving fails, a warning bar offers a backup download. If saved data can't be parsed, a copy is kept before anything overwrites it.

## Gotchas before modifying

- **Always call `save()` and then `render()` after changing `state`.** `updateBook()` does both and also bumps `updatedAt`. Use it for edits to a single book.
- **Escape all user text in HTML with `escapeHtml()`.** Rendering uses `innerHTML`, so a title like `<img onerror=...>` would run as code if it went in unescaped.
- **New fields must be added to `normalize()`.** Every load, import and cross-tab sync goes through `normalize()`, and it rebuilds each record from a fixed list of fields. A field you don't add there gets silently dropped. Give it a safe default for old data that doesn't have it.
- **Don't change the storage format in a breaking way.** Existing users have data under `reading-tracker:v1`, and exported JSON files will be imported later. `normalize()` already migrates an older `finished` date field to `read`; handle future changes the same way rather than bumping the key.
- **`read` is cleared unless the status is `finished`.** `normalize()` and `readValue()` both enforce this. A DNF book has no date.
- **`rating` of 0 means "not rated."** Stats skip it when averaging, and the CSV writes it as a blank, not 0.
- **`STATUSES` has an `all` entry** that's only a filter tab label, not a real status. The checks in `normalize()` exclude it; do the same anywhere you validate a status.
- **`coverChecked` controls the background lookup.** `backfillCovers()` only searches books where `coverChecked` is false and there's no `coverId`. Once a user picks "No cover" or a search has run in the dialog, it's set to true so the lookup never overrides the user's choice.
- **`backfillCovers()` runs slowly on purpose.** It looks up one book at a time with a 500 ms pause, to be polite to Open Library. It re-checks that the book still exists and wasn't edited meanwhile before writing.
- **Several tabs can be open.** The `storage` listener replaces `state` when another tab saves, so this tab doesn't overwrite newer data on its next save. Keep that working if you change how state is held.
- **Imports replace everything; they don't merge.** That's deliberate, and the user is asked to confirm first.
- **Image `error` events don't bubble.** The fallback-to-letter-tile handlers on covers use capture (`addEventListener(..., true)`).
- **CSV escaping lives in `csvCell()`.** It quotes cells that contain commas, quotes or line breaks, and puts an apostrophe before cells starting with `= + - @` so Excel doesn't run them as formulas. The file starts with a UTF-8 BOM so Excel on a Mac shows accents and emoji correctly. Don't remove either.
- **Safari needs the download URL to live briefly.** `download()` revokes the blob URL after a 1 s delay rather than right away.
- **A title made of only spaces passes `required`.** `checkTitle()` sets a custom validity message to catch it, and submit has a backstop check.
- **Styling uses CSS variables** on `:root`, with a dark-mode override in `prefers-color-scheme: dark`. Add new colors as variables in both places rather than hard-coding them. The layout has phone breakpoints at 520px and larger tap targets under `pointer: coarse`.
- **There's no automated testing.** Test changes by hand in a browser, including dark mode and a narrow window, and ideally in Safari as well as Chrome.
