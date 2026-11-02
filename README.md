# Stanzas

Personal notebook for reading poems stanza by stanza: the verse, my reading,
and a summary. Works in the browser on PC and as a Home Screen app on iPhone.
Poems are stored as Markdown files in my Dropbox.

## First-time setup

1. **GitHub Pages:** push this repository to GitHub (public), then
   Settings → Pages → Deploy from a branch → `main`, `/ (root)`.
   The site will be at `https://<username>.github.io/<repo>/`.
2. **Dropbox app:** https://www.dropbox.com/developers/apps → Create app →
   Scoped access → App folder.
   - Permissions tab: tick `files.metadata.read`, `files.content.read`,
     `files.content.write`, then Submit (before the first sign-in).
   - Settings tab → Redirect URIs: add the GitHub Pages address (with the
     trailing slash) and `http://localhost:8000/` for local testing.
   - Copy the App key.
3. Paste the App key into `DROPBOX_APP_KEY` at the top of the script in
   `index.html`, commit and push.
4. **PC:** open the site and connect Dropbox.
5. **iPhone:** open the site in Safari → Share → Add to Home Screen, open it
   from the icon and connect Dropbox there too (if sign-in doesn't return to
   the app, use "Sign in with a code").

## Updating

Edit locally (see `CLAUDE.md`), test with `python -m http.server 8000`,
bump the cache version in `sw.js`, commit and push.
