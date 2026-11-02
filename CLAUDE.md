# Stanzas

Personal PWA for stanza-by-stanza notes on poems. Used on a Windows PC (browser)
and an iPhone (Safari → Add to Home Screen). Hosted on GitHub Pages; all data
lives in the owner's Dropbox. No server of our own, no build step.

## Files
- `index.html` — the whole app: markup, CSS, JS in one file.
- `sw.js` — service worker: caches the app shell for offline use
  (network-first for own files, cache-first for Google Fonts, Dropbox untouched).
- `manifest.webmanifest`, `icon-180.png` (apple-touch-icon), `icon-192.png`, `icon-512.png`.

## Architecture
- **Auth:** Dropbox OAuth 2 with PKCE, `token_access_type=offline`. Refresh token,
  access token and expiry live in localStorage (`st.auth`). `DROPBOX_APP_KEY` is
  public by design in PKCE; there is no secret and there must never be one.
  Two sign-in paths: redirect (primary) and code paste (fallback for iOS
  Home Screen apps, whose storage is separate from Safari).
- **Dropbox app:** scoped, "App folder" access; scopes `files.metadata.read`,
  `files.content.read`, `files.content.write`. All paths are relative to the
  app folder root (`''`). Poems may live in subfolders (Poet, Poet/Group);
  `listRemote` lists recursively and also returns the folders
  (`cache.folders`). The picker groups poems by folder with `<optgroup>`
  («Poet › Group»), and «New» lets you pick or create a folder (upload
  creates missing folders). If the open poem's path disappears, sync follows it
  by file name, so poems can be moved in Dropbox.
- **Dropbox-API-Arg header** must be ASCII: non-ASCII chars are escaped as `\uXXXX`
  (see `hdr()`). Poem titles are Cyrillic, so this matters.
- **Local cache:** `st.cache` in localStorage, `poems[lowercased path] =
  { path, rev, remoteRev, md, dirty, ver, mod, ownMod }` (`mod` = Dropbox
  `server_modified`, `ownMod` = mod of this device's last upload). Edits update the cache immediately,
  mark it dirty and upload after 1.5 s of inactivity; also on `online`,
  on `visibilitychange` (hidden → push, visible → sync).
- **Last poem across devices:** `st.current = { key, ack }`. `touch()` (on
  open and on edit) sets `ack` to the newest `mod` not made by this device.
  When sync sees a newer foreign `mod`, it opens that poem, unless a field is
  focused, the New dialog is open or there are unsaved edits. Server times are
  compared only with server times, so device clocks don't matter.
- **Conflicts:** uploads use `mode: update` with the last known rev. On a
  conflict the local version is uploaded as "<name> (my version).md" with
  autorename, and the remote version is reloaded. Never silently overwrite.

## Poem file format (keep backward compatible — existing files must still parse)
```
# Title

## 1

verse lines

### My understanding

notes

### Other interpretations (hidden)

optional column with a <mark class="yellow">highlight</mark>

### Question 1

shown as Q<stanza>.1

### Answer 1

optional answer to question 1

## Conclusion

optional, after the last stanza

## Additional interpretation

optional, long (pasted from Claude), shown collapsed
```
- One `.md` file per poem; file name = sanitized title.
- Content lines starting with `#` are escaped as `\#` (`escLine`/`unescLine`).
- `### My understanding` is always written, even when empty.
  `### Other interpretations` (`EXTRAS`) is optional per stanza.
- Questions: a stanza has any number of `### Question N` sections. N is the
  position, renumbered on removal, and the UI shows «Q<stanza>.<N>».
  `### Answer N` (at most one per question) belongs to question N; an answer
  without a number goes to the last question. In memory:
  `stanza.qs = [{ q, a?, hidden, marks }]`. The legacy single `### Questions`
  becomes question 1.
- A ` (hidden)` suffix marks a collapsed column or question.
- Poem-level `SECTIONS` come after the last stanza: `## Conclusion`, then
  `## Additional interpretation`. Any other `## ` heading starts a stanza. The
  interpretation shows collapsed whenever a poem opens; that state isn't saved.
- Summary is gone: `### Summary` / `### Выжимка` sections are dropped when
  read, and a 4th TSV import column is ignored.
- Highlights are stored inline as `<mark class="COLOR">…</mark>`, COLOR one of
  `COLORS` (yellow, green, blue, pink, violet); in memory they are ranges
  `obj.marks[field] = [[start, end, color]]` over the plain text. Backslashes
  before a literal `<mark`/`</mark>` are escaped (`encodeMarks`/`decodeMarks`).
  In the UI each field is a transparent textarea over a `.backdrop` that draws
  the marks; keep their font, padding and border identical.
- Legacy files use Russian headings (`### Моё понимание`) and must keep
  parsing (`HEADINGS`); files are re-saved with English headings.
- Import (`parseImport`) accepts TSV copied from Google Sheets (columns
  №, text, my understanding; Russian or English header row; quoted cells with newlines) or plain text
  with blank lines between stanzas.

## Conventions
- Vanilla JS, no frameworks, no dependencies, no bundler. Keep it one HTML file.
- UI text is English. Use typographic quotes and apostrophes (“ ” ’) in UI strings.
  Poem content is often Russian, so keep Cyrillic-safe handling everywhere.
- Layout: stacked cards on phones; at ≥900px two columns (text, understanding),
  three when a stanza shows optional columns (stacked in the third). Light and dark
  themes via CSS variables and `prefers-color-scheme`. Respect iOS safe areas.
- Build DOM with `textContent`/`value`, never `innerHTML` with user content.
- The poem picker is a native `<select id="poem">` (source of truth; tests use
  `selectOption`). On fine-pointer devices a click opens our own `#poemList`
  instead, because Edge draws the native list cramped or cut off on some
  setups. It is a collapsible folder tree with search; open folders are
  remembered per device in `st.open`. Touch devices (iPhone) keep the native
  picker.
- The highlight palette sits in the sticky top bar on fine-pointer devices, and
  on its own row under the focused field's label on touch.
- Keyboard: Ctrl/Cmd+Enter in «My understanding» or a question adds a
  question. Shift+Tab goes from a question back to «My understanding», and
  from an answer back to its question.
- Error messages say what happened and what to do next.
- **Bump `CACHE` in `sw.js` (stanzas-vN) on every change to app files**,
  otherwise installed phones keep the old version.
- Never commit poem texts or notes to this repository: the repo is public and
  the poems are copyrighted. Test data must be invented placeholder text.

## Testing
- Run `python -m http.server 8000` in the project root and open
  `http://localhost:8000/` (file:// won't work: OAuth and the service worker
  need a real origin). `http://localhost:8000/` must be listed as a Redirect URI
  in the Dropbox app settings.
- Local runs use the real Dropbox app folder, i.e. real data.
- If the page seems stale locally, DevTools → Application → Service Workers →
  "Update on reload".
- Quick parser check: extract `parseImport`, `serialize`, `parseMd` and verify
  that `parseMd(serialize(m))` round-trips.

## Deploy
Commit and push to `main`; GitHub Pages republishes in about a minute.
