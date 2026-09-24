# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Working rules

- **Stay strictly within the scope of what the user asked for.** Do not take on extra work based on your own judgment. That includes unrequested features, refactors, style tweaks, extra files, backups, or verification steps. If something outside the request seems worth doing, suggest it in one line and wait for the user's go-ahead instead of doing it.

## What this is

A two-page static site written in vanilla HTML/CSS/JS. There is no build step, package manager, linter, or test suite. Each page is a single self-contained file with inline `<style>` and `<script>`. UI copy is Korean (`lang="ko"`).

- `index.html`: the "다이어리" app — a month calendar plus a per-day detail card (that day's todos + that day's free-text note).
- `about.html`: a self-introduction page (name → 3 interest cards → contact), linked to and from the todo app.
- `index.backup-20260908.html`: a frozen copy of `index.html` from before dark mode was added. Do not edit it.

This directory is the user's home folder (`C:\Users\gkgk8`), so most other files here (`.codex/`, `Downloads/`, etc.) are unrelated to this site.

## Running / checking

- Open the pages directly in a browser: `file:///C:/Users/gkgk8/index.html`.
- Environment quirks on this machine:
  - `python3` is a broken Windows Store stub. Use `node` (v24) for any scripting or a temporary static server.
  - The Claude-in-Chrome automation browser can reach neither `file://` URLs nor servers started on localhost here.
  - Browser checks have worked by navigating to a public page and injecting the HTML with `document.open(); document.write(html); document.close()`. Inject only once per fresh navigation: re-injecting into the same window throws "Identifier ... has already been declared" because top-level `const`s persist.
- **What actually works best here** (verified 2026-09-24): drive headless Chrome from the shell instead of the extension.
  1. `python -m http.server 8765 --bind 127.0.0.1 --directory C:\Users\gkgk8` in the background (`python`, not `python3`, is a real 3.12 install).
  2. `"C:\Program Files\Google\Chrome\Application\chrome.exe" --headless=new --disable-gpu --no-sandbox --window-size=W,H --virtual-time-budget=6000 --screenshot=<path> --dump-dom <url>`.
  3. For behaviour, put a throwaway harness page next to `index.html` that loads it in a same-origin `<iframe>`, drives it, seeds/reads `localStorage`, and prints PASS/FAIL into a `<pre>`; read the results out of `--dump-dom`. Delete the harness afterwards.
  - Two traps: a harness that reloads the iframe must guard its `load` listener with a flag or it reload-loops forever; and **`--window-size` below roughly 500px is clamped by Windows**, so a "narrow screen" screenshot silently renders wide and gets cropped. Measure narrow layouts by putting the app in a fixed-width `<iframe>` instead.

## Architecture

### Shared theme contract (duplicated in both pages; keep in sync)
Dark mode is implemented separately in `index.html` and `about.html`. The two copies must stay identical, or the theme will stop carrying over between pages:
1. **Pre-paint script in `<head>`**: reads `localStorage['today-theme']` and sets `document.documentElement.dataset.theme = 'dark'` before first paint. With no saved value, it falls back to `prefers-color-scheme`.
2. **CSS tokens**: the light palette is defined on `:root` (`--bg`, `--card`, `--text`, `--muted`, `--line`, `--primary`, `--primary-dark`, `--soft-primary`, `--shadow`, plus `--danger` in index). Every token is overridden in `:root[data-theme="dark"]`. Any token added to one block needs a counterpart in the other. Because a new token costs four edits across two files, prefer reusing the existing ones — the calendar was built without adding any.
3. **Toggle**: a fixed top-right `#theme-toggle` button, driven by `THEME_KEY = 'today-theme'`, `currentTheme()` and `applyTheme()`. Clicking it updates the icon (🌙/☀️), `aria-pressed` and `aria-label`, then writes to localStorage inside `try/catch`.

If you change the theme logic, tokens, or key name, change both files.

### Diary app data flow (`index.html`)

Two independent stores, both read through `readJson()` and written through `writeJson()` (which returns `false` instead of throwing, so a quota/private-mode failure can be shown in the UI):

- **Todos** — `todos`, an array of `{ id: Date.now() + Math.random(), date: 'YYYY-MM-DD', text, done }` under `localStorage['today-todos-v2']` (`STORAGE_KEY`). `date` is the day the todo *belongs to*, not a deadline; there is no `due` field any more.
- **Notes** — `notes`, a flat date-keyed object `{ 'YYYY-MM-DD': '...' }` under `localStorage['today-notes-v1']` (`NOTE_KEY`). An empty note deletes its key rather than storing `''`, so `notes[date] || ''` is the only read pattern needed.

**Migration**: on first load with no `today-todos-v2`, `loadTodos()` converts the old `today-todos-v1` records, mapping the old `due` to `date` (falling back to today) and dropping `due`. It sets `migrated`, and the bootstrap at the bottom of the script writes v2 once. **`today-todos-v1` is deliberately left in place** as a rollback source — do not delete it.

**Screen state**: `selectedDate` (the day the detail card shows, initially today) and `viewMonth` (the first of the month the calendar shows). `selectDate()` is the only way to change the selection; it first calls `storeNote()` so an unsaved note is never lost when the date changes, and it pulls `viewMonth` along when the new date is in another month.

- Every todo mutation follows **mutate `todos` → `saveTodos()` → `renderAll()`** — the calendar dots depend on the todos, so both halves must redraw.
- `renderCalendar()` rebuilds `#cal-grid` (7 weekday headers + 42 day buttons); `renderDay()` rebuilds `#todo-list` and refills `#note-text`. User text is inserted with `textContent`, and `escapeHtml()` is used only where text lands inside an attribute in a template string. Keep that XSS-safe pattern.
- Calendar cells encode state **without relying on colour alone**: today is a coloured bold number, the selected day is a background + border, an incomplete todo is a filled dot and a completed one an outlined dot, `+N` covers more than three, and `✎` marks a day that has a note. Each cell also carries a spelled-out `aria-label`. Keep that when adding new per-day signals.
- `.cal-grid` must use `repeat(7, minmax(0, 1fr))`. Plain `1fr` has a min-content floor, so the dot row widens the columns and the seven cells overflow the card (which is clipped by `.card { overflow: hidden }`) on narrow screens.
- All dates are handled as local `'YYYY-MM-DD'` strings via `dateKey()` / `parseKey()` / `shiftKey()`. Never `new Date('2026-09-23')` — it parses as UTC and can shift a day.

### About page (`about.html`)
- Content that the user still needs to personalize (name, tagline, the 3 interest cards, GitHub URL, location) is marked with `<!-- ✏️ ... -->` HTML comments. The ✏️ markers must stay inside comments so they never render on screen.
- The interest cards use a 3-column grid that collapses to 1 column at `max-width: 620px`. Both pages have a second breakpoint at `520px` that shrinks the toggle.

### Cross-links
- `about.html` links to `index.html` through the `.app-link` card.
- `index.html` links to `about.html` through the footer's `.about-link`.

Both links are relative, so the two files must stay in the same directory.
