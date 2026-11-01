# Строфы

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
  app folder root (`''`).
- **Dropbox-API-Arg header** must be ASCII: non-ASCII chars are escaped as `\uXXXX`
  (see `hdr()`). Poem titles are Cyrillic, so this matters.
- **Local cache:** `st.cache` in localStorage, `poems[lowercased path] =
  { path, rev, remoteRev, md, dirty, ver }`. Edits update the cache immediately,
  mark it dirty and upload after 1.5 s of inactivity; also on `online`,
  on `visibilitychange` (hidden → push, visible → sync).
- **Conflicts:** uploads use `mode: update` with the last known rev. On a
  conflict the local version is uploaded as "<name> (моя версия).md" with
  autorename, and the remote version is reloaded. Never silently overwrite.

## Poem file format (keep backward compatible — existing files must still parse)
```
# Title

## 1

verse lines

### Моё понимание

notes

### Выжимка

summary
```
- One `.md` file per poem; file name = sanitized title.
- Content lines starting with `#` are escaped as `\#` (`escLine`/`unescLine`).
- Both `###` subheadings are always written, even when empty.
- Import (`parseImport`) accepts TSV copied from Google Sheets (columns
  №, Текст, Моё понимание, Выжимка; quoted cells with newlines) or plain text
  with blank lines between stanzas.

## Conventions
- Vanilla JS, no frameworks, no dependencies, no bundler. Keep it one HTML file.
- UI text is Russian. Use typographic quotes «» in UI strings.
- Layout: stacked cards on phones, three columns at ≥900px. Light and dark
  themes via CSS variables and `prefers-color-scheme`. Respect iOS safe areas.
- Build DOM with `textContent`/`value`, never `innerHTML` with user content.
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
