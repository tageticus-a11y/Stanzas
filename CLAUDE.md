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
- **Interpreter lists:** any folder may hold `_interpreters.txt` (one name per
  line). Sync downloads them when their rev changes into `cache.names[lowercased
  folder, '' = root] = { path, rev, list }`.
  - **Which names a poem gets:** its folder's list and every parent's, nearest
    first (`levels()`, `namesHere()`); levels without a file are skipped.
  - **Name menu:** the interpretation's «Name» `<select>` offers these names,
    «Other name…» (typed by hand), «No name» and «Edit list for …» per level.
    Edits upload at once (`mode: update` with the rev; on a conflict the latest
    list is shown again).
  - **Quick button:** the «+ <name>» button on each stanza uses the first name
    of the list closest to the root (`quickName()`).
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

## Title

optional title block (first), with notes like a stanza; its
### My understanding is optional

## Epigraph

optional: only the first stanza can be one; the stanzas after it count from 1

## 1: II. optional heading

verse lines

### My understanding

notes

### Other interpretation 1: Name (hidden)

optional, any number, with a <mark class="yellow">highlight</mark>

### Question 1

shown as Q<stanza>.1

### Answer 1

optional answer to question 1

## Date

optional free text (authors write dates in many ways)

## Conclusion

optional, after the last stanza

## Additional interpretation

optional, long (pasted from Claude), shown collapsed
```
- One `.md` file per poem; file name = sanitized title.
- Content lines starting with `#` are escaped as `\#` (`escLine`/`unescLine`).
- `### My understanding` is always written, even when empty, except on the
  title block, where `und === null` means there is none (it can be removed and
  added back).
- Stanza heading (`stanza.head`: «II», «Часть первая», «----»): written after a
  colon on the stanza's `##` line and read from the original line, so case and
  further colons are kept. Ordinary stanzas only, not the title or the epigraph.
  It has no notes. «+ Add → Heading above» adds it, and it shows as one small
  centred line above the text. «Copy verse» includes it.
- Other interpretations: any number of `### Other interpretation N` per
  stanza, in memory `stanza.oths = [{ o, by?, hidden, marks }]`. `by` is the
  interpreter's name (the «Name» menu), written after a colon in the heading
  and parsed from the original line so its letter case is kept. The caption is
  the name if there is one; otherwise the UI numbers them, but only when
  there's more than one (`otherLabel`). The legacy single
  `### Other interpretations` becomes interpretation 1.
- Questions: a stanza has any number of `### Question N` sections. N is the
  position, renumbered on removal, and the UI shows «Q<stanza>.<N>».
  `### Answer N` (at most one per question) belongs to question N; an answer
  without a number goes to the last question. In memory:
  `stanza.qs = [{ q, a?, hidden, marks }]`. The legacy single `### Questions`
  becomes question 1.
- A ` (hidden)` suffix marks a collapsed interpretation or question.
- Title block and epigraph:
  - **Title block:** `stanza.isTitle`, written as `## Title`; only allowed
    first.
  - **Epigraph:** `stanza.epigraph`, written as `## Epigraph`; allowed first,
    or right after the title block.
  - **Rules:** `kindOf()` enforces both positions, and both blocks keep
    understanding, interpretations and questions like any stanza.
  - **Numbering:** `stanzaNo()` labels them «T» and «E» (questions «QT.1»,
    «QE.1») and counts ordinary stanzas from 1 after them.
  - **Content:** the title block's text is separate from `# Title` (which
    stays the poem/file title); it's prefilled from it when added. «Copy verse»
    uses it as the title.
- Poem-level `SECTIONS` come after the last stanza: `## Date`, `## Conclusion`,
  then `## Additional interpretation`. Any other `## ` heading starts a stanza. The
  interpretation shows collapsed whenever a poem opens; that state isn't saved.
- Summary is gone: `### Summary` / `### Выжимка` sections are dropped when
  read, and a 4th TSV import column is ignored.
- Formatting is stored inline:
  - **Tags:** `<mark class="COLOR">…</mark>` (COLOR one of `COLORS`: yellow,
    green, blue, pink, violet), `<i>…</i>` and `<u>…</u>`, nested in that
    order (`encodeField`, `decodeMarks`).
  - **In memory:** range lists over the plain text: `obj.marks[field]` for
    colours, `obj.marks[field + '/i']` and `obj.marks[field + '/u']`
    (`[[start, end, kind]]`).
  - **Escaping:** backslashes before a literal `<mark`, `</mark`, `<i>`, `</i>`,
    `<u>` or `</u>` are escaped.
  - **Rendering:** each field is a textarea with transparent text over a
    `.backdrop` that draws the visible text with its formatting. Keep their
    font, padding and border identical.
  - **Italic must not change letter widths,** or the caret drifts from the
    drawn text. That's why no italic font face is loaded: the italic is
    synthesised from the regular face. Don't add `ital` back to the Google
    Fonts URL.
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
- The format palette (5 colours, I, U, Clear; Ctrl/Cmd+I and +U on PC) sits in the sticky top bar on fine-pointer devices, and
  on its own row under the focused field's label on touch.
- Each stanza's «+ Add» select has two groups:
  - This stanza: Question, Other interpretation.
  - New stanza:
    - Stanza above: not on a title block or epigraph.
    - Stanza below: not on a title block that has an epigraph after it.
    - Epigraph above: on the first ordinary stanza, if there's no epigraph.
    - Title above: on the first block, if there's no title block.
  - This stanza also offers Heading above (ordinary stanzas without one) and,
    on a title block without one, My understanding.
- Copy buttons:
  - «Copy verse»: title, stanzas and date as plain text, no notes or highlights.
  - «Copy all for Claude»: `serialize` without the additional interpretation.
  - Conclusion «Copy»: just its text.
  - Per-stanza «Copy for Claude»: the stanza with its notes, highlights kept as
    `<mark>`.
- Long Other interpretations, questions and answers are cut to `CLAMP` px with
  a fade (`grow()`). They're shown in full while focused or after «Expand».
  The expanded state lives in memory and is cleared when a poem opens.
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
